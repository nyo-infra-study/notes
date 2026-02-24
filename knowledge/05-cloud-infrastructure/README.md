# Cloud Infrastructure

[← Back to Knowledge Base](../README.md)

## Overview

AWS services that power and support your Kubernetes infrastructure. Your cluster doesn't run in a vacuum — it sits on top of compute, storage, networking, and security primitives provided by AWS.

---

## Overall Flow: EKS on AWS with Terraform + ArgoCD

This is the full picture of how everything connects — from raw AWS infrastructure to a running application managed by GitOps.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TERRAFORM (IaC Layer)                            │
│                                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐     │
│  │  Networking  │   │   Security   │   │        Compute           │     │
│  │              │   │              │   │                          │     │
│  │  VPC         │   │  IAM Roles   │   │  EKS Control Plane       │     │
│  │  Subnets     │   │  IRSA        │   │  EKS Node Groups         │     │
│  │  IGW / NAT   │   │  KMS Keys    │   │  (or Karpenter nodes)    │     │
│  │  Route Tables│   │  Sec Groups  │   │                          │     │
│  └──────┬───────┘   └──────┬───────┘   └────────────┬─────────────┘     │
│         └──────────────────┴────────────────────────┘                   │
│                                    │                                    │
│                         ┌──────────▼──────────┐                         │
│                         │   Data Layer         │                        │
│                         │                      │                        │
│                         │  RDS (PostgreSQL)    │                        │
│                         │  ElastiCache (Redis) │                        │
│                         │  S3 Buckets          │                        │
│                         └──────────────────────┘                        │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                              terraform apply
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     EKS CLUSTER (Kubernetes)                            │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Platform Layer  (Terraform bootstraps via helm_release)        │    │
│  │                                                                 │    │
│  │   ArgoCD          Karpenter        cert-manager                 │    │
│  │   aws-load-       external-dns     External Secrets             │    │
│  │   balancer-ctrl                    Operator (ESO)               │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │                                           │
│                    ArgoCD takes over                                    │
│                             │                                           │
│  ┌──────────────────────────▼──────────────────────────────────────┐    │
│  │  Application Layer  (ArgoCD syncs from Git)                     │    │
│  │                                                                 │    │
│  │   backend-server      web-frontend      backend-db              │    │
│  │   (Helm chart)        (Helm chart)      (Helm chart)            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Node Scaling (Karpenter)                                       │    │
│  │                                                                 │    │
│  │   Pod pending → Karpenter reads NodePool → launches EC2 node    │    │
│  │   Node idle   → Karpenter consolidates  → terminates node       │    │
│  └─────────────────────────────────────────────────────────────────┘    s │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                              traffic flows in
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Traffic & Observability                         │
│                                                                         │
│   Route 53 → ALB (aws-load-balancer-controller) → Ingress → Services    │
│                                                                         │
│   CloudWatch Logs / Metrics   ←   EKS control plane + nodes             │
│   PLG Stack (Grafana/Loki)    ←   application pods                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step: How It All Comes Together

**1. Networking** — Terraform provisions a VPC with public/private subnets across multiple AZs, an Internet Gateway, and NAT Gateways so private nodes can reach the internet.
→ See [Networking](./04-networking.md)

**2. Security** — IAM roles, IRSA bindings (one role per workload), KMS keys for encryption at rest, and security groups scoped to least privilege.
→ See [Security and IAM](./05-security-and-iam.md)

**3. EKS Cluster** — Terraform creates the control plane and an initial managed node group. Control plane logs ship to CloudWatch. Public API endpoint is disabled; access is private only.
→ See [Compute](./01-compute.md)

**4. Data Layer** — RDS, ElastiCache, and S3 are provisioned by Terraform. Connection strings and credentials are written to AWS Secrets Manager. ESO syncs them into Kubernetes Secrets so apps can consume them without touching Terraform state.
→ See [Database](./03-database.md) · [Storage](./02-storage.md)

**5. Platform Bootstrap** — Terraform installs cluster-wide tools via `helm_release`: ArgoCD, Karpenter, cert-manager, aws-load-balancer-controller, external-dns, and ESO. These are platform concerns, not app concerns — Terraform owns them.
→ See [Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md)

**6. GitOps Handoff** — Terraform creates one root ArgoCD `Application` pointing at your `apps/` directory in Git. From this point, Git is the source of truth. Adding or changing an app means a PR, not a `terraform apply`.
→ See [ArgoCD](../03-deployment-automation/01-argocd.md) · [Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md)

**7. Node Autoscaling** — Karpenter watches for unschedulable pods and launches the right EC2 instance (Spot or On-Demand) based on your `NodePool` constraints. It also consolidates underutilized nodes to cut costs.
→ See [Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md)

**8. Traffic** — The AWS Load Balancer Controller provisions an ALB for each Ingress resource. Route 53 records are managed by external-dns. TLS certs are issued by cert-manager via ACM or Let's Encrypt.
→ See [Networking](./04-networking.md)

**9. Observability** — CloudWatch captures EKS control plane and node metrics. The PLG stack (Prometheus/Loki/Grafana) handles application-level metrics and logs.
→ See [Management and Governance](./06-management-governance.md) · [PLG Stack](../04-observability/01-plg-stack.md)

---

## Learning Path

1. **[Compute](./01-compute.md)** — EC2, EKS, Lambda, pricing models, AMI lookup
2. **[Storage](./02-storage.md)** — S3, EBS, EFS
3. **[Database](./03-database.md)** — RDS, DynamoDB, ElastiCache
4. **[Networking](./04-networking.md)** — VPC, ALB, Route 53, security groups
5. **[Security and IAM](./05-security-and-iam.md)** — IAM, IRSA, KMS, WAF
6. **[Management and Governance](./06-management-governance.md)** — CloudWatch, CloudTrail, Budgets
7. **[Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md)** — Layer separation, Karpenter, wiring Terraform into ArgoCD

---

## Prerequisites

- [Foundations](../01-foundations/README.md)
- [Container Orchestration](../02-container-orchestration/README.md)
- [Deployment Automation](../03-deployment-automation/README.md)
- [Observability](../04-observability/README.md)

## Estimated Time

⏱️ 6-8 hours total

## Difficulty

🎯 Advanced
