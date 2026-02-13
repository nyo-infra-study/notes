# Argo Workflows

> Kubernetes-native CI/CD for building Docker images inside the cluster

## What & Why

**Problem:** Our React frontend needs different `VITE_API_URL` for dev/staging/prod. Vite bakes these at **build time**, so we can't just change an env var.

**Solution:** Argo Workflows builds images **inside Kubernetes** with environment-specific config.

```
Local Build ❌              Argo Workflows ✅
└─ docker build            └─ Builds in cluster
   └─ One env only            └─ Any env on demand
   └─ Requires Docker         └─ No Docker needed
   └─ Manual process          └─ Automated via events
```

---

## Core Concepts

### 1. WorkflowTemplate (Reusable)

Like a function - define once, call many times.

**Our implementation:** [`infrastructure/argo-workflows/frontend-build-template.yaml`](../../infrastructure/argo-workflows/frontend-build-template.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: build-frontend-template
spec:
  arguments:
    parameters:
      - name: environment # dev, staging, prod
      - name: vite-api-url # http://localhost:8080/api
      - name: image-tag # Commit SHA

  templates:
    - name: build-and-push
      container:
        image: gcr.io/kaniko-project/executor:latest
        args:
          - "--context=git://github.com/nyo-infra-study/infrastructure#main"
          - "--context-sub-path=web-frontend"
          - "--build-arg=VITE_API_URL={{inputs.parameters.vite-api-url}}"
          - "--destination=skimpjr/web-frontend:{{inputs.parameters.image-tag}}"
```

**Key points:**

- **Parameters** = Variables you can override
- **Kaniko** = Builds Docker images without Docker daemon
- **Build args** = Passed to Dockerfile `ARG VITE_API_URL`

### 2. Kaniko

Builds Docker images inside Kubernetes pods (no Docker daemon needed).

```yaml
container:
  image: gcr.io/kaniko-project/executor:latest
  args:
    - "--context=git://repo#branch" # Clone from Git
    - "--context-sub-path=web-frontend" # Dockerfile location
    - "--build-arg=VAR=value" # Pass to Dockerfile
    - "--destination=dockerhub/image:tag" # Push here
```

### 3. Workflow vs WorkflowTemplate

| Type                 | Use Case                         | Managed by ArgoCD |
| -------------------- | -------------------------------- | ----------------- |
| **Workflow**         | One-off execution                | ❌ No             |
| **WorkflowTemplate** | Reusable, called by Sensors/Cron | ✅ Yes            |

**Always use WorkflowTemplates for production!**

---

## Our Setup

### Directory Structure

```
infrastructure/
├── apps/dev/
│   └── argo-workflows.yaml              # ArgoCD manages argo-workflows/*
│
└── argo-workflows/
    └── frontend-build-template.yaml     # Reusable build definition
```

### ArgoCD Integration

**File:** [`infrastructure/apps/dev/argo-workflows.yaml`](../../infrastructure/apps/dev/argo-workflows.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-workflows
spec:
  source:
    path: argo-workflows # ArgoCD watches this folder
  destination:
    namespace: argo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**What this does:**

- ArgoCD watches `argo-workflows/` folder
- Any changes to YAML files auto-sync to cluster
- You just commit to Git - ArgoCD deploys it

---

## How It Works

### The Build Flow

```
1. Git Push
   ↓
2. Sensor detects (Argo Events)
   ↓
3. Creates Workflow from WorkflowTemplate
   ↓
4. Kaniko clones repo
   ↓
5. Builds Dockerfile with VITE_API_URL
   ↓
6. Pushes image to Docker Hub
```

### Example: Automated Build

When you push code, Argo Events triggers this workflow:

```yaml
# Sensor creates Workflow
spec:
  workflowTemplateRef:
    name: build-frontend-template
  arguments:
    parameters:
      - name: environment
        value: "dev"
      - name: vite-api-url
        value: "http://localhost:8080/api"
      - name: image-tag
        value: "abc123" # Commit SHA from webhook
```

See: [Argo Events](./argo-events.md) for automation details.

---

## Installation

```bash
# 1. Create namespace
kubectl create namespace argo

# 2. Install (use --server-side for large CRDs)
kubectl apply -n argo \
  -f https://github.com/argoproj/argo-workflows/releases/download/v4.0.0/install.yaml \
  --server-side

# 3. Create Docker Hub secret (for Kaniko)
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_TOKEN \
  -n argo

# 4. Verify
kubectl get pods -n argo
```

---

## Common Commands

```bash
# List workflows
kubectl get workflows -n argo

# Watch workflow execution
kubectl get workflows -n argo -w

# Get workflow logs
kubectl logs -n argo -l workflows.argoproj.io/workflow=frontend-build-abc123

# List templates
kubectl get workflowtemplates -n argo

# Delete completed workflows
kubectl delete workflows -n argo --field-selector status.phase=Succeeded
```

---

## CronWorkflows: Polling Git for Changes

**Problem:** Webhooks require internet-accessible endpoints. In local dev, we need **polling**.

**Solution:** CronWorkflow checks GitHub every 2 minutes for new commits.

**Our implementation:** [`infrastructure/argo-workflows/github-poller.yaml`](../../infrastructure/argo-workflows/github-poller.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: github-poller
spec:
  schedule: "*/2 * * * *" # Every 2 minutes
  workflowSpec:
    entrypoint: poll-github
    templates:
      - name: poll-github
        script:
          image: alpine/git
          command: [sh]
          source: |
            # Get latest commit SHA from GitHub
            LATEST_COMMIT=$(git ls-remote https://github.com/nyo-infra-study/web-frontend.git refs/heads/main | cut -f1)

            # Update ConfigMap (triggers EventSource → Sensor → Build)
            kubectl create configmap github-latest-commit \
              --from-literal=commit=$LATEST_COMMIT \
              --dry-run=client -o yaml | kubectl apply -f -
```

### How Polling Works

```
┌─────────────────────────────────────────────────────────────┐
│  CronWorkflow (every 2 min)                                  │
│    ↓                                                         │
│  git ls-remote → Get commit SHA                              │
│    ↓                                                         │
│  Update ConfigMap with new commit                            │
│    ↓                                                         │
│  EventSource detects ConfigMap change                        │
│    ↓                                                         │
│  Sensor triggers build Workflow                              │
│    ↓                                                         │
│  Kaniko builds & pushes image with 'dev' tag                 │
│    ↓                                                         │
│  ArgoCD detects new image (ImagePullPolicy: Always)          │
│    ↓                                                         │
│  Deployment automatically updates! ✅                         │
└─────────────────────────────────────────────────────────────┘
```

**Key points:**

- **No webhooks needed** - Works in local k3d clusters
- **ConfigMap as bridge** - Polling writes, EventSource reads
- **RBAC required** - Workflow needs permission to manage ConfigMaps

---

## Automated Deployments with Stable Tags

**Problem:** Using commit SHA tags requires manual image tag updates in Git after each build.

**Solution:** Use a **stable tag** (like `dev`, `staging`) that gets overwritten on each build.

### Build Template Configuration

```yaml
args:
  - "--destination=skimpjr/web-frontend:{{inputs.parameters.environment}}"
  # This pushes to skimpjr/web-frontend:dev (overwrites each time)
```

### ArgoCD Configuration

```yaml
# apps/dev/web-frontend.yaml
valuesObject:
  image:
    tag: "dev" # Static tag, always points to latest
    pullPolicy: Always # Force pull on every deployment
```

### Why This Works

1. **Build pushes to `skimpjr/web-frontend:dev`** - Same tag, new image content
2. **`imagePullPolicy: Always`** - Kubernetes pulls the image even if tag exists
3. **ArgoCD detects image change** - Sees the image digest changed
4. **Deployment restarts** - New pods pull the updated `dev` image

**Result:** Push to main → Auto-build → Auto-deploy (no manual steps!) 🎉

---

## Troubleshooting

### Workflow Stuck in Pending

**Check:**

```bash
kubectl describe pod -n argo frontend-build-xxx
```

**Common causes:**

- Missing `docker-credentials` secret
- Not enough cluster resources
- Image pull error

### Kaniko Build Fails

**Check logs:**

```bash
kubectl logs -n argo -l workflows.argoproj.io/workflow=frontend-build-xxx
```

**Common causes:**

- Wrong Git URL or branch
- `context-sub-path` doesn't exist
- Dockerfile missing `ARG` for build arg

---

## Key Takeaways

✅ **WorkflowTemplates** - Reusable, ArgoCD-managed  
✅ **Kaniko** - Builds without Docker daemon  
✅ **Parameters** - Environment-specific builds  
✅ **GitOps** - Commit → ArgoCD syncs → Ready to use

**Next:** [Argo Events](./argo-events.md) for automated triggering

## References

- [Official Docs](https://argo-workflows.readthedocs.io/)
- [Kaniko](https://github.com/GoogleContainerTools/kaniko)
- [Our Implementation](../../infrastructure/argo-workflows/)
