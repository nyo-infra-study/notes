# Helm

> 📖 Official Docs: [https://helm.sh/docs/](https://helm.sh/docs/) · Last updated: 2026-02-12

Helm is the **package manager for Kubernetes**. It simplifies deploying and managing applications on a K8s cluster by bundling Kubernetes manifests into reusable packages called **Charts**.

> **Think of it as:** `brew` for macOS, but for Kubernetes.

## Core Concepts

| Concept        | Description                                                                                     |
| -------------- | ----------------------------------------------------------------------------------------------- |
| **Chart**      | A package of pre-configured Kubernetes resources (Deployments, Services, etc.).                 |
| **Release**    | A running instance of a Chart deployed to a cluster. You can have multiple releases of a chart. |
| **Repository** | A place where Charts are stored and shared (like a package registry).                           |
| **Values**     | Configuration that overrides Chart defaults. Passed via `values.yaml` or `--set` flags.         |
| **Template**   | Kubernetes manifest files with Go templating (`{{ .Values.x }}`) that Helm renders into YAML.   |

---

## How Helm Works

```
┌─────────────┐     helm install      ┌──────────────┐     kubectl apply     ┌─────────────┐
│  Chart Repo  │ ──────────────────▶  │  Helm Engine  │ ──────────────────▶  │  K8s Cluster │
│  (or local)  │   + values.yaml      │  (templating) │   rendered manifests  │  (Pods, Svc) │
└─────────────┘                       └──────────────┘                       └─────────────┘
```

1. You provide a **Chart** (from a repo or local directory) + your **Values**.
2. Helm **renders** the templates into plain Kubernetes YAML manifests.
3. Helm **applies** those manifests to the cluster via the Kubernetes API.
4. Helm **tracks** the release so you can upgrade, rollback, or uninstall later.

---

## Main Components

> 📖 Source: [Helm Architecture](https://helm.sh/docs/topics/architecture/)

Helm is implemented as a single executable with two distinct parts:

### 1. Helm Client (CLI)

The command-line tool you interact with directly. It is responsible for:

- **Local chart development** — Creating and packaging charts.
- **Managing repositories** — Adding, updating, and searching chart repos.
- **Managing releases** — Installing, upgrading, rolling back, and uninstalling.
- **Sending requests** to the Helm Library for execution.

### 2. Helm Library (SDK)

The internal engine that does the actual work. It interfaces with the **Kubernetes API server** and provides:

- **Combining** a chart + configuration to build a release.
- **Installing** charts into Kubernetes and producing the release object.
- **Upgrading and uninstalling** charts by interacting with the K8s API.

> **💡 Key Detail:** Helm stores release information as **Secrets** inside the Kubernetes cluster itself. It does **not** need its own database. This means release history lives in the cluster and is visible via `kubectl get secrets -l owner=helm`.

### 3. Three Core Concepts

| Concept     | Description                                                                                          |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| **Chart**   | A bundle of information necessary to create an instance of a Kubernetes application.                 |
| **Config**  | Configuration that gets merged into a chart to create a releasable object (`values.yaml` + `--set`). |
| **Release** | A running instance of a chart, combined with a specific config. Each install creates a new release.  |

```
                  ┌──────────┐
                  │  Chart   │ ─── templates, defaults
                  └────┬─────┘
                       │  merge
                  ┌────▼─────┐
                  │  Config  │ ─── values.yaml, --set overrides
                  └────┬─────┘
                       │  produces
                  ┌────▼─────┐
                  │ Release  │ ─── running instance on the cluster
                  └──────────┘
```

---

## Installation

📖 **Official Docs:** [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

```bash
# macOS (via Homebrew — recommended)
brew install helm

# From official install script (Linux/macOS)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

# Or one-liner
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash

# Verify installation
helm version
```

Other package managers:

| OS / Manager         | Command                                                                                    |
| -------------------- | ------------------------------------------------------------------------------------------ |
| **macOS** (Homebrew) | `brew install helm`                                                                        |
| **Windows** (Choco)  | `choco install kubernetes-helm`                                                            |
| **Windows** (Scoop)  | `scoop install helm`                                                                       |
| **Windows** (Winget) | `winget install Helm.Helm`                                                                 |
| **Debian/Ubuntu**    | See [official Apt instructions](https://helm.sh/docs/intro/install/#from-apt-debianubuntu) |
| **Fedora**           | `sudo dnf install helm`                                                                    |
| **Snap**             | `sudo snap install helm --classic`                                                         |

---

## Repository Management

Repositories are where Helm Charts are hosted. You need to add a repo before you can install charts from it.

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update local cache (fetch latest chart versions)
helm repo update

# Search for charts in added repos
helm search repo nginx

# List added repositories
helm repo list

# Remove a repository
helm repo remove bitnami
```

### Common Repositories

| Repository        | URL                                          | Notes                                    |
| ----------------- | -------------------------------------------- | ---------------------------------------- |
| **Bitnami**       | `https://charts.bitnami.com/bitnami`         | Huge library of production-ready charts. |
| **Cilium**        | `https://helm.cilium.io/`                    | Official Cilium CNI chart.               |
| **ArgoCD**        | `https://argoproj.github.io/argo-helm`       | Official ArgoCD charts.                  |
| **Ingress-Nginx** | `https://kubernetes.github.io/ingress-nginx` | Official NGINX Ingress Controller.       |
| **Jetstack**      | `https://charts.jetstack.io`                 | cert-manager and related tools.          |

---

## Chart Operations

### Installing a Chart

```bash
# Install a chart from a repo (creates a "release")
helm install my-release bitnami/nginx

# Install with a custom values file
helm install my-release bitnami/nginx -f my-values.yaml

# Install with inline value overrides
helm install my-release bitnami/nginx --set service.type=ClusterIP

# Install into a specific namespace (creates it if --create-namespace is used)
helm install my-release bitnami/nginx -n my-namespace --create-namespace

# Dry run (preview rendered manifests without applying)
helm install my-release bitnami/nginx --dry-run
```

### Upgrading a Release

```bash
# Upgrade with new values
helm upgrade my-release bitnami/nginx -f updated-values.yaml

# Upgrade with install fallback (install if release doesn't exist yet)
helm upgrade --install my-release bitnami/nginx -f values.yaml
```

> **💡 Tip:** `helm upgrade --install` is the most common pattern in CI/CD pipelines. It is idempotent — it installs on first run and upgrades on subsequent runs.

### Rollback

```bash
# View release history
helm history my-release

# Rollback to a specific revision
helm rollback my-release 1
```

### Uninstalling

```bash
# Uninstall a release (removes all K8s resources created by it)
helm uninstall my-release

# Uninstall from a specific namespace
helm uninstall my-release -n my-namespace
```

### Inspecting

```bash
# List all releases in the current namespace
helm list

# List all releases across all namespaces
helm list -A

# Show the values used by a release
helm get values my-release

# Show all rendered manifests of a release
helm get manifest my-release

# Show details of a chart before installing
helm show values bitnami/nginx
```

---

## Chart Structure

> 📖 Source: [Charts](https://helm.sh/docs/topics/charts/)

A Chart is a **directory** containing everything needed to deploy an application. "Bundle of information" simply means: all the Kubernetes YAML manifests, default config, metadata, and dependencies are packaged together in one folder.

```
my-app/
├── Chart.yaml          # Metadata (name, version, description)
├── values.yaml         # Default configuration values
├── templates/          # Kubernetes manifest templates (Go-templated)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Reusable template snippets
│   └── NOTES.txt       # Post-install instructions shown to user
├── charts/             # Sub-charts (dependencies)
├── crds/               # Custom Resource Definitions
└── .helmignore         # Files to ignore when packaging
```

### Example: `Chart.yaml`

The **identity card** of the chart. Only `apiVersion`, `name`, and `version` are required.

```yaml
apiVersion: v2 # Required: Chart API version (v2 for Helm 3+)
name: my-app # Required: Name of the chart
version: 0.1.0 # Required: Chart version (SemVer)
appVersion: "1.0.0" # Version of the app being deployed
description: A simple web server
type: application # "application" or "library"
keywords:
  - web
  - nginx
dependencies: # Other charts this chart depends on
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

### Example: `values.yaml`

The **default configuration**. Users can override any of these values.

```yaml
# Default values for my-app
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  hostname: my-app.local

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

### Example: `templates/deployment.yaml`

A **Go-templated** Kubernetes manifest. Helm replaces `{{ .Values.x }}` with actual values at install time.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app     # Uses the release name (e.g., "my-release-app")
  labels:
    app: {{ .Chart.Name }}           # Uses the chart name from Chart.yaml
spec:
  replicas: {{ .Values.replicaCount }}  # Reads from values.yaml → replicaCount: 2
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # ↑ Renders to: "nginx:1.25"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
            # ↑ Renders the resources block from values.yaml
```

### What Helm Actually Does

When you run `helm install my-release ./my-app`, Helm takes the template above and produces this **plain YAML**:

```yaml
# This is what gets applied to Kubernetes — no more {{ }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-release-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: "nginx:1.25"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
            requests:
              cpu: 50m
              memory: 64Mi
```

> **💡 Key Takeaway:** A Chart bundles templates + defaults so that anyone can deploy the same app with different configurations just by changing `values.yaml` — no need to edit raw Kubernetes manifests.

---

## Values & Overrides

Values follow a priority order (highest wins):

1. `--set` flags on the command line
2. `-f custom-values.yaml` files (last file wins if multiple)
3. Default `values.yaml` in the chart

```bash
# Check what values a chart accepts
helm show values bitnami/nginx

# Override a nested value
helm install my-release bitnami/nginx --set controller.replicaCount=3

# Override multiple values
helm install my-release bitnami/nginx \
  --set service.type=ClusterIP \
  --set replicaCount=2
```

> **💡 Best Practice:** For anything beyond a quick test, always use a `values.yaml` file instead of `--set`. It's easier to version control and review.

---

## Common Workflows

### Installing Cilium via Helm

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set ingressController.enabled=true
```

### Installing ArgoCD via Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace
```

### Installing NGINX Ingress Controller via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

---

## Helm vs. Raw Manifests

| Feature             | **Raw `kubectl apply`**   | **Helm**                                |
| ------------------- | ------------------------- | --------------------------------------- |
| **Templating**      | ❌ None (static YAML)     | ✅ Go templates with values             |
| **Versioning**      | ❌ Manual                 | ✅ Built-in release history & rollbacks |
| **Reusability**     | ❌ Copy-paste             | ✅ Parameterized charts                 |
| **Dependency Mgmt** | ❌ Manual ordering        | ✅ Sub-charts & hooks                   |
| **Ecosystem**       | N/A                       | ✅ Thousands of community charts        |
| **Complexity**      | ✅ Simple and transparent | ⚠️ Adds a layer of abstraction          |

> **When to use raw manifests?** For very simple deployments or when you want full transparency (e.g., learning K8s basics).
> **When to use Helm?** For anything production-level, reusable, or involving third-party tools (Cilium, ArgoCD, etc.).

---

## Helm with ArgoCD

> 📖 Source: [ArgoCD Helm Docs](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)

### How ArgoCD Uses Helm (Important Distinction)

ArgoCD does **not** run `helm install`. Instead, it uses `helm template` to render the chart into plain YAML, then applies those manifests itself via the Kubernetes API.

This means:

- **No Helm releases** are created — `helm list` will show nothing.
- **ArgoCD manages the lifecycle**, not Helm (sync, diff, rollback are all ArgoCD features).
- Helm is only used as a **templating engine**.

### Deploying a Helm Chart via ArgoCD Application

Instead of running `helm install`, you create an ArgoCD `Application` resource that points to a Helm chart:

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  source:
    chart: sealed-secrets # Chart name
    repoURL: https://bitnami-labs.github.io/sealed-secrets # Helm repo URL
    targetRevision: 1.16.1 # Chart version
    helm:
      releaseName: sealed-secrets # Equivalent of helm install <name>
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubeseal
```

### Passing Values to Helm Charts

ArgoCD provides multiple ways to override chart values, similar to `helm install --set` or `-f values.yaml`:

#### 1. Inline values (valuesObject) — Recommended for declarative setups

```yaml
source:
  chart: nginx
  repoURL: https://charts.bitnami.com/bitnami
  targetRevision: 15.x.x
  helm:
    valuesObject:
      replicaCount: 3
      service:
        type: ClusterIP
      ingress:
        enabled: true
        hostname: my-app.example.com
```

#### 2. Values files (valueFiles) — Reference files in the same repo

```yaml
source:
  helm:
    valueFiles:
      - values-production.yaml
```

#### 3. Parameters (parameters) — Like `--set` flags

```yaml
source:
  helm:
    parameters:
      - name: "service.type"
        value: LoadBalancer
      - name: "replicaCount"
        value: "3"
```

### Value Precedence in ArgoCD

When multiple value sources are used, ArgoCD follows this order (highest wins):

```
parameters > valuesObject > values > valueFiles > chart's values.yaml
```

| Priority   | Source                        | Equivalent in Helm CLI  |
| ---------- | ----------------------------- | ----------------------- |
| 🥇 Highest | `parameters`                  | `--set key=value`       |
| 🥈         | `valuesObject`                | Inline YAML values      |
| 🥉         | `values` (string)             | Inline values as string |
| 4th        | `valueFiles`                  | `-f values-prod.yaml`   |
| 🔽 Lowest  | Chart's default `values.yaml` | Default chart values    |

### Full Example: Cilium via ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cilium
  namespace: argocd
spec:
  project: default
  source:
    chart: cilium
    repoURL: https://helm.cilium.io/
    targetRevision: 1.15.x
    helm:
      valuesObject:
        ingressController:
          enabled: true
        hubble:
          relay:
            enabled: true
          ui:
            enabled: true
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> **💡 Key Difference:**
>
> - `helm install` → You run it manually. Helm tracks the release.
> - ArgoCD `Application` → You commit a YAML file to Git. ArgoCD continuously syncs it. **GitOps.**

---

## Best Practices: Commenting Helm Templates

> ⚠️ **Critical Lesson Learned:** Comments before conditional blocks persist even when conditions are false!

### The Real Problem

When adding header comments **before** Helm conditional blocks, you may encounter errors like:

```
failed to discover server resources for group version : groupVersion shouldn't be empty
```

**Root Cause:** Comments placed **before** `{{- if }}` statements are outside the conditional logic and persist in the rendered template regardless of whether the condition evaluates to true or false. When the condition is false and the resource doesn't render, you end up with a YAML file containing only comments, causing Kubernetes parsing to fail.

### How It Breaks - Real Example

**Our actual problematic template:**

```yaml
# ============================================================
# Kubernetes Ingress - Backend Server
# ============================================================
# This Helm template generates a Kubernetes Ingress...
# ============================================================
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
...
{{- end }}
```

**What happens when `.Values.ingress.enabled = false`:**

The conditional block `{{- if }}...{{- end }}` completely disappears, **but the header comments before it remain** because they're outside the conditional:

```yaml
# ============================================================
# Kubernetes Ingress - Backend Server
# ============================================================
# This Helm template generates a Kubernetes Ingress...
# ============================================================
```

Result: **Empty YAML document with only comments**, no `apiVersion`.

This causes:

- `groupVersion shouldn't be empty` errors
- Kubernetes API unable to determine resource type
- ArgoCD sync failures

### ❌Primary Issue: Comments Before Conditional Blocks

```yaml
# Large header block explaining ingress
# Multiple lines of documentation
# These are NOT inside the condition!
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

**Why it fails:**

- Comments are **before** the `{{- if }}`, not inside it
- When condition is `false`, resource disappears but comments persist
- Kubernetes receives a file with comments but no actual resource

### ✅ Recommended: Inline Comments

Keep templates clean with concise inline comments:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{.Release.Name}}
  labels:
    app: {{.Release.Name}}
spec:
  replicas: {{.Values.replicaCount}} # Number of pods

  selector:
    matchLabels:
      app: {{.Release.Name}} # Must match template labels

  template:
    metadata:
      labels:
        app: {{.Release.Name}}
    spec:
      containers:
        - name: {{.Release.Name}}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{.Values.containerPort}} # App listens here
```

### Best Practices for Documentation

1. **Put comprehensive docs in `values.yaml` comments**
   - Explain each configurable value
   - This is where users look first anyway

2. **Use inline comments in templates for clarity**
   - Brief explanations next to the code
   - Keep it on the same line or directly above

3. **Create external documentation**
   - Use `README.md` in the chart directory
   - Add chart-level docs in `Chart.yaml` description
   - Keep detailed explanations in separate markdown files

4. **If you must have header comments**
   - Keep them minimal (2-3 lines max)
   - Don't use template syntax as examples in comments
   - Avoid special formatting characters

### Example: Well-Commented Chart Structure

```
my-chart/
├── Chart.yaml              # Chart metadata with description
├── README.md               # Comprehensive usage guide
├── values.yaml             # Heavily commented with explanations
└── templates/
    ├── deployment.yaml     # Clean with inline comments only
    ├── service.yaml        # Clean with inline comments only
    └── ingress.yaml        # Clean with inline comments only
```

### Safe Comment Guidelines

✅ **Safe - Inline comments:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{.Release.Name}} # Service name matches release
spec:
  type: ClusterIP # Internal only
```

✅ **Safe - Comments inside conditionals (brief):**

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    # Strip /api prefix before forwarding to backend
    {{- if .Values.ingress.stripPrefixes }}
    traefik.ingress.kubernetes.io/router.middlewares: {{ .Release.Name }}-strip@kubernetescrd
    {{- end }}
{{- end }}
```

✅ **Safe - Brief header (2-3 lines max):**

```yaml
# Service for backend API
apiVersion: v1
kind: Service
spec:
  selector:
    app: {{.Release.Name}} # Routes to matching pods
```

❌ **Dangerous - Large header in conditional:**

```yaml
{{- if .Values.ingress.enabled -}}
# ==================================
# Multi-line header with formatting
# Template syntax examples: {{- if }}
# Special chars and long explanations
# ==================================
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}
```

**Why it's dangerous:** If `ingress.enabled = false`, the block disappears but comments may persist, creating an empty YAML document.

❌ **Dangerous - Comments before conditional resource:**

```yaml
{{- with .Values.ingress.tls }}
# TLS configuration for HTTPS
# This section handles SSL certificates
tls:
  {{- range . }}
  - hosts: ...
  {{- end }}
{{- end }}
```

**Why it's dangerous:** If `tls` is empty array `[]`, the `with` block doesn't render, but comments might persist.

### The Fix: Use `with` for Empty Arrays

The issue often occurs with empty arrays. Change from `if` to `with`:

```yaml
# ❌ Problematic with empty arrays
{{- if .Values.ingress.tls }}
tls:
  {{- range .Values.ingress.tls }}
  ...
  {{- end }}
{{- end }}

# ✅ Fixed - with only renders if array is non-empty
{{- with .Values.ingress.tls }}
tls:
  {{- range . }}
  ...
  {{- end }}
{{- end }}
```

**Why this works:**

- `if []` evaluates to `true` (empty array is truthy)
- `with []` evaluates to `false` (with checks for non-empty)
- Using `with` prevents rendering empty sections entirely

### Key Takeaways

> **1. Never put large comment blocks inside conditional template blocks (`{{- if }}`, `{{- with }}`). Comments may persist even when conditions are false, creating empty YAML documents that cause parsing errors.**

> **2. Use `{{- with array }}` instead of `{{- if array }}` for arrays - empty arrays are truthy with `if` but falsy with `with`, preventing empty section rendering.**

> **3. Keep Helm templates focused on structure, not extensive documentation. Put comprehensive docs in `values.yaml`, `README.md`, or external documentation files.**

This approach keeps templates clean, avoids parsing issues, and makes documentation easier to maintain and discover.

### Real-World Example: The Bug

Here's what actually caused the `groupVersion shouldn't be empty` error in our setup:

```yaml
{{- if .Values.ingress.tls }}
# TLS/SSL Configuration (for HTTPS)
tls:
  {{- range .Values.ingress.tls | default list }}
  - hosts: ...
  {{- end }}
{{- end }}
```

**The problem:**

- `values.yaml` had `tls: []` (empty array)
- Empty array `[]` is truthy, so the `if` condition passed
- But `range` had nothing to iterate, rendering only comments
- Result: YAML document with comments but no `apiVersion`

**The solution:**

```yaml
{{- with .Values.ingress.tls }}
tls:
  {{- range . }}
  - hosts: ...
  {{- end }}
{{- end }}
```

Now the entire section doesn't render when `tls` is empty, completely avoiding the issue.
