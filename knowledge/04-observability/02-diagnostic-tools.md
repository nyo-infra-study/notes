# Diagnostic Tools

[Knowledge Base](../README.md) > [Observability](./README.md) > Diagnostic Tools

A practical guide to diagnosing issues in a running Kubernetes cluster. This covers `kubectl` debugging commands, pod log inspection, and resource monitoring — the tools you reach for when something is broken and you need to figure out why.

> For OS-level diagnostics (lsof, vmstat), see [System Monitoring](../01-foundations/02-system-monitoring.md).

---

## kubectl Debugging Commands

### Inspecting Resources

```bash
# Describe a pod — shows events, conditions, container statuses
kubectl describe pod <pod-name> -n <namespace>

# Describe a node — shows allocatable resources, conditions, running pods
kubectl describe node <node-name>

# Get all resources in a namespace with their status
kubectl get all -n <namespace>

# Show resource requests and limits for pods
kubectl get pods -n <namespace> -o custom-columns=\
'NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory'
```

### Checking Pod Status

```bash
# List pods with extra info (node, IP, restart count)
kubectl get pods -n <namespace> -o wide

# Watch pod status in real time
kubectl get pods -n <namespace> -w

# Show pods sorted by restart count (find crash-looping pods)
kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'

# Get pods in a bad state
kubectl get pods -n <namespace> --field-selector=status.phase!=Running
```

### Exec Into a Container

```bash
# Open a shell in a running pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run a one-off command
kubectl exec <pod-name> -n <namespace> -- env

# Exec into a specific container in a multi-container pod
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/bash
```

### Debugging with Ephemeral Containers

When a pod's image doesn't have a shell, use an ephemeral debug container:

```bash
# Attach a debug container to a running pod
kubectl debug -it <pod-name> -n <namespace> --image=busybox --target=<container-name>

# Create a copy of a pod with a debug container
kubectl debug <pod-name> -n <namespace> -it --copy-to=debug-pod --image=ubuntu
```

---

## Pod Log Inspection

### Basic Log Commands

```bash
# Tail logs for a pod
kubectl logs <pod-name> -n <namespace>

# Follow logs in real time
kubectl logs -f <pod-name> -n <namespace>

# Show last N lines
kubectl logs --tail=100 <pod-name> -n <namespace>

# Logs from a specific container in a multi-container pod
kubectl logs <pod-name> -n <namespace> -c <container-name>

# Logs from a previous (crashed) container instance
kubectl logs <pod-name> -n <namespace> --previous
```

### Aggregating Logs Across Pods

```bash
# Logs from all pods matching a label selector
kubectl logs -n <namespace> -l app=<app-name> --all-containers

# Follow logs from all pods in a deployment
kubectl logs -n <namespace> -l app=<app-name> -f
```

> For persistent, searchable, cross-service log aggregation, use the [PLG Stack](./01-plg-stack.md). `kubectl logs` is best for real-time debugging of a single pod; Loki is better for historical analysis and multi-service traces.

---

## Resource Monitoring in Kubernetes

### Cluster-Level Resource Usage

```bash
# Node CPU and memory usage (requires metrics-server)
kubectl top nodes

# Pod CPU and memory usage
kubectl top pods -n <namespace>

# Top pods across all namespaces, sorted by CPU
kubectl top pods -A --sort-by=cpu

# Top pods sorted by memory
kubectl top pods -n <namespace> --sort-by=memory
```

### Checking Resource Pressure

```bash
# Node conditions (MemoryPressure, DiskPressure, PIDPressure)
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason'

# Events in a namespace (sorted by time — great for spotting OOMKills, evictions)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Watch events in real time
kubectl get events -n <namespace> -w
```

### Investigating OOMKills and Evictions

```bash
# Check if a container was OOMKilled
kubectl describe pod <pod-name> -n <namespace> | grep -A5 "Last State"

# Find recently evicted pods
kubectl get pods -n <namespace> --field-selector=status.phase=Failed \
  -o jsonpath='{range .items[?(@.status.reason=="Evicted")]}{.metadata.name}{"\n"}{end}'
```

---

## Common Diagnostic Workflows

### Pod Won't Start

1. `kubectl get pods -n <namespace>` — check the STATUS column (Pending, CrashLoopBackOff, ImagePullBackOff)
2. `kubectl describe pod <pod-name> -n <namespace>` — read the Events section at the bottom
3. `kubectl logs <pod-name> -n <namespace> --previous` — check logs from the last crash

### Service Not Reachable

```bash
# Verify the service exists and has endpoints
kubectl get svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Test connectivity from inside the cluster
kubectl run tmp-shell --rm -it --image=busybox -n <namespace> -- wget -qO- http://<service-name>:<port>
```

### High Memory / CPU Usage

1. `kubectl top pods -n <namespace> --sort-by=memory` — find the heavy hitter
2. `kubectl describe pod <pod-name> -n <namespace>` — check resource requests/limits
3. `kubectl get events -n <namespace> --sort-by='.lastTimestamp'` — look for OOMKill events
4. For node-level pressure, see [System Monitoring](../01-foundations/02-system-monitoring.md) (`vmstat`, `lsof`)

---

## Prerequisites

- [Kubernetes](../02-container-orchestration/01-kubernetes.md) — understanding pods, services, deployments
- [System Monitoring](../01-foundations/02-system-monitoring.md) — OS-level tools (lsof, vmstat)

## Next Steps

- [PLG Stack](./01-plg-stack.md) — centralized logging for historical analysis and multi-service debugging
