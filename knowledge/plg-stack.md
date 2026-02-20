# PLG Stack (Promtail, Loki, Grafana)

This document outlines the setup and configuration of the PLG stack for log aggregation and monitoring in the `dev` environment.

## Components

- **Loki**: Log aggregation system (like Prometheus, but for logs).
- **Promtail**: Agent that ships logs from pods to Loki.
- **Grafana**: Visualization dashboard.

## Why Centralized Logging? (Vs ArgoCD/Kubectl)

You might ask: _"Why do we need all this? Can't I just view logs in ArgoCD or use `kubectl logs`?"_

While ArgoCD and `kubectl` are great for **real-time debugging** of a single pod, they fall short for **production monitoring** and **historical analysis**.

| Requirement     | **ArgoCD / Kubectl**                                                                                                                                                         | **Centralized Logging (Loki)**                                                                                                                                            |
| :-------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Persistence** | **None.** When a pod restarts or is deleted, its logs are **lost forever**. If a bug causes a crash loop, you lose the error trace exactly when you need it.                 | **Permanent.** Logs are shipped to separate storage. You can view logs for pods that died last week or were deleted hours ago.                                            |
| **Aggregation** | **Single View.** You can usually only see logs for _one_ pod at a time. Tracing a request across `frontend` -> `backend` -> `db` requires opening 3 different terminal tabs. | **Unified View.** Query `{namespace="dev"}` to see logs from _all_ microservices interleaved by time. Trace a request ID across the entire stack in one view.             |
| **Search**      | **Basic.** Limited to `grep` on the last N lines.                                                                                                                            | **Powerful.** Filter by latency (`duration > 500ms`), error codes (`status=500`), or specific users. Calculate metrics from logs (e.g., "Rate of 500 errors per second"). |
| **Volume**      | **Limited.** Browsers crash if you try to load 100k lines of logs in ArgoCD UI.                                                                                              | **Scalable.** Can handle terabytes of logs. You only fetch what you need for your query.                                                                                  |

**In short**: Use **ArgoCD** to check "is it running?". Use **Loki** to answer "why did it fail at 3 AM last Tuesday?".

## Why PLG Stack? (Comparison & Reasoning)

We chose the PLG stack (Promtail, Loki, Grafana) for this environment. Here's why compared to alternatives:

| Feature            | **PLG (Loki)**                                                                                                         | **ELK (Elasticsearch)**                                                                                        | **SaaS (Datadog/Splunk)**                                                                |
| :----------------- | :--------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **Resource Usage** | **Very Low.** Loki only indexes metadata (labels), not the log content itself. Ideal for small-to-medium clusters.     | **High.** Elasticsearch indexes the full text of every log line, requiring significant RAM and CPU (Java JVM). | **Zero (Local).** Offloaded to vendor, but consumes network bandwidth.                   |
| **Cost**           | **Low.** Cheap storage (S3/GCS/MinIO) and low compute.                                                                 | **High.** Requires expensive storage (SSD) and larger nodes for indexing.                                      | **Very High.** Pricing usually scales with log volume ($/GB), which gets expensive fast. |
| **Query Language** | **LogQL.** Inspired by PromQL. Great for metrics-from-logs but limited full-text search capabilities (grep-style).     | **Lucene/KQL.** Extremely powerful full-text search and analytics.                                             | **Proprietary.** Usually very powerful and user-friendly.                                |
| **Integration**    | **Native Kubernetes.** Seamlessly integrates with Grafana (which we use for metrics). Labels match Prometheus metrics. | Good, but requires separate Kibana dashboard management.                                                       | Excellent, but requires vendor agents.                                                   |
| **Maintenance**    | **Moderate.** Easy to deploy via Helm, but retention/compaction needs tuning.                                          | **High.** Managing ES indices, shards, and JVM heap is complex.                                                | **Zero.** Fully managed service.                                                         |

**Verdict**:
For our Kubernetes-native environment, **Loki** provides the best balance. It allows us to correlate logs with metrics (using the same labels) without the immense resource overhead of running a full Elasticsearch cluster. It is "grep for your logs" but distributed and scalable.

## Installation

The stack is deployed via ArgoCD using the `loki-stack` Helm chart.

- **App Manifest**: `infrastructure/apps/dev/plg-stack.yaml`
- **Helm Values**: `infrastructure/platform/plg-stack/values.yaml`
- **Namespace**: `monitoring`

## Configuration Details

### 1. Authentication

Grafana authentication is configured to use a static Kubernetes Secret (`grafana-auth`) instead of a generated password or plain-text value.

**Setup Command:**

```bash
kubectl create secret generic grafana-auth -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=P5F8lxHPhr58CLlCzFRTpr2iUoxjb2YieWnFBHLY
```

### 2. Ingress

Grafana is exposed via Traefik Ingress at:

- **URL**: `http://localhost:9000/grafana`
- **Ingress Class**: `traefik`

### 3. Loki & Promtail Connectivity

We explicitly configured Promtail to send logs to the correct Loki service address.

- **Values**: `promtail.config.clients[0].url: http://dev-plg-stack-loki:3100/loki/api/v1/push`

### 4. Grafana Datasource

We explicitly added a "Loki" datasource to ensure it points to the correct service URL (`http://dev-plg-stack-loki:3100`), overriding any broken defaults.

## Troubleshooting

### "Failed to load log volume" Error

**Symptom**: Grafana Explore shows a red error box "parse error... unexpected IDENTIFIER" but logs still load below.
**Cause**: Compatibility issue between Grafana v10+ and older Loki versions (v2.6.1) regarding LogQL queries for histograms.
**Fix**: Upgrade Loki image to v2.9.3 or later in `values.yaml`.

```yaml
loki:
  image:
    tag: 2.9.3
```

### "No labels found"

**Cause**: Promtail cannot connect to Loki (DNS resolution failure or wrong URL) OR Grafana is looking at the wrong datasource URL.
**Fix**:

1.  Check Promtail logs: `kubectl logs -n monitoring -l app=promtail`
2.  Verify `lokiAddress` or `clients.url` in Promtail config.
3.  Verify Grafana Datasource URL in `Configuration -> Data Sources`.

### ArgoCD Sync Issues

**Symptom**: `dev-plg-stack` application is always "OutOfSync" due to `volumeClaimTemplates`.
**Fix**: Add an `ignoreDifferences` block to the Application manifest for the Loki StatefulSet.

```yaml
ignoreDifferences:
  - group: apps
    kind: StatefulSet
    name: dev-plg-stack-loki
    jsonPointers:
      - /spec/volumeClaimTemplates
```
