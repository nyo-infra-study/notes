# Grafana Observability Exercises

This document serves as a cheat sheet for how to navigate and use the Grafana hub to interact with your decentralized observability stack (Loki, Mimir, Tempo, and Pyroscope).

## Core Concept
Grafana acts as our single pane of glass. While the data lives in specialized, separate backends, Grafana brings them all together in the **Explore** tab (the compass icon on the left sidebar).

From the Explore tab, you pick your "Data Source" via the dropdown at the top-left to query different observability signals.

---

## 1. Logs (Loki)
**Purpose:** View real-time cluster and application logs.

1. Go to the **Explore** tab and select **Loki** as the data source.
2. Click the **Label Browser**. 
3. Click the `app` label to see a list of your microservices (e.g., `web-frontend`, `backend-server`).
4. Select `backend-server` and click **Show Logs**.
5. You can refine your search by adding search strings or pipe filters:
   ```logql
   {app="backend-server"} |= "error"
   ```

## 2. Profiles (Pyroscope)
**Purpose:** Understand which Go functions or memory allocations are taking up the most resources in real-time.

1. Go to the **Explore** tab and select **Pyroscope** as the data source.
2. In the "Select application" dropdown, pick `backend-server`.
3. Choose the profile type you wish to see (e.g., `cpu`, `alloc_objects`, `inuse_space`).
4. **The Flame Graph**: Read it from top down. The broader the bar horizontally, the more resources that specific function call is consuming. Click a bar to drill down.

## 3. Distributed Traces (Tempo)
**Purpose:** Follow a specific API request horizontally across multiple microservices to find performance bottlenecks.

1. Go to the **Explore** tab and select **Tempo** as the data source.
2. Querying Traces:
   * **Trace ID**: If you saw a specific `trace_id` in your Loki logs or application output, you can paste it directly here.
   * **Search**: You can utilize the "Search" tab to natively query for spans that took longer than `X` milliseconds.
3. This generates a **Waterfall Diagram**, showing exactly how long an overarching request spent in `web-frontend` vs `backend-server` vs `database`.
*(Note: To utilize this, your applications must be instrumented with the OpenTelemetry SDK to emit spans).*

## 4. Metrics (Mimir)
**Purpose:** Aggregated numerical data over time (e.g. CPU overall usage, memory spikes, HTTP request counts).

1. Go to the **Explore** tab and select **Mimir** as the data source.
2. It uses **PromQL**.
3. **Common basic queries to try:**
   * `up`: Shows 1 or 0 for which scrape targets are successfully reporting in.
   * `container_cpu_usage_seconds_total`: See container-level CPU consumption.
   * `go_memstats_alloc_bytes{job="backend-server"}`: Track Go memory consumption (if scraped).
*(Note: To populate Mimir with rich application metrics, deploy an OpenTelemetry Collector or Prometheus Agent to scrape your pods and "remote-write" the metrics here).*
