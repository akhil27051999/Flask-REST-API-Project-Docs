# Observability Stack Milestone: Interview Prep Notes

## 1. **Overview**

This milestone is about making a Kubernetes-based system **observable**—meaning you can monitor, log, and analyze all aspects of your applications and infrastructure. You’ll use the **PLG stack** (Promtail, Loki, Grafana) for logs, and **Prometheus** for metrics. You’ll also set up exporters for databases and endpoints, and visualize everything in Grafana.

---

## 2. **Key Tools & Their Roles**

- **Prometheus:**  
  - Pulls metrics from applications, Kubernetes resources (nodes, pods, deployments), databases, and endpoints.
  - Scrapes metrics using ServiceMonitors and custom scrape configs.
  - Stores time-series data; supports alerting.

- **Loki:**  
  - Stores and indexes logs.
  - Receives logs from Promtail.

- **Promtail:**  
  - Collects and ships application logs from nodes/pods to Loki.
  - Configurable to filter specific log sources (e.g., only application logs).

- **Grafana:**  
  - Visualizes metrics (from Prometheus) and logs (from Loki).
  - Dashboards and alerting.

- **Blackbox Exporter:**  
  - Used by Prometheus to probe internal/external endpoints (HTTP, TCP, ICMP).
  - Measures latency, uptime, availability of endpoints like REST APIs, ArgoCD server, Vault.

- **DB Exporter (e.g., Postgres/MySQL Exporter):**  
  - Exposes database metrics (connections, queries, health) for Prometheus to scrape.

- **Kube-State-Metrics:**  
  - Exposes Kubernetes resource metrics (state of deployments, pods, etc.).

---

## 3. **Architecture & Deployment**

- All components are deployed in a dedicated namespace (e.g., `observability`).
- Components are scheduled to a specific node (`dependent_services`) using `nodeSelector`.
- Each component is installed via official Helm charts for declarative, version-controlled deployment.
- Helm values files are customized for:
  - Namespace and node targeting.
  - Service endpoints.
  - Data source configuration (e.g., Grafana points to Prometheus & Loki).
  - Scrape configs for Prometheus (kube-state-metrics, node, DB exporter, blackbox, etc.).
  - Log filtering in Promtail.

---

## 4. **How Monitoring Works**

- **Metrics Collection:**
  - Prometheus Operator manages Prometheus instances and ServiceMonitors.
  - Prometheus scrapes metrics from:
    - Kubernetes cluster (nodes, pods, kube-state-metrics).
    - Application metrics endpoints (usually `/metrics`).
    - DB Exporter.
    - Blackbox Exporter (for endpoints).
- **Log Collection:**
  - Promtail reads log files or pod logs and ships them to Loki.
- **Visualization:**
  - Grafana connects to Prometheus (metrics) and Loki (logs).
  - Dashboards visualize health, latency, usage, errors, etc.
  - Alerts can be set up (e.g., high latency, downtime).

---

## 5. **Why This Matters (Interview Focus)**

- **Observability is key for:**
  - Detecting issues (failures, bottlenecks).
  - Debugging (combining metrics and logs).
  - Ensuring reliability and performance.
  - Capacity planning.
  - Root cause analysis.

- **Why Helm?**
  - Declarative, repeatable installations.
  - Version control for configurations.
  - Easy upgrades and rollbacks.

- **Why separate namespace/node?**
  - Isolation from workloads.
  - Ensures observability stack is not affected by application resource pressure.

---

## 6. **Common Interview Questions**

- **How would you monitor a distributed system?**
  - By collecting metrics (Prometheus), logs (Loki), and visualizing them (Grafana).
  - Use exporters for databases, blackbox for endpoints.
  - Ensure scraping all critical services and infrastructure.

- **How do you ensure only application logs go to Loki?**
  - Configure Promtail’s `scrape_configs` to target only application log files or pods.

- **How do you monitor database health?**
  - Deploy DB Exporter, configure Prometheus to scrape its metrics, visualize/alert in Grafana.

- **How do you monitor endpoint latency and uptime?**
  - Use Blackbox Exporter configured in Prometheus, set up alerts for probe failures/high latency.

- **How do you ensure observability stack is highly available?**
  - Deploy replicas, use persistent storage for metrics/logs, monitor the observability stack itself.

---

## 7. **Verification & Troubleshooting**

- Check running pods/services in the `observability` namespace.
- Verify Prometheus targets (scrape status).
- Confirm Grafana data sources for metrics/logs.
- Query logs in Grafana Explore.
- Validate alerting rules.

---

## 8. **Summary: What You Achieve**

- **End-to-end observability** for Kubernetes workloads and infrastructure.
- **Metrics and logs** for applications, dependencies, databases, and endpoints.
- **Declarative, repeatable deployment** using official Helm charts.
- **Easy extension** for more exporters, dashboards, and alerts.

---

## 9. **Extra Tips**

- Be ready to explain how you’d extend this stack (e.g., tracing with Tempo/Jaeger, alerting integrations).
- Know how to debug problems (permissions, service discovery, scrape configs).
- Understand how each component interacts (Promtail → Loki → Grafana; Prometheus → Grafana).
- Know how to secure the stack (RBAC, TLS).

---
# Monitoring Flow

## 1. **Log Flow (PLG Stack: Promtail → Loki → Grafana)**

1. **Promtail** is deployed as a DaemonSet/Deployment in the `observability` namespace, scheduled to the `dependent_services` node.
2. Promtail is configured to **only collect application logs** (e.g., from `/var/log/app/*.log` or relevant pod/container logs).
3. Promtail pushes these logs to **Loki** via HTTP.
4. **Loki** stores and indexes the logs.
5. **Grafana** connects to Loki as a log data source.
6. Users can query, visualize, and analyze logs in Grafana dashboards (and in Grafana’s Explore tab).

## 2. **Metrics Flow (Prometheus Stack: Exporters → Prometheus → Grafana)**

1. **Prometheus Operator** manages the Prometheus server in the `observability` namespace.
2. **Prometheus** scrapes metrics from various sources:
    - **Kubernetes cluster metrics**:
        - **kube-state-metrics**: exposes resource states (pods, deployments, etc.).
        - **node-exporter**: exposes node-level metrics (CPU, memory, disk).
    - **Application metrics endpoints**: applications expose metrics at `/metrics`.
    - **Database exporter** (e.g., postgres-exporter): exposes DB connection, query stats.
    - **Blackbox exporter**: exposes probe results for internal endpoints (ArgoCD, Vault, REST API), measuring **latency**, **availability**, etc.
3. Prometheus stores these metrics as time-series data.
4. **Grafana** connects to Prometheus as a metrics data source.
5. Users create **dashboards** in Grafana to visualize metrics (e.g., CPU, DB connections, endpoint latency).
6. **Alerts** can be configured in Prometheus or Grafana (e.g., notify on downtime, high latency, resource exhaustion).

## 3. **How Everything Connects**

- **Promtail → Loki → Grafana:**  
  - Promtail ships logs → Loki stores logs → Grafana visualizes logs.
- **Exporters → Prometheus → Grafana:**  
  - Exporters (kube-state-metrics, node-exporter, db-exporter, blackbox-exporter) expose metrics → Prometheus scrapes and stores metrics → Grafana visualizes metrics.

---

## Monitoring Endpoints

- **Blackbox Exporter:**  
  - Prometheus is configured (via scrape configs or ServiceMonitor) to use blackbox-exporter to probe endpoints (HTTP, TCP, etc.).
  - Example: probe ArgoCD server, Vault, REST API for uptime/latency.
  - Results are available in Prometheus and visualized in Grafana.

---

## Deployment Control/Isolation

- All components are placed in a dedicated namespace (`observability`) and scheduled to a specific node (`dependent_services`) for resource isolation.

---

## Verification

- **Grafana dashboards** show both metrics and logs.
- **Prometheus UI** shows scrape status and target health.
- **Grafana Explore** lets you query logs directly from Loki.
- **Alerts** notify you of failures, latency spikes, or abnormal resource usage.

---

## Summary Table

| Data Type | Source            | Ingestion | Storage    | Visualization |
|-----------|-------------------|-----------|------------|---------------|
| Logs      | Application logs  | Promtail  | Loki       | Grafana       |
| Metrics   | Kube/node/app/DB  | Prometheus| Prometheus | Grafana       |
| Endpoints | Blackbox exporter | Prometheus| Prometheus | Grafana       |

---

## Why This Matters

- Enables detection of issues (failures, bottlenecks).
- Supports debugging (combining metrics and logs).
- Ensures reliability and performance.
- Aids in capacity planning and root cause analysis.
- Declarative, repeatable deployment using Helm charts.
- Easy extension for more exporters, dashboards, and alerts.


**Mastering this milestone means you can confidently design, deploy, and operate a fully observable Kubernetes platform.**

# Observability Stack: Installation & Verification

## Prerequisites

- [Helm 3](https://helm.sh/docs/intro/install/)
- `kubectl` configured for your cluster
- The following Helm repositories added:
  ```sh
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo add grafana https://grafana.github.io/helm-charts
  helm repo update
  ```

---

## 1. Create the Namespace

```sh
kubectl create namespace observability
```

---

## 2. Install Prometheus (kube-prometheus-stack)

```sh
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace observability \
  -f helm/prometheus/values.yaml
```

---

## 3. Install Loki

```sh
helm install loki grafana/loki \
  --namespace observability \
  -f helm/loki/values.yaml
```

---

## 4. Install Promtail

```sh
helm install promtail grafana/promtail \
  --namespace observability \
  -f helm/promtail/values.yaml
```

---

## 5. Install Grafana

```sh
helm install grafana grafana/grafana \
  --namespace observability \
  -f helm/grafana/values.yaml
```

---

## 6. Install Blackbox Exporter

```sh
helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  --namespace observability \
  -f helm/blackbox-exporter/values.yaml
```

---

## 7. Install DB Exporter (example: PostgreSQL)

```sh
helm install db-exporter prometheus-community/prometheus-postgres-exporter \
  --namespace observability \
  -f helm/db-exporter/values.yaml
```

---

## 8. Install kube-state-metrics

```sh
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace observability \
  -f helm/kube-state-metrics/values.yaml
```

---

## Verification Steps

### 1. **Check Pods Status**
```sh
kubectl get pods -n observability
```
All pods should be in `Running` or `Completed` state.

---

### 2. **Check Service Endpoints**
```sh
kubectl get svc -n observability
```
- `prometheus-kube-prometheus-prometheus`
- `loki`
- `grafana`
- `blackbox-exporter`
- `db-exporter`
- `kube-state-metrics`
should be listed and have ClusterIP addresses.

---

### 3. **Grafana Access & Data Sources**
- Port-forward Grafana (if not using ingress):
  ```sh
  kubectl port-forward svc/grafana 3000:3000 -n observability
  ```
- Open `http://localhost:3000`
  - Login with credentials (admin/admin or as set in your values).
  - Go to **Configuration > Data Sources**
    - Ensure there are **Prometheus** and **Loki** data sources configured.

---

### 4. **Prometheus Targets**
- Port-forward Prometheus:
  ```sh
  kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n observability
  ```
- Open `http://localhost:9090/targets`
  - Verify that targets for:
    - **kube-state-metrics**
    - **node-exporter**
    - **db-exporter**
    - **blackbox-exporter**
    are present and marked as `UP`.

---

### 5. **Loki Integration**
- In Grafana, go to the **Explore** tab.
- Select **Loki** data source.
- Enter a log query (e.g., `{job="app"}`) to see application logs.

---

### 6. **Blackbox Exporter Verification**
- In Prometheus, check the targets for Blackbox exporter.
- Verify endpoint metrics for ArgoCD, Vault, REST API are scraped.
- You can use a Prometheus query like:
  ```
  probe_success{instance="https://argocd-server.internal"}
  ```

---

### 7. **DB Exporter Metrics**
- In Prometheus, search for metrics like `pg_stat_activity_count` (Postgres) or equivalent for your DB.
- Ensure values are being scraped.

---

### 8. **Kube-State-Metrics**
- In Prometheus, query metrics such as `kube_pod_info` or `kube_deployment_status_replicas`.

---

### 9. **Promtail Log Forwarding**
- Check Promtail logs:
  ```sh
  kubectl logs deploy/promtail -n observability
  ```
- Ensure logs are being pushed to Loki.

---

## Troubleshooting

- Use `kubectl describe pod <pod-name> -n observability` for debug info.
- Check Helm release status:
  ```sh
  helm status <release-name> -n observability
  ```

---

## Cleanup

To remove the observability stack:
```sh
helm uninstall prometheus loki promtail grafana blackbox-exporter db-exporter kube-state-metrics -n observability
kubectl delete namespace observability
```

---

**You are now fully set up with an observable, production-ready Kubernetes monitoring stack!**
