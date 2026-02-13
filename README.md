# Notes

Personal notes by **Yonatan Edward Njoto** for the Infrastructure Study.

## Learning Path

Start here to understand the infrastructure:

### 1. Kubernetes Basics

- [Kubernetes](./knowledge/kubernetes.md) - Pods, Services, Deployments
- [Helm](./knowledge/helm.md) - Kubernetes package manager

### 2. GitOps with ArgoCD

- [ArgoCD](./knowledge/argocd.md) - Continuous deployment from Git
- **Infrastructure:** [`apps/dev/*.yaml`](../infrastructure/apps/dev/) - Application definitions

### 3. CI/CD Pipeline

- [Argo Workflows](./knowledge/argo-workflows.md) - Build Docker images in K8s
  - **Infrastructure:** [`argo-workflows/frontend-build-template.yaml`](../infrastructure/argo-workflows/frontend-build-template.yaml)
- [Argo Events](./knowledge/argo-events.md) - Automate builds on git push
  - **Infrastructure:** [`argo-events/`](../infrastructure/argo-events/)

### 4. Setup Guide

- [HOW-TO-RUN.md](../infrastructure/HOW-TO-RUN.md) - Complete setup from scratch

---

## Directory Structure

```
notes/
├── README.md                    # You are here
├── knowledge/                   # Concepts & explanations
│   ├── kubernetes.md           # K8s basics
│   ├── helm.md                 # Templating & packaging
│   ├── argocd.md               # GitOps deployment
│   ├── argo-workflows.md       # CI/CD builds
│   └── argo-events.md          # Event automation
└── tutorial/                    # Step-by-step guides (TODO)

infrastructure/                  # Actual implementation
├── apps/dev/                   # ArgoCD applications
├── argo-workflows/             # Build templates
├── argo-events/                # Event automation
├── charts/                     # Helm charts
└── HOW-TO-RUN.md              # Setup guide
```

---

## Quick Reference

| What                  | Where                            | Docs                                       |
| --------------------- | -------------------------------- | ------------------------------------------ |
| **App definitions**   | `infrastructure/apps/dev/`       | [ArgoCD](./knowledge/argocd.md)            |
| **Build pipeline**    | `infrastructure/argo-workflows/` | [Workflows](./knowledge/argo-workflows.md) |
| **GitHub automation** | `infrastructure/argo-events/`    | [Events](./knowledge/argo-events.md)       |
| **Helm charts**       | `infrastructure/charts/`         | [Helm](./knowledge/helm.md)                |
| **Setup guide**       | `infrastructure/HOW-TO-RUN.md`   | -                                          |
