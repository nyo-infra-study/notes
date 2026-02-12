# Kubernetes

Kubernetes (K8s) is an open-source **container orchestration platform** that automates the deployment, scaling, and management of containerized applications.

## Core Concepts

| Concept        | Description                                                               |
| -------------- | ------------------------------------------------------------------------- |
| **Cluster**    | A set of machines (nodes) that run containerized applications.            |
| **Node**       | A single machine (physical or virtual) within a cluster.                  |
| **Pod**        | The smallest deployable unit — wraps one or more containers.              |
| **Service**    | A stable network endpoint to expose and route traffic to a set of Pods.   |
| **Deployment** | A declarative way to manage Pod replicas, rolling updates, and rollbacks. |
| **Ingress**    | An API object that manages external HTTP/HTTPS access to Services.        |

---

## Container Runtimes (macOS)

Tools like **Kind** and **K3s** (when run locally via docker) require a container runtime to function. On Linux, this is native. On macOS, you need a virtualization layer.

| Feature            | **Docker Desktop**           | **Colima**               | **OrbStack**                 | **Rancher Desktop**  |
| :----------------- | :--------------------------- | :----------------------- | :--------------------------- | :------------------- |
| **Type**           | GUI App (VM-based)           | CLI Tool (Lima-based)    | Native App (Swift)           | GUI App (Lima/QEMU)  |
| **Cost**           | Free (Personal) / Paid (Biz) | 🆓 **Open Source**       | Free (Personal) / Paid (Biz) | 🆓 **Open Source**   |
| **Battery Impact** | 🔴 **High**                  | 🟡 **Medium-Low**        | 🟢 **Ultra-Low**             | 🟡 **Medium**        |
| **Speed**          | 🐢 Slow I/O                  | 🐇 Moderate              | 🐆 **Fastest**               | 🐇 Moderate          |
| **K8s Included?**  | ✅ Yes (Single node)         | ✅ Yes (k3s/k8s options) | ✅ Yes (Single node)         | ✅ Yes (k3s bundled) |
| **Setup**          | Easy Install                 | `brew install colima`    | Easy Install                 | Easy Install         |

#### 1. Colima (Containers in Lima)

**"The Hacker's Choice"**

- A CLI tool that runs Docker/Kubernetes in a lightweight Linux VM (Lima).
- **Pros**: Free, Open Source, flexible (specify cpu/ram/disk easily via CLI), supports both Docker and Containerd.
- **Cons**: CLI-only (no GUI dashboard), requires Homebrew.
- **Usage**:

  ```bash
  # Start with Docker runtime
  colima start

  # Start with specific resources (4CPU, 8GB RAM)
  colima start --cpu 4 --memory 8

  # Start with Kubernetes enabled
  colima start --kubernetes
  ```

#### 2. OrbStack

**"The Native Speedster"**

- A newer, native macOS application designed to replace Docker Desktop.
- **Pros**: **Incredible performance** (fast network/disk), minimal battery usage (doesn't keep a heavy VM running), instant startup.
- **Cons**: Paid for commercial use (similar to Docker Desktop).
- **Why switch?** If `kind create cluster` feels slow or your Mac gets hot, switch to this.

#### 3. Rancher Desktop

**"The Enterprise Open Source"**

- A full GUI replacement for Docker Desktop provided by SUSE/Rancher.
- **Pros**: Free & Open Source (even for business), allows selecting specific K8s versions easily from a dropdown, switch between `dockerd` (Moby) and `containerd`.
- **Cons**: Electron app (can be heavier than Colima/OrbStack).

> **💡 Interaction with Cluster Tools:**
>
> - If you use **Colima**, `kind` and `minikube` generally work out of the box because Colima maps the docker socket.
> - **OrbStack** also seamlessly replaces the docker command, so `kind` works instantly.

---

## Cluster Creation

A cluster is the foundation of Kubernetes. It consists of at least one **control plane** (manages the cluster) and one or more **worker nodes** (run your workloads).

### Local Options

| Feature              | **Minikube**                          | **Kind**                             | **K3s**                                | **K0s**                             |
| :------------------- | :------------------------------------ | :----------------------------------- | :------------------------------------- | :---------------------------------- |
| **Primary Use Case** | Learning & Exploration                | CI/CD, Testing, Multi-node           | Edge, IoT, Production Lite             | Zero-friction, Flexible K8s         |
| **Architecture**     | VM (mostly) or Docker                 | Containers (Docker/Podman)           | Single Binary (<60MB)                  | Single Binary                       |
| **Weight**           | 🐘 **Heavy** (Full VM overhead)       | 🐎 **Medium-Light** (Shared kernel)  | 🪶 **Ultralight** (Striped binaries)   | 🪶 **Ultralight**                   |
| **Battery Impact**   | 🔴 **High** (Constant VM load)        | 🟡 **Medium** (Docker dependent)     | 🟢 **Low** (Efficient compiled binary) | 🟢 **Low** (Minimal background ops) |
| **Setup Hassle**     | ⚠️ **Medium** (Driver/Network issues) | ✅ **Low** (Requires Docker running) | ⚠️ **Medium** (If modifying defaults)  | ✅ **Low** (Dependencies handled)   |
| **Stability**        | ⭐⭐⭐⭐ Mature                       | ⭐⭐⭐⭐⭐ Rock solid (K8s Std)      | ⭐⭐⭐⭐⭐ Prod Grade                  | ⭐⭐⭐⭐ Growing fast               |
| **Cilium Support**   | ✅ Built-in flag `--cni=cilium`       | ✅ Easy (disable default CNI)        | ⚠️ Doable (must disable Flannel)       | ✅ Easy (Custom CNI support)        |

### Detailed Breakdown (ArgoCD, Cilium, Helm, Battery)

- **Minikube ("The Teacher")**:
  - **Pros**: Excellent compatibility. Has a specific flag `minikube start --cni=cilium` making it the easiest to start with Cilium.
  - **Cons**: **Heaviest & Power Hungry**. Runs a virtual machine (VM) that keeps CPU active even when idle, causing significant battery drain on laptops. Networking (accessing ArgoCD UI) often requires `minikube tunnel` which can be flaky on Mac.

- **Kind ("The CI Standard")**:
  - **Pros**: **Best balance**. Lighter than Minikube. The standard for testing K8s itself. Excellent for ArgoCD/Helm. Easy to swap CNI for Cilium by disabling the default `kindnet`.
  - **Cons**: **Moderate Power Drain**. Uses Docker containers. While efficient, Docker Desktop on macOS can be resource-intensive. Requires mapping ports to localhost manually in config if you want to access Ingress directly.

- **K3s ("The IoT Powerhouse")**:
  - **Pros**: **Ultralight**. The binary itself is highly optimized for low-power devices (ARM/IoT), making it the most battery-friendly software architecture.
  - **Cons**: **Opinionated**. Ships with Flannel (CNI) and Traefik (Ingress) by default. Replacing them for custom labs (Cilium/Nginx) requires extra configuration flags, which increases setup hassle.

- **K0s ("The Operations Distro")**:
  - **Pros**: Very clean and lightweight. Optimized for efficiency like K3s with no extra "batteries" to disable.
  - **Cons**: Separation of Controller and Worker commands makes the initial local lab setup slightly more manual than `kind create`.

> **🔋 Battery Tip for Mac Users:**
> For the absolute best battery life with **Kind** or **K3s**, consider using **OrbStack** instead of Docker Desktop as your container runtime. It is significantly more power-efficient.

> **🏆 Recommendation for ArgoCD + Cilium:**
> **Kind** is generally preferred for this stack. It simulates multi-node networking well (crucial for Cilium features) without the heavy overhead of VMs, and cleanly supports custom CNIs.

> **💡 Note:** All major local tools support multi-node clusters. This is useful for testing scenarios like node failures, pod scheduling across nodes, and affinity/anti-affinity rules.

### Setting up a Local Cluster

While tools like **Minikube**, **Kind**, and **K0s** are excellent options, **K3s (via K3d)** is often the preferred choice for its speed, low resource usage, and "batteries-included" approach.

#### 🚀 Recommended: K3s (via K3d)

Since K3s is Linux-native, we use **K3d** to run it as docker containers on macOS.

**Prerequisite:** Ensure Docker (via Docker Desktop, OrbStack, or Colima) is running.

```bash
# 1. Install k3d
brew install k3d

# 2. Create a simple single-node cluster
k3d cluster create my-cluster

# 3. Create a multi-node cluster (1 Server + 3 Agents)
# This simulates a real production environment with multiple workers.
k3d cluster create my-cluster --agents 3

# 4. Create a cluster for Cilium/Nginx labs (Disable default Traefik)
# We disable the built-in ingress (Traefik) so we can install our own later.
k3d cluster create my-cluster --k3s-arg "--disable=traefik@server:0"

# 5. Delete cluster
k3d cluster delete my-cluster
```

#### Other Alternatives

- **Minikube**: Good for beginners, uses a VM. (`minikube start`)
- **Kind**: "Kubernetes in Docker", very similar to K3d but uses standard K8s images. (`kind create cluster`)
- **K0s**: Zero-friction distribution, useful for bare-metal limits.

### Cloud Options (Managed Kubernetes)

| Provider                | Service |
| ----------------------- | ------- |
| **Google Cloud**        | GKE     |
| **Amazon Web Services** | EKS     |
| **Microsoft Azure**     | AKS     |

Managed services handle the control plane for you, so you only need to configure and scale worker nodes.

---

## Node Creation

A **Node** is a worker machine in a Kubernetes cluster. Each node runs:

- **kubelet** — An agent that ensures containers in Pods are running.
- **kube-proxy** — Handles network routing for Services.
- **Container runtime** — e.g., containerd or CRI-O, responsible for running containers.

Nodes can be added to a cluster by:

1. **Locally** — Ideally by defining them at creation time (e.g., `k3d cluster create --agents 3`).
2. **Cloud** — Configuring node pools/groups in your managed Kubernetes service.

---

## Pod Creation

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more tightly coupled containers that share networking and storage.

- Pods are typically **not created directly** — they are managed by higher-level resources like Deployments.
- Each Pod gets its own **IP address** within the cluster.

---

## Service Creation

A **Service** provides a stable endpoint to access a group of Pods, even as individual Pods are created or destroyed.

Common Service types:

| Type             | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| **ClusterIP**    | Exposes the Service internally within the cluster (default). |
| **NodePort**     | Exposes the Service on a static port on each node's IP.      |
| **LoadBalancer** | Provisions an external load balancer (cloud environments).   |

---

## Deployment Creation

A **Deployment** manages a set of identical Pods (replicas) and provides:

- **Rolling updates** — Gradually replaces old Pods with new ones.
- **Rollbacks** — Reverts to a previous version if something goes wrong.
- **Scaling** — Adjusts the number of replicas up or down.

---

## Ingress Creation

An **Ingress** manages external access to Services, typically via HTTP/HTTPS. It requires an **Ingress Controller** to function (e.g., NGINX, Cilium, Traefik).

Key features:

- **Path-based routing** — Route `/api` to one Service and `/web` to another.
- **Host-based routing** — Route `api.example.com` and `web.example.com` to different Services.
- **TLS termination** — Handle HTTPS certificates at the Ingress level.
