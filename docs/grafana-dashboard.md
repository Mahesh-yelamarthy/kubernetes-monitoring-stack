# Grafana Dashboard

This document describes the first Grafana dashboard added to the monitoring stack.

## Dashboard File

```text
grafana/dashboards/kubernetes-overview.json
```

The dashboard is designed for a Prometheus data source and provides an initial Kubernetes operations view for on-call triage.

## Operational Questions

The dashboard helps answer:

- Are Prometheus scrape targets failing?
- Are any Kubernetes nodes NotReady?
- Are pods restarting or stuck Pending?
- Are deployments below desired replica availability?
- Are node CPU or memory resources saturated?
- Which pods are consuming the most CPU?

## Metric Dependencies

The dashboard expects Prometheus to scrape:

- kube-state-metrics for namespace, node, pod, and deployment state
- node-exporter for node CPU and memory metrics
- cAdvisor or kubelet metrics for container CPU usage
- Prometheus target health through the built-in `up` metric

If a panel is empty, confirm the required metric exists in Prometheus before changing the dashboard query.

## Import Procedure

1. Open Grafana.
2. Go to **Dashboards**.
3. Select **New** and then **Import**.
4. Upload or paste `grafana/dashboards/kubernetes-overview.json`.
5. Select the Prometheus data source.
6. Save the dashboard in the appropriate folder.

## Suggested Folder

Use a folder such as:

```text
Kubernetes / Platform
```

Keeping platform dashboards separate from application dashboards makes incident navigation faster.

## Panel Guidance

| Panel | Use During Incident Response |
| --- | --- |
| Down Scrape Targets | Confirms whether monitoring data may be missing. |
| Nodes Not Ready | Identifies node-level availability problems. |
| Pod Restarts - 15m | Shows recent workload instability. |
| Pending Pods | Highlights scheduling, image pull, or capacity problems. |
| Node CPU Usage | Shows node saturation trends. |
| Node Memory Usage | Shows memory pressure trends. |
| Deployments Below Desired Availability | Identifies workloads running below desired replica count. |
| Top Pods by CPU Usage | Helps locate high CPU workload contributors. |

## Review Checklist

Before using this dashboard as an operational standard:

- Confirm all panels return data in the target cluster.
- Confirm variable filtering works for namespaces.
- Compare dashboard values with direct Prometheus queries.
- Confirm alert rules and dashboard panels use compatible metric names.
- Review the dashboard after the first incident where it is used.

## Future Dashboard Work

Future dashboards should add:

- Namespace resource usage
- Workload restart and readiness detail
- Persistent volume capacity and pressure
- API server health
- Alert overview panels
- Links from panels to related runbooks
