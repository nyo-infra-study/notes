# Deployment Automation

[← Back to Knowledge Base](../README.md)

## Overview

Automate your deployment pipeline with the Argo ecosystem. Manual deployments are error-prone and slow — this section covers how to go from a Git commit to a running application automatically, with full auditability and rollback capability.

## Learning Path

1. **[ArgoCD](./01-argocd.md)** - Continuous deployment from Git using the GitOps model
2. **[Argo Workflows](./02-argo-workflows.md)** - Build and test pipelines as Kubernetes-native workflows
3. **[Argo Events](./03-argo-events.md)** - Event-driven automation with sources, sensors, and triggers

## Prerequisites

Before diving into this section, you should understand:
- [Container Orchestration](../02-container-orchestration/README.md) — Kubernetes fundamentals (Pods, Deployments, Services) and Helm basics (Charts and Values)

## What You'll Learn

- How to implement GitOps so your cluster state always matches your Git repository
- How to build CI/CD pipelines that run natively inside Kubernetes
- How to trigger workflows automatically in response to events like image pushes or webhooks

## Estimated Time

⏱️ 3-5 hours total

## Difficulty

🎯 Intermediate

## Next Steps

After completing this section, continue to:
- [Observability](../04-observability/README.md)
