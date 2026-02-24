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

👉 **Read more:** [`kubernetes.md`](./kubernetes.md)

### 2. The problem with raw YAML: Helm

Writing `deployment.yaml`, `service.yaml`, and `ingress.yaml` for every single application becomes a massive hassle. It's repetitive and hard to manage.
**Helm** solves this by letting us use **Templates**. Instead of writing 10 files, we write one "Chart" and just change the values (like image name or port) for each app.

👉 **Read more:** [`helm.md`](./helm.md)

### 3. Automating the rollout: ArgoCD

Now that we have our apps defined, we want our pushed Docker images to be rolled out to the cluster as soon as they are ready.
**ArgoCD** sits inside the cluster and actively monitors your Git repository. It ensures that what is running in the cluster **matches exactly** what is defined in Git. If you push a change, ArgoCD pulls it and syncs your cluster automatically.

👉 **Read more:** [`argocd.md`](./argocd.md)

### 4. The Frontend Problem: Argo Workflows & Events

We have a specific challenge with Frontend applications. If you use a static site generator (like Vite/React), environment variables are **baked in** at build time. You can't just change an env var on a running container; you have to **rebuild the whole image**.

To solve this, we use **Argo Workflows** and **Argo Events**:

1.  **Argo Events** (via Sensors/Webhooks) keeps track of your Git pushes.
2.  It triggers an **Argo Workflow** to spin up a builder Pod inside the cluster.
3.  This builder creates the Docker image _with the correct environment variables baked in_ and pushes it to the registry.

To complete the cycle, we configure our Kubernetes YAML to use a stable tag (like `dev`) with `imagePullPolicy: Always`. When the new image hits the registry, ArgoCD restarts the pods, they pull the new "dev" image, and your frontend is updated with the correct config.

👉 **Read more:** [`argo-workflows.md`](./argo-workflows.md) and [`argo-events.md`](./argo-events.md)

### 5. Seeing what happened: Centralized Logging (PLG)

Once your apps are running, things _will_ break. Relying on `kubectl logs` or ArgoCD UI is not enough because:

1.  **Persistence**: If a pod crashes and restarts, its logs are gone. You lose the error trace.
2.  **Aggregation**: You can't see "all 500 errors across frontend AND backend" in one view.
3.  **Search**: You can't filter by "duration > 1s" easily.

We use the **PLG Stack** (Promtail, Loki, Grafana) to solve this. It aggregates logs from all pods into a scalable, searchable storage that persists even when pods die.

👉 **Read more:** [`plg-stack.md`](./plg-stack.md)

## 📚 Additional Resources

- [Text Manipulation](./text-manipulation.md) - Essential Linux commands for processing text, logs, and data
- [Monitoring Tools](./monitoring/README.md) - System diagnostic and monitoring commands
- [Networking Tools](./networking/README.md) - Network and file transfer utilities
- [AWS Core Services](./aws-core-services-summary/README.md) - Comprehensive guides on foundational AWS infrastructure
