# Infrastructure Knowledge Base

This directory contains comprehensive notes and guides on the core technologies used in our infrastructure.

## 🔄 The "Big Picture" Flow

Before diving into individual tools, here is how everything fits together in our complete pipeline:

```
[Developer] --(Git Push)--> [Git Repo]
                               │
                        (Webhook/Poll)
                               ↓
                        [Argo Events] --(Trigger)--> [Argo Workflows]
                                                          │
                                                    (Kaniko Build)
                                                          ↓
                                                  [Docker Registry]
                                                          │
                                                      (New Image)
                                                          ↓
[Kubernetes] <--(Apply)-- [Helm Charts] <--(Sync)-- [ArgoCD]
     │
 (Run Pods)
     ↓
 [Promtail] --(Ship Logs)--> [Loki] --(Query)--> [Grafana]
```

## 📖 The Story of Our Stack

Instead of just listing tools, here is the story of _why_ we use them and how they solve specific problems.

### 1. Where it all runs: Kubernetes

Your applications need a place to live. In the modern world, they run in **Clusters**. In **Kubernetes**, a running instance of your app is named a **Pod**.

- **Wait, what's a Pod?** It's just a wrapper around your container.
- **How do I reach it?** You need a **Service**.
- **How do I expose it to the world?** You need an **Ingress**.

👉 **Read more:** [Kubernetes](./02-container-orchestration/01-kubernetes.md)

### 2. The problem with raw YAML: Helm

Writing `deployment.yaml`, `service.yaml`, and `ingress.yaml` for every single application becomes a massive hassle. It's repetitive and hard to manage.
**Helm** solves this by letting us use **Templates**. Instead of writing 10 files, we write one "Chart" and just change the values (like image name or port) for each app.

👉 **Read more:** [Helm](./02-container-orchestration/02-helm.md)

### 3. Automating the rollout: ArgoCD

Now that we have our apps defined, we want our pushed Docker images to be rolled out to the cluster as soon as they are ready.
**ArgoCD** sits inside the cluster and actively monitors your Git repository. It ensures that what is running in the cluster **matches exactly** what is defined in Git. If you push a change, ArgoCD pulls it and syncs your cluster automatically.

👉 **Read more:** [ArgoCD](./03-deployment-automation/01-argocd.md)

### 4. The Frontend Problem: Argo Workflows & Events

We have a specific challenge with Frontend applications. If you use a static site generator (like Vite/React), environment variables are **baked in** at build time. You can't just change an env var on a running container; you have to **rebuild the whole image**.

To solve this, we use **Argo Workflows** and **Argo Events**:

1.  **Argo Events** (via Sensors/Webhooks) keeps track of your Git pushes.
2.  It triggers an **Argo Workflow** to spin up a builder Pod inside the cluster.
3.  This builder creates the Docker image _with the correct environment variables baked in_ and pushes it to the registry.

To complete the cycle, we configure our Kubernetes YAML to use a stable tag (like `dev`) with `imagePullPolicy: Always`. When the new image hits the registry, ArgoCD restarts the pods, they pull the new "dev" image, and your frontend is updated with the correct config.

👉 **Read more:** [Argo Workflows](./03-deployment-automation/02-argo-workflows.md) and [Argo Events](./03-deployment-automation/03-argo-events.md)

### 5. Seeing what happened: Centralized Logging (PLG)

Once your apps are running, things _will_ break. Relying on `kubectl logs` or ArgoCD UI is not enough because:

1.  **Persistence**: If a pod crashes and restarts, its logs are gone. You lose the error trace.
2.  **Aggregation**: You can't see "all 500 errors across frontend AND backend" in one view.
3.  **Search**: You can't filter by "duration > 1s" easily.

We use the **PLG Stack** (Promtail, Loki, Grafana) to solve this. It aggregates logs from all pods into a scalable, searchable storage that persists even when pods die.

👉 **Read more:** [PLG Stack](./04-observability/01-plg-stack.md)

## 🎓 Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                     START HERE                                   │
│                                                                  │
│  📚 01. Foundations (Est. 2-3 hours)                            │
│      ├─ Text Manipulation                                       │
│      ├─ System Monitoring                                       │
│      └─ Networking Basics                                       │
│                          ↓                                       │
│  🐳 02. Container Orchestration (Est. 4-6 hours)                │
│      ├─ Kubernetes ⭐ CORE                                      │
│      └─ Helm                                                     │
│                          ↓                                       │
│  🚀 03. Deployment Automation (Est. 3-5 hours)                  │
│      ├─ ArgoCD ⭐ CORE                                          │
│      ├─ Argo Workflows                                          │
│      └─ Argo Events                                             │
│                          ↓                                       │
│  📊 04. Observability (Est. 2-4 hours)                          │
│      ├─ PLG Stack ⭐ CORE                                       │
│      └─ Diagnostic Tools                                        │
│                          ↓                                       │
│  ☁️  05. Cloud Infrastructure (Est. 6-8 hours) [REFERENCE]     │
│      ├─ Compute                                                 │
│      ├─ Storage                                                 │
│      ├─ Database                                                │
│      ├─ Networking                                              │
│      ├─ Security and IAM                                        │
│      └─ Management and Governance                               │
└─────────────────────────────────────────────────────────────────┘

⭐ CORE = Essential for day-to-day work
[REFERENCE] = Deep-dive material, read as needed
```

## 📚 Content Groups

### [01. Foundations](./01-foundations/README.md)
Essential command-line skills | ⏱️ 2-3 hours | 🎯 Beginner

### [02. Container Orchestration](./02-container-orchestration/README.md)
Kubernetes and Helm | ⏱️ 4-6 hours | 🎯 Intermediate

### [03. Deployment Automation](./03-deployment-automation/README.md)
The Argo ecosystem | ⏱️ 3-5 hours | 🎯 Intermediate

### [04. Observability](./04-observability/README.md)
Monitoring, logging, and diagnostics | ⏱️ 2-4 hours | 🎯 Intermediate

### [05. Cloud Infrastructure](./05-cloud-infrastructure/README.md)
AWS services reference | ⏱️ 6-8 hours | 🎯 Advanced

## 🚀 Quick Start

New to the stack? Start here:
1. Read [Foundations](./01-foundations/README.md) to build baseline skills
2. Learn [Kubernetes](./02-container-orchestration/01-kubernetes.md) — the platform
3. Understand [ArgoCD](./03-deployment-automation/01-argocd.md) — the deployment tool
4. Set up [PLG Stack](./04-observability/01-plg-stack.md) — the logging system

Already familiar with the basics? Jump to specific topics:
- [Cloud Infrastructure](./05-cloud-infrastructure/README.md) — AWS services reference
- [Diagnostic Tools](./04-observability/02-diagnostic-tools.md) — Kubernetes debugging guide
