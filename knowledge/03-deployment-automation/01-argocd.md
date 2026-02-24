# ArgoCD

[Knowledge Base](../README.md) > [Deployment Automation](./README.md) > ArgoCD

> 📖 Official Docs: [https://argo-cd.readthedocs.io/en/stable/](https://argo-cd.readthedocs.io/en/stable/) · Last updated: 2026-02-12

ArgoCD is a **declarative, GitOps continuous delivery tool** for Kubernetes. It continuously monitors your Git repositories and automatically syncs the desired application state to your cluster.

> **Think of it as:** Git is the source of truth. Push a change → ArgoCD deploys it. That's GitOps.

## Prerequisites

Before diving in, you should understand:
- [Kubernetes](../02-container-orchestration/01-kubernetes.md) — pods, deployments, services, and namespaces
- [Helm](../02-container-orchestration/02-helm.md) — ArgoCD uses Helm as a templating engine for chart-based apps

## Core Concepts

> 📖 Source: [Core Concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)

| Concept          | Description                                                                                        |
| ---------------- | -------------------------------------------------------------------------------------------------- |
| **Application**  | A group of Kubernetes resources defined by a manifest. This is a CRD (`Application`).              |
| **Target State** | The desired state of an app, as represented by files in a Git repository.                          |
| **Live State**   | The current state of the app running in the cluster (actual pods, services, etc.).                 |
| **Sync**         | The process of making the live state match the target state (applying changes to the cluster).     |
| **Sync Status**  | Whether the live state matches the target state — **Synced** or **OutOfSync**.                     |
| **Health**       | Whether the application is running correctly and can serve requests — **Healthy** or **Degraded**. |
| **Refresh**      | Compare the latest code in Git with the live state to detect drift.                                |
| **Source Type**  | The tool used to build manifests — Helm, Kustomize, Jsonnet, or plain YAML.                        |

---

## How ArgoCD Works

```
┌──────────┐     watches      ┌──────────────┐     compares      ┌─────────────┐
│  Git Repo │ ◄───────────── │   ArgoCD      │ ──────────────▶  │  K8s Cluster │
│ (target)  │                 │  Controller   │                   │  (live)      │
└──────────┘                  └──────┬───────┘                   └─────────────┘
                                     │
                              if OutOfSync
                                     │
                                     ▼
                              Apply manifests
                              (auto or manual)
```

1. You define your desired state in a **Git repo** (Helm charts, YAML manifests, etc.).
2. ArgoCD **watches** the repo and **compares** it to the live cluster state.
3. If they differ (**OutOfSync**), ArgoCD can **automatically sync** or wait for manual approval.
4. ArgoCD **reports health** and provides a web UI to visualize everything.

---

## Main Components

> 📖 Source: [Architecture](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)

### 1. API Server

The gRPC/REST server that powers the Web UI, CLI, and CI/CD integrations. Handles:

- Application management and status reporting
- Triggering operations (sync, rollback, user-defined actions)
- Repository and cluster credential management (stored as K8s Secrets)
- Authentication and RBAC enforcement

### 2. Repository Server

Internal service that maintains a **local cache** of Git repositories. Responsible for:

- Cloning and caching repos
- Generating Kubernetes manifests from Helm charts, Kustomize, plain YAML, etc.
- Returning rendered manifests given a repo URL, revision, and app path

### 3. Application Controller

The Kubernetes controller that **continuously monitors** running applications. It:

- Compares live state vs. target state
- Detects **OutOfSync** status
- Optionally takes corrective action (auto-sync)
- Invokes lifecycle hooks (PreSync, Sync, PostSync)

---

## Installation (via Helm)

The ArgoCD Helm chart is community-maintained at [argoproj/argo-helm](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd).

We install ArgoCD itself using Helm — so ArgoCD is managed by Helm, and then ArgoCD manages everything else.

```bash
# Add the ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access the UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 9000:443
# Then open https://localhost:9000 (user: admin)
```

> **Where does `https://argoproj.github.io/argo-helm` come from?**
> This is a **GitHub Pages URL** that serves the Helm chart index. The source code lives at [github.com/argoproj/argo-helm](https://github.com/argoproj/argo-helm), and GitHub Pages automatically publishes the packaged charts as a Helm repository at `https://argoproj.github.io/argo-helm`. This is a common pattern — many open-source projects host their Helm repos this way (the `gh-pages` branch contains the `index.yaml` that Helm uses to discover available charts).

### Install ArgoCD CLI

```bash
brew install argocd
```

---

## The Application CRD

The `Application` is ArgoCD's core resource. It tells ArgoCD **what** to deploy, **from where**, and **to where**.

### Example: Plain Manifests from Git

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app.git
    targetRevision: main
    path: k8s/ # Directory containing YAML manifests
  destination:
    server: "https://kubernetes.default.svc"
    namespace: my-app
  syncPolicy:
    automated:
      prune: true # Delete resources removed from Git
      selfHeal: true # Revert manual changes on cluster
```

### Example: Helm Chart from a Registry

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    chart: nginx # Chart name
    repoURL: https://charts.bitnami.com/bitnami # Helm repo URL
    targetRevision: 15.x.x # Chart version
    helm:
      valuesObject:
        replicaCount: 2
        service:
          type: ClusterIP
  destination:
    server: "https://kubernetes.default.svc"
    namespace: nginx
```

> **💡 Important:** ArgoCD uses `helm template` to render charts, **not** `helm install`. Helm is only used as a templating engine — ArgoCD manages the full lifecycle. This means `helm list` will show nothing.

---

## Using Helm with ArgoCD

> 📖 Source: [ArgoCD Helm Docs](https://argo-cd.readthedocs.io/en/latest/user-guide/helm/)

### How It Works

ArgoCD is [only used to inflate charts with `helm template`](https://argo-cd.readthedocs.io/en/latest/faq/#after-deploying-my-helm-application-with-argo-cd-i-cannot-see-it-with-helm-ls-and-other-helm-commands). The lifecycle of the application is handled by ArgoCD instead of Helm. This means:

- No Helm releases are created — `helm list` shows nothing
- ArgoCD manages sync, diff, rollback — not Helm
- Every operation is a **sync** (ArgoCD cannot distinguish between "install" and "upgrade")

### Passing Values to Helm Charts

There are three ways to provide values in an ArgoCD Application:

#### 1. `valuesObject` — Inline YAML (recommended)

```yaml
source:
  helm:
    valuesObject:
      replicaCount: 3
      service:
        type: ClusterIP
      ingress:
        enabled: true
        hostname: my-app.example.com
```

#### 2. `valueFiles` — Reference files in the repo

```yaml
source:
  helm:
    valueFiles:
      - values-production.yaml
```

> **Note:** Since ArgoCD v2.6, values files can be sourced from a **separate repository** using [multiple sources](https://argo-cd.readthedocs.io/en/latest/user-guide/multiple_sources/#helm-value-files-from-external-git-repository).

#### 3. `parameters` — Like `--set` flags

```yaml
source:
  helm:
    parameters:
      - name: "service.type"
        value: LoadBalancer
      - name: "replicaCount"
        value: "3"
```

Or via CLI:

```bash
argocd app set my-app -p service.type=LoadBalancer
```

### Value Precedence

When multiple value sources are used, the order is (highest wins):

```
parameters > valuesObject > values > valueFiles > chart's values.yaml
```

| Priority   | Source                        | Equivalent in Helm CLI |
| ---------- | ----------------------------- | ---------------------- |
| 🥇 Highest | `parameters`                  | `--set key=value`      |
| 🥈         | `valuesObject`                | Inline YAML values     |
| 🥉         | `values` (string)             | Inline values string   |
| 4th        | `valueFiles`                  | `-f values-prod.yaml`  |
| 🔽 Lowest  | Chart's default `values.yaml` | Default chart values   |

### Release Name

By default, the Helm release name equals the Application name. You can override it:

```yaml
source:
  helm:
    releaseName: my-custom-release
```

> ⚠️ **Warning:** Overriding the release name can cause issues with `app.kubernetes.io/instance` label tracking. ArgoCD injects this label with the Application name, not the release name.

### Helm Hooks

ArgoCD maps Helm hooks to its own hook system:

| Helm Hook      | ArgoCD Hook     |
| -------------- | --------------- |
| `pre-install`  | `PreSync`       |
| `pre-upgrade`  | `PreSync`       |
| `post-install` | `PostSync`      |
| `post-upgrade` | `PostSync`      |
| `pre-delete`   | `PreDelete`     |
| `post-delete`  | `PostDelete`    |
| `pre-rollback` | _(unsupported)_ |
| `test-success` | _(unsupported)_ |

> ⚠️ If you define any ArgoCD hooks, **all Helm hooks will be ignored**.

---

## Sync Policies

| Policy        | Description                                                                                |
| ------------- | ------------------------------------------------------------------------------------------ |
| **Manual**    | You click "Sync" in the UI or run `argocd app sync`. Default behavior.                     |
| **Automated** | ArgoCD auto-syncs when it detects the live state is OutOfSync from Git.                    |
| **Prune**     | Automatically delete K8s resources that were removed from Git. Must be explicitly enabled. |
| **Self-Heal** | Revert any manual changes made directly on the cluster (e.g., `kubectl edit`).             |

```yaml
syncPolicy:
  automated:
    prune: true # Remove resources deleted from Git
    selfHeal: true # Revert manual cluster changes
```

---

## Automatic Image Updates

**Problem:** After building a new Docker image, how does ArgoCD know to update the deployment?

### Strategy 1: Manual Tag Updates (Not Recommended)

Update image tag in Git after each build:

```yaml
# apps/dev/web-frontend.yaml
image:
  tag: "abc123" # Update this manually after build
```

❌ Requires manual Git commits  
❌ Breaks automation

### Strategy 2: Stable Tags + Always Pull (✅ Recommended for Dev)

Use a **stable tag** that gets overwritten:

```yaml
# apps/dev/web-frontend.yaml
valuesObject:
  image:
    tag: "dev" # Static tag (dev, staging, main, etc.)
    pullPolicy: Always # Force pull even if tag exists
```

**How it works:**

1. Build pushes to `skimpjr/web-frontend:dev` (overwrites existing)
2. ArgoCD checks image digest periodically
3. Detects digest changed (even though tag is same)
4. Triggers deployment rollout
5. Pods restart with `imagePullPolicy: Always` → pulls new image

✅ Fully automatic  
✅ No Git commits needed  
✅ Works in local dev  
⚠️ Not suitable for production (no rollback capability)

### Strategy 3: ArgoCD Image Updater (Production)

For production, use [ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/):

- Watches Docker registry for new semantic version tags
- Automatically updates Git with new image versions
- Supports rollback (Git history)
- Requires proper tagging strategy (semver)

**Example workflow:**

```
Build tags image: v1.2.3
  ↓
Image Updater detects new tag
  ↓
Updates Git: image.tag: "v1.2.3"
  ↓
ArgoCD syncs
  ↓
Deployment updated
```

### Our Implementation

**File:** [`apps/dev/web-frontend.yaml`](../../infrastructure/apps/dev/web-frontend.yaml)

```yaml
valuesObject:
  image:
    tag: "dev"
    pullPolicy: Always
```

**Build template:** [`argo-workflows/frontend-build-template.yaml`](../../infrastructure/argo-workflows/frontend-build-template.yaml)

```yaml
args:
  - "--destination=skimpjr/web-frontend:{{inputs.parameters.environment}}"
  # Pushes to 'dev' tag, overwrites on each build
```

**Complete flow:**

```
git push (web-frontend)
  ↓
CronWorkflow detects (every 2 min)
  ↓
Updates ConfigMap
  ↓
Sensor triggers build
  ↓
Kaniko pushes skimpjr/web-frontend:dev
  ↓
ArgoCD detects image digest change
  ↓
Deployment updates automatically! ✅
```

---

## Common CLI Commands

```bash
# Login to ArgoCD
argocd login localhost:9000

# List applications
argocd app list

# Get app details
argocd app get my-app

# Sync an application
argocd app sync my-app

# View app diff (what would change)
argocd app diff my-app

# Delete an application
argocd app delete my-app
```

---

## How We'll Use It

In our setup, ArgoCD manages deployments via Helm:

```
┌─────────────────────────────────────────────────────┐
│                  Our GitOps Flow                     │
│                                                      │
│  1. ArgoCD itself     → Installed via Helm           │
│  2. backend-server    → Helm chart → ArgoCD App      │
│  3. web-frontend      → Helm chart → ArgoCD App      │
│                                                      │
│  Git repo change → ArgoCD detects → Auto-syncs       │
└─────────────────────────────────────────────────────┘
```

The Helm charts define **what** gets deployed (Deployment, Service, Ingress), and ArgoCD ensures the cluster **always matches** what's in Git. Any drift is automatically detected and corrected.

---

## Next Steps

- [Argo Workflows](./02-argo-workflows.md) — build Docker images inside the cluster as part of your pipeline
- [Argo Events](./03-argo-events.md) — trigger workflows automatically on git push or other events
- [PLG Stack](../04-observability/01-plg-stack.md) — monitor your ArgoCD-managed applications
