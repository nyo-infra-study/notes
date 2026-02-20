# Argo Events

> Event-driven automation for Kubernetes - trigger workflows on git push, webhooks, etc.

## What & Why

**Problem:** Manually running workflows after every git push is tedious.

```bash
# Every time you push code:
git push
argo submit --from workflowtemplate/build-frontend -p image-tag=$(git rev-parse HEAD)
# Update image tag in Git...
# Push again...
```

**Solution:** Argo Events listens to events (webhooks or polling) and automatically triggers workflows.

```
# Webhook approach (requires public endpoint):
git push → GitHub webhook → EventSource → Sensor → Workflow → Build + Push

# Polling approach (works locally):
git push → CronWorkflow polls → Updates ConfigMap → EventSource → Sensor → Workflow
```

---

## Architecture

```
GitHub
  │
  ├─ Webhook: POST /push
  │
  ▼
EventSource (receives webhook)
  │
  ├─ Publishes to EventBus (NATS)
  │
  ▼
Sensor (listens & filters)
  │
  ├─ Extracts commit SHA
  │
  ▼
Creates Workflow
  │
  ▼
Kaniko builds image
```

---

## Core Components

### 1. EventSource

Receives events from external systems (GitHub, Slack, SQS, etc.)

**Our implementation:** [`infrastructure/argo-events/github-event-source.yaml`](../../infrastructure/argo-events/github-event-source.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github
  namespace: argo
spec:
  github:
    frontend-repo:
      repositories:
        - owner: nyo-infra-study
          names:
            - infrastructure

      webhook:
        endpoint: /push
        port: "12000"

      events:
        - push

      apiToken:
        name: github-access # Secret with GitHub token
        key: token
```

### 2. EventBus

Event transport using NATS - routes events from EventSource to Sensors.

```bash
# Creates 3 NATS pods for high availability
kubectl apply -n argo \
  -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```

### 3. Sensor

Listens to events and triggers actions (creates Workflows).

**Our implementation:** [`infrastructure/argo-events/frontend-build-sensor.yaml`](../../infrastructure/argo-events/frontend-build-sensor.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: frontend-build-trigger
  namespace: argo
spec:
  dependencies:
    - name: github-push
      eventSourceName: github
      eventName: frontend-repo

  triggers:
    - template:
        k8s:
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              spec:
                workflowTemplateRef:
                  name: build-frontend-template
                arguments:
                  parameters:
                    - name: vite-api-url
                      value: "http://localhost:9000/api"
                    - name: image-tag
                      value: "latest" # Overridden by commit SHA below

          # Extract commit SHA from GitHub webhook
          parameters:
            - src:
                dependencyName: github-push
                dataKey: body.after
              dest: spec.arguments.parameters.2.value
```

**Key points:**

- **dependencies** = What events to listen for
- **triggers** = What to do when event happens
- **parameters** = Extract data from webhook (commit SHA)

---

## Our Setup

### Directory Structure

```
infrastructure/
├── apps/dev/
│   └── argo-events.yaml                 # ArgoCD manages argo-events/*
│
└── argo-events/
    ├── github-event-source.yaml         # Receives GitHub webhooks
    └── frontend-build-sensor.yaml       # Triggers workflow
```

### ArgoCD Integration

**File:** [`infrastructure/apps/dev/argo-events.yaml`](../../infrastructure/apps/dev/argo-events.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-events
spec:
  source:
    path: argo-events # ArgoCD watches this folder
  destination:
    namespace: argo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Installation

```bash
# 1. Install Argo Events
kubectl create namespace argo-events
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml

# 2. Install EventBus (NATS)
kubectl apply -n argo \
  -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml

# 3. Create GitHub token secret
kubectl create secret generic github-access \
  -n argo \
  --from-literal=token=ghp_your_github_token

# 4. Verify
kubectl get pods -n argo-events    # Controller
kubectl get eventbus -n argo       # NATS pods
```

**GitHub token scopes needed:**

- ✅ `repo` (access repositories)
- ✅ `admin:repo_hook` (manage webhooks)

Generate at: https://github.com/settings/tokens/new

---

## How It Works

### The Complete Flow

```
Developer
  │
  ├─ git commit -m "feat: new feature"
  ├─ git push origin main
  │
  ▼
GitHub sends webhook
  │
  ├─ POST /push
  │   {
  │     "ref": "refs/heads/main",
  │     "after": "abc123...",  // Commit SHA
  │     ...
  │   }
  │
  ▼
EventSource receives
  │
  ├─ Validates signature
  ├─ Publishes to EventBus
  │
  ▼
Sensor detects event
  │
  ├─ Filters: only "push" events
  ├─ Extracts: commit SHA from body.after
  │
  ▼
Sensor creates Workflow
  │
  ├─ References: build-frontend-template
  ├─ Overrides: image-tag = abc123
  │
  ▼
Kaniko builds image
  │
  ├─ Tags: skimpjr/web-frontend:abc123
  ├─ Pushes to Docker Hub
  │
  ▼
Developer updates deployment
  │
  ├─ vim apps/dev/web-frontend.yaml
  ├─ image.tag: "abc123"
  ├─ git push
  │
  ▼
ArgoCD syncs
  │
  └─ Kubernetes pulls new image
```

---

## Common Commands

```bash
# Check EventSource
kubectl get eventsources -n argo
kubectl logs -n argo -l eventsource-name=github

# Check Sensor
kubectl get sensors -n argo
kubectl logs -n argo -l sensor-name=frontend-build-trigger

# Check EventBus
kubectl get eventbus -n argo
kubectl get pods -n argo | grep eventbus

# List triggered workflows
kubectl get workflows -n argo
```

---

## Polling Alternative: ConfigMap EventSource

**When to use:** Local development without public webhooks.

**How it works:**

1. **CronWorkflow** polls GitHub every 2 minutes
2. Writes latest commit SHA to **ConfigMap**
3. **EventSource** watches ConfigMap changes
4. **Sensor** triggers build on new commits

### ConfigMap EventSource

**Our implementation:** [`infrastructure/argo-events/github-event-source.yaml`](../../infrastructure/argo-events/github-event-source.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github-commits
spec:
  resource:
    github-commits:
      namespace: argo
      group: ""
      version: v1
      resource: configmaps
      eventTypes:
        - ADD
        - UPDATE
      filter:
        labels:
          - key: app
            value: github-poller
```

### Sensor Configuration

**Our implementation:** [`infrastructure/argo-events/frontend-build-sensor.yaml`](../../infrastructure/argo-events/frontend-build-sensor.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: frontend-build-trigger
spec:
  dependencies:
    - name: github-commit
      eventSourceName: github-commits
      eventName: github-commits

  triggers:
    - template:
        name: build-frontend
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              spec:
                workflowTemplateRef:
                  name: build-frontend-template
```

**Key differences from webhooks:**

- ✅ No webhook configuration needed
- ✅ Works in local clusters (no public IP)
- ✅ Simple RBAC (just ConfigMap read)
- ⚠️ 2-minute delay (polling interval)
- ⚠️ More API calls to GitHub

**Best for:** Local dev, testing, environments without webhook access

---

## Troubleshooting

### EventSource Pod Not Starting

**Symptom:** Pod stuck in `ContainerCreating`

**Debug:**

```bash
kubectl describe pod -n argo -l eventsource-name=github
```

**Common causes:**

- Missing `github-access` secret
- EventBus not ready (NATS pods not running)

**Fix:**

```bash
# Check secret exists
kubectl get secret github-access -n argo

# Check EventBus
kubectl get pods -n argo | grep eventbus
```

### Sensor Crashing

**Symptom:** `CrashLoopBackOff` with error:

```
dial tcp: lookup eventbus-default-stan-svc: no such host
```

**Cause:** EventBus pods weren't ready when sensor started

**Fix:**

```bash
# Wait for EventBus
kubectl get pods -n argo | grep eventbus
# All should be Running

# Restart sensor
kubectl delete pod -n argo -l sensor-name=frontend-build-trigger
```

### Workflow Not Triggering

**Symptom:** Git push but no workflow created

**Debug:**

```bash
# Check EventSource received webhook
kubectl logs -n argo -l eventsource-name=github --tail=50

# Check Sensor detected event
kubectl logs -n argo -l sensor-name=frontend-build-trigger --tail=50
```

**Common causes:**

- GitHub webhook not configured
- Event filtered out (wrong branch, etc.)
- Sensor dependencies mismatch

---

## Key Takeaways

✅ **EventSource** - Receives webhooks from GitHub  
✅ **EventBus** - Routes events via NATS  
✅ **Sensor** - Triggers workflows automatically  
✅ **GitOps** - Manage in Git, ArgoCD syncs  
✅ **Commit SHA** - Auto-extracted as image tag

**Result:** Push code → Image automatically built with commit SHA!

## References

- [Official Docs](https://argoproj.github.io/argo-events/)
- [Event Sources](https://argoproj.github.io/argo-events/eventsources/setup/github/)
- [Sensors](https://argoproj.github.io/argo-events/sensors/sensors/)
- [Our Implementation](../../infrastructure/argo-events/)
- [Argo Workflows](./argo-workflows.md) for workflow definitions
