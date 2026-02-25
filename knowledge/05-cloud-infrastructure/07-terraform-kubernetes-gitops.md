# Terraform + Kubernetes + GitOps

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Terraform + Kubernetes + GitOps

How to structure your infrastructure code so that Terraform, EKS, Karpenter, and ArgoCD all work together cleanly — without stepping on each other.

## Prerequisites

- [Compute Services](./01-compute.md) — EKS cluster and node groups
- [ArgoCD](../03-deployment-automation/01-argocd.md) — GitOps delivery on top of the cluster
- [Networking Services](./04-networking.md) — VPC and subnets the cluster lives in

---

## The Core Problem

Terraform and ArgoCD both want to "own" things in your cluster. If you're not deliberate about the boundary, you'll end up with:

- Terraform trying to manage resources ArgoCD already owns (and vice versa)
- `terraform apply` breaking live ArgoCD apps
- Drift between what Terraform thinks exists and what's actually running

The fix is a clear **layer separation**.

---

## Layer Separation: What Owns What

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1 — AWS Infrastructure (Terraform owns this)         │
│                                                             │
│  VPC, Subnets, IGW, NAT, Route Tables                       │
│  EKS Cluster, Node Groups, Launch Templates                 │
│  IAM Roles, IRSA bindings, IAM Policies                     │
│  KMS Keys                                                   │
│  RDS, ElastiCache, S3, ECR                                  │
│  ACM certs, Route53                                         │
│  Secrets Manager containers  ← shell only, no values        │
│  IAM policies granting ESO/app access to those secrets      │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ outputs (cluster endpoint, role ARNs,
                          │         secret ARNs, db endpoints)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 1b — Secret Values (You / CI owns this)              │
│                                                             │
│  Actual credentials written to Secrets Manager              │
│  via AWS CLI or CI pipeline — never in .tf files            │
│  or Git                                                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2 — Cluster Bootstrap (Terraform owns this too)      │
│                                                             │
│  ArgoCD install          (helm_release)                     │
│  Karpenter install       (helm_release + kubectl_manifest)  │
│  aws-load-balancer-ctrl  (helm_release)                     │
│  external-dns            (helm_release)                     │
│  cert-manager            (helm_release)                     │
│  External Secrets Op.    (helm_release)                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ ArgoCD takes over from here
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3 — GitOps Config (ArgoCD owns this)                 │
│                                                             │
│  ExternalSecret CRDs     ← tells ESO what to sync          │
│  ClusterSecretStore CRD  ← tells ESO where to pull from    │
│  Namespaces                                                 │
│  backend-server, web-frontend, backend-db (Helm charts)     │
│  HPA, PodDisruptionBudgets, NetworkPolicies                 │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ ESO reads Secrets Manager,
                          │ creates Kubernetes Secrets
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 4 — Runtime (ESO + Kubernetes owns this)             │
│                                                             │
│  Kubernetes Secret objects  ← auto-created by ESO           │
│  Pods read secrets via envFrom — never touch AWS directly   │
└─────────────────────────────────────────────────────────────┘
```

**Rule of thumb:**
- AWS resource or cluster-wide platform tool → Terraform
- Secret value → you or CI via AWS CLI, never Terraform
- ESO/ArgoCD manifests, app workloads → ArgoCD (Git)
- Resulting Kubernetes Secrets → ESO (auto-managed)

---

## Recommended Repo Structure

Keep infrastructure and app config in separate repos (or at minimum separate directories with separate state).

```
infrastructure/          ← this repo (Terraform)
  terraform/
    modules/
      eks/               ← EKS cluster, node groups, IRSA
      karpenter/         ← Karpenter install + NodePool CRDs
      networking/        ← VPC, subnets, NAT
      argocd/            ← ArgoCD Helm install (bootstrap only)
    envs/
      dev/
        main.tf          ← wires modules together
        outputs.tf       ← exports cluster name, role ARNs, etc.
        backend.tf       ← remote state (S3 + DynamoDB lock)
      prod/
        ...

apps/                    ← ArgoCD Application manifests
  dev/
    backend-server.yaml
    web-frontend.yaml
  prod/
    ...

charts/                  ← your Helm charts
  backend-server/
  web-frontend/
```

---

## AWS Owns This: Secrets Management

This is one of the most important ownership boundaries to get right. The rule:

```
Terraform  →  creates the secret container + grants access (IAM)
You/CI     →  puts the actual value in
ESO        →  syncs it into the cluster as a Kubernetes Secret
ArgoCD app →  reads it via envFrom — never sees the raw value
```

Terraform should never hold the actual secret value. If it did, the value would be stored in plaintext in your Terraform state file — which is a serious security problem.

---

### Step 1 — Terraform Creates the Container and Declares the Variables

Terraform creates the secret resource in AWS Secrets Manager (just the shell, no value yet) and the IAM policy that allows the app to read it:

```hcl
# Create the secret container — no value set here
resource "aws_secretsmanager_secret" "db" {
  name                    = "${var.environment}/app/db"
  description             = "Database credentials for the app"
  kms_key_id              = aws_kms_key.app.arn
  recovery_window_in_days = 7
}

# Declare the expected keys — documents what the secret should contain
# but does NOT set the values
resource "aws_secretsmanager_secret_version" "db_placeholder" {
  secret_id = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "REPLACE_ME"
    password = "REPLACE_ME"
    host     = aws_db_instance.postgres.address
    port     = aws_db_instance.postgres.port
    dbname   = aws_db_instance.postgres.db_name
  })

  lifecycle {
    ignore_changes = [secret_string] # Terraform sets this once, then hands off
  }
}

# IAM policy — allows the app's IRSA role to read this secret
resource "aws_iam_policy" "read_db_secret" {
  name = "${var.environment}-read-db-secret"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
      Resource = aws_secretsmanager_secret.db.arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "app_read_db_secret" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.read_db_secret.arn
}
```

The `ignore_changes` on `secret_string` is critical — it tells Terraform "you created this, but don't touch the value again". After the first apply, the real value is written by a human or CI, and Terraform won't overwrite it on the next plan.

---

### Step 2 — You (or CI) Put the Real Value In

After `terraform apply`, update the secret value manually or via CI — never in the `.tf` file:

```bash
# Manually via AWS CLI
aws secretsmanager put-secret-value \
  --secret-id "dev/app/db" \
  --secret-string '{"username":"appuser","password":"s3cr3t","host":"mydb.xxxx.rds.amazonaws.com","port":"5432","dbname":"appdb"}'

# Or via CI pipeline (GitHub Actions example)
- name: Update DB secret
  run: |
    aws secretsmanager put-secret-value \
      --secret-id "${{ env.ENVIRONMENT }}/app/db" \
      --secret-string "${{ secrets.DB_CREDENTIALS }}"
```

The value lives in AWS Secrets Manager, encrypted with your KMS key. It never touches your Git repo or Terraform state.

---

### Step 3 — ESO Syncs It into the Cluster

External Secrets Operator watches AWS Secrets Manager and creates a Kubernetes Secret automatically. ArgoCD manages the `ExternalSecret` CRD — it's just another manifest in your `apps/` directory:

```yaml
# apps/dev/external-secrets.yaml  (ArgoCD manages this)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: backend
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials        # name of the Kubernetes Secret it creates
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: dev/app/db
        property: username
    - secretKey: password
      remoteRef:
        key: dev/app/db
        property: password
    - secretKey: host
      remoteRef:
        key: dev/app/db
        property: host
```

ESO creates a real `Secret` named `db-credentials` in the `backend` namespace. It refreshes every hour — if you rotate the value in Secrets Manager, the Kubernetes Secret updates automatically within the refresh window.

---

### Step 4 — The App Reads It via envFrom

Your Helm chart or deployment manifest reads the secret as environment variables. The app never calls Secrets Manager directly — it just sees normal env vars:

```yaml
# In your Helm chart values / deployment template
containers:
  - name: backend
    image: myapp:latest
    envFrom:
      - secretRef:
          name: db-credentials   # the Secret ESO created
```

Inside the container:
```bash
echo $username   # appuser
echo $password   # s3cr3t
echo $host       # mydb.xxxx.rds.amazonaws.com
```

---

### Updating a Secret Value

When you need to rotate or update a secret:

```
1. Update the value in AWS Secrets Manager (CLI, console, or CI)
        ↓
2. ESO detects the change on next refresh (up to 1h, or force with annotation)
        ↓
3. Kubernetes Secret is updated automatically
        ↓
4. Pod needs to restart to pick up the new env var value
        ↓
5. Trigger a rollout: kubectl rollout restart deployment/backend -n backend
```

To force an immediate ESO refresh without waiting:

```bash
kubectl annotate externalsecret db-credentials \
  force-sync=$(date +%s) \
  --overwrite \
  -n backend
```

---

### Summary: Who Owns What

```
┌──────────────────────────────────────────────────────────────┐
│  Terraform owns                                              │
│                                                              │
│  aws_secretsmanager_secret  (the container)                  │
│  aws_iam_policy             (who can read it)                │
│  aws_kms_key                (encryption key)                 │
│  ignore_changes on value    (hands off after first apply)    │
└──────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  You / CI owns                                               │
│                                                              │
│  The actual secret value (via AWS CLI or CI pipeline)        │
│  Secret rotation schedule                                    │
└──────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  ArgoCD owns                                                 │
│                                                              │
│  ExternalSecret CRD manifest (in apps/ Git directory)        │
│  ClusterSecretStore manifest                                 │
└──────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  ESO owns                                                    │
│                                                              │
│  The actual Kubernetes Secret object (auto-created/updated)  │
└──────────────────────────────────────────────────────────────┘
```



---

## Karpenter: Auto Scaling New Nodes

Karpenter replaces managed node groups for dynamic workloads. Terraform installs it; you define `NodePool` and `EC2NodeClass` resources that Karpenter uses to decide what nodes to spin up.

### Install Karpenter via Terraform

```hcl
# IRSA role for Karpenter controller
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.0"

  cluster_name = aws_eks_cluster.main.name

  # Karpenter needs to create/delete EC2 instances
  enable_irsa                     = true
  irsa_oidc_provider_arn          = aws_iam_openid_connect_provider.eks.arn
  irsa_namespace_service_accounts = ["karpenter:karpenter"]
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  version    = "0.37.0"
  namespace  = "karpenter"
  create_namespace = true

  set {
    name  = "settings.clusterName"
    value = aws_eks_cluster.main.name
  }

  set {
    name  = "settings.interruptionQueue"
    value = module.karpenter.queue_name # SQS queue for Spot interruptions
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = module.karpenter.irsa_arn
  }

  depends_on = [aws_eks_cluster.main]
}
```

### Define NodePool and EC2NodeClass

These are Kubernetes CRDs — manage them in Terraform (since they're cluster-wide platform config, not app config):

```hcl
resource "kubectl_manifest" "karpenter_node_class" {
  yaml_body = <<-YAML
    apiVersion: karpenter.k8s.aws/v1
    kind: EC2NodeClass
    metadata:
      name: default
    spec:
      amiFamily: AL2023
      role: "${module.karpenter.node_iam_role_name}"
      subnetSelectorTerms:
        - tags:
            karpenter.sh/discovery: "${aws_eks_cluster.main.name}"
      securityGroupSelectorTerms:
        - tags:
            karpenter.sh/discovery: "${aws_eks_cluster.main.name}"
      blockDeviceMappings:
        - deviceName: /dev/xvda
          ebs:
            volumeSize: 50Gi
            volumeType: gp3
            encrypted: true
  YAML

  depends_on = [helm_release.karpenter]
}

resource "kubectl_manifest" "karpenter_node_pool" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1
    kind: NodePool
    metadata:
      name: default
    spec:
      template:
        spec:
          nodeClassRef:
            group: karpenter.k8s.aws
            kind: EC2NodeClass
            name: default
          requirements:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["spot", "on-demand"]   # prefer spot, fall back to on-demand
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
            - key: karpenter.k8s.aws/instance-category
              operator: In
              values: ["c", "m", "r"]         # compute, general, memory families
            - key: karpenter.k8s.aws/instance-generation
              operator: Gt
              values: ["2"]                   # only modern instance generations
      limits:
        cpu: 100                              # hard cap — prevents runaway scaling
        memory: 400Gi
      disruption:
        consolidationPolicy: WhenEmptyOrUnderutilized
        consolidateAfter: 1m                  # reclaim idle nodes quickly
  YAML

  depends_on = [kubectl_manifest.karpenter_node_class]
}
```

> Tag your subnets and security groups with `karpenter.sh/discovery: <cluster-name>` — Karpenter uses these tags to discover where to launch nodes.

```hcl
resource "aws_subnet" "private" {
  # ...
  tags = {
    "karpenter.sh/discovery" = aws_eks_cluster.main.name
  }
}
```

---

## Tips: Keeping Terraform and ArgoCD in Sync

### Tip 1: Use remote state to share values across repos

Store Terraform state in S3 and reference outputs from other configs:

```hcl
# In your apps repo CI or another Terraform module
data "terraform_remote_state" "infra" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "envs/dev/terraform.tfstate"
    region = "us-east-1"
  }
}

# Now you can use infra outputs
locals {
  cluster_name  = data.terraform_remote_state.infra.outputs.cluster_name
  db_host       = data.terraform_remote_state.infra.outputs.db_host
}
```

### Tip 2: Tag everything consistently

Use a common tag set across all Terraform resources. Makes cost allocation, Karpenter discovery, and debugging much easier:

```hcl
locals {
  common_tags = {
    Environment = var.environment   # dev, staging, prod
    ManagedBy   = "terraform"
    Cluster     = var.cluster_name
  }
}

resource "aws_vpc" "main" {
  # ...
  tags = merge(local.common_tags, { Name = "main-vpc" })
}
```

### Tip 3: Never let Terraform touch ArgoCD Application resources

If Terraform creates an `Application` CRD and ArgoCD also manages it, you'll get conflicts. Bootstrap ArgoCD via Terraform, then hand off — use an App of Apps pattern where one root ArgoCD Application manages all others from Git.

```
Terraform installs ArgoCD
  ↓
Terraform creates ONE root Application pointing to apps/ in Git
  ↓
ArgoCD reads apps/ and creates all child Applications
  ↓
From here, Git is the only way to add/change apps
```

### Tip 4: Use `ignore_changes` for fields ArgoCD mutates

ArgoCD adds annotations and labels to resources it manages. If Terraform also manages those resources (e.g., a Namespace), it'll see drift on every plan:

```hcl
resource "kubernetes_namespace" "backend" {
  metadata {
    name = "backend"
  }

  lifecycle {
    ignore_changes = [
      metadata[0].annotations,
      metadata[0].labels,
    ]
  }
}
```

### Tip 5: Separate Terraform state per layer

Don't put everything in one state file. Split by concern:

```
state/
  networking/terraform.tfstate    ← VPC, subnets (rarely changes)
  eks/terraform.tfstate           ← cluster, node groups, IRSA
  platform/terraform.tfstate      ← ArgoCD, Karpenter, cert-manager
```

This way a change to your app's IAM role doesn't require a plan that touches your VPC.

### Tip 6: Pin versions everywhere

```hcl
terraform {
  required_version = "~> 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.13"
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = "~> 1.14"
    }
  }
}
```

Unpinned providers will silently upgrade and break things. Always pin, and upgrade deliberately.

### Tip 7: IRSA over node-level IAM roles

Give each workload its own IAM role via IRSA (IAM Roles for Service Accounts) rather than attaching broad permissions to the node role. This way a compromised pod can only access what its service account allows.

```hcl
# Create IRSA role for a specific app
module "app_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name = "backend-server"

  oidc_providers = {
    main = {
      provider_arn               = aws_iam_openid_connect_provider.eks.arn
      namespace_service_accounts = ["backend:backend-server"] # namespace:serviceaccount
    }
  }

  role_policy_arns = {
    s3 = aws_iam_policy.s3_read.arn
  }
}
```

Then in your Helm chart / ArgoCD app values:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/backend-server
```

---

## Full Flow: Infra Change → ArgoCD Picks It Up

```
1. You update Terraform (e.g., new RDS instance, new S3 bucket)
        ↓
2. terraform apply
        ↓
3. Terraform writes outputs to Kubernetes Secret (or AWS Secrets Manager)
        ↓
4. ESO / app reads the new secret value
        ↓
5. If image/chart values changed → update Git → ArgoCD auto-syncs
        ↓
6. Karpenter watches for pending pods → spins up nodes as needed
        ↓
7. Pods schedule, pull image, start up ✅
```

---

### ➡️ Next: [Back to Cloud Infrastructure](./README.md)
