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
│  VPC, Subnets, EKS Cluster, IAM Roles, IRSA, ECR,          │
│  RDS, ElastiCache, S3, KMS, ACM certs, Route53             │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ outputs (cluster endpoint, role ARNs, etc.)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2 — Cluster Bootstrap (Terraform owns this too)      │
│                                                             │
│  ArgoCD install, Karpenter install, aws-load-balancer-      │
│  controller, external-dns, cert-manager                     │
│  (Helm releases via Terraform's helm_release resource)      │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ ArgoCD takes over from here
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3 — Applications (ArgoCD owns this)                  │
│                                                             │
│  backend-server, web-frontend, backend-db,                  │
│  any app-level config, HPA, PodDisruptionBudgets            │
└─────────────────────────────────────────────────────────────┘
```

**Rule of thumb:** If it's an AWS resource or a cluster-wide platform tool, Terraform. If it's an application workload, ArgoCD.

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

## Passing Terraform Outputs into ArgoCD Apps

This is the key integration point. Terraform provisions AWS resources and produces outputs (IAM role ARNs, RDS endpoints, S3 bucket names). Your ArgoCD apps need those values.

### Pattern 1: Terraform writes to a Kubernetes Secret (recommended)

Terraform creates the secret directly after provisioning:

```hcl
resource "kubernetes_secret" "app_config" {
  metadata {
    name      = "app-config"
    namespace = "backend"
  }

  data = {
    db_host        = aws_db_instance.postgres.address
    db_port        = aws_db_instance.postgres.port
    s3_bucket      = aws_s3_bucket.assets.id
    irsa_role_arn  = aws_iam_role.app.arn
  }
}
```

Your app then reads from the secret via `envFrom` — no hardcoded values anywhere.

### Pattern 2: Terraform writes outputs → CI injects into ArgoCD `valuesObject`

Useful when you want values in Helm values rather than env vars:

```bash
# In your CI pipeline after terraform apply
DB_HOST=$(terraform output -raw db_host)

# Patch the ArgoCD Application
argocd app set backend-server \
  -p "config.dbHost=${DB_HOST}"
```

### Pattern 3: External Secrets Operator (best for secrets)

Don't put secrets in Terraform state or Kubernetes Secrets directly. Use ESO to pull from AWS Secrets Manager:

```yaml
# ArgoCD manages this ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets
  data:
    - secretKey: db_password
      remoteRef:
        key: prod/app/db          # AWS Secrets Manager path
        property: password
```

Terraform creates the secret in AWS Secrets Manager, ESO syncs it into the cluster. Clean separation.

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
