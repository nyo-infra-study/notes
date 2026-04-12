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

---

## 5. Building Dashboards
**Purpose:** Creating permanent, visual dashboards out of the queries you constructed in the Explore tab.

While the *Explore* tab is for ad-hoc debugging, **Dashboards** are exactly what you put on a TV screen in your engineering bay.

### Step-by-Step Dashboard Creation:
1. On the left sidebar menu, click the **Plus (+)** icon and select **New Dashboard**.
2. Click **Add visualization**.
3. **Select your Data Source** (e.g., Loki for log metrics, Mimir for numerical metrics).
4. **Write your Query**:
   * *Example (Loki)*: To create a graph of error rates over time, select Loki and type: 
     `sum(rate({app="backend-server"} |= "error" [5m]))`
   * *Example (Mimir)*: To chart CPU usage, select Mimir and type: 
     `rate(container_cpu_usage_seconds_total{app="backend-server"}[1m])`
5. **Customize the Panel**: On the right-hand panel, you can change the visualization (Time series, Bar chart, Stat box, Gauge), rename the axis, and give the panel a Title.
6. Click **Apply** in the top-right corner.
7. Click the **Save** icon at the top of the dashboard, give it a name like "Backend Golden Signals", and you're done!

**Pro Tip:** You do not have to build dashboards from scratch! You can go to [Grafana's Community Dashboards](https://grafana.com/grafana/dashboards/) to find thousands of pre-built JSON dashboards (like kubernetes cluster monitoring or Go application monitoring) and import them directly into your UI by clicking `+ -> Import` and pasting the dashboard ID.
