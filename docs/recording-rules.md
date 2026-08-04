# Prometheus Recording Rules

This document describes the baseline recording rules for the Kubernetes monitoring stack.

Recording rules precompute frequently used PromQL expressions. They make dashboards faster, reduce repeated query complexity, and create stable query names that alerts and dashboards can share.

## Rule File

```text
prometheus/rules/recording-rules.yml
```

The base Prometheus configuration already loads:

```text
/etc/prometheus/rules/*.yml
```

## Current Rules

| Recording rule | Purpose | Primary dependency |
| --- | --- | --- |
| `cluster:node_cpu_utilization:ratio` | Cluster-wide CPU utilization ratio. | node-exporter |
| `node:node_cpu_utilization:ratio` | Per-node CPU utilization ratio. | node-exporter |
| `node:node_memory_utilization:ratio` | Per-node memory utilization ratio using `MemAvailable`. | node-exporter |
| `node:node_filesystem_available:ratio` | Per-node filesystem availability ratio. | node-exporter |
| `namespace:pod_container_restarts:rate5m` | Namespace-level container restart rate. | kube-state-metrics |
| `namespace:pod_pending:count` | Namespace-level pending pod count. | kube-state-metrics |
| `namespace:deployment_unavailable_replicas:sum` | Namespace-level unavailable deployment replicas. | kube-state-metrics |
| `job:prometheus_targets_down:count` | Down scrape targets grouped by job. | Prometheus `up` metric |

## Naming Convention

The rules use the format:

```text
scope:metric:aggregation
```

Examples:

- `node:node_cpu_utilization:ratio`
- `namespace:pod_pending:count`
- `job:prometheus_targets_down:count`

This convention keeps rule names readable in dashboards, alerts, and incident notes.

## Dashboard Usage

Dashboards should prefer recording rules when the same query is used repeatedly.

Example dashboard query:

```promql
node:node_cpu_utilization:ratio * 100
```

This is easier for responders to read than repeating a full CPU idle-rate expression in every panel.

## Alerting Usage

Alert expressions may continue to use direct metrics when the expression needs full label control. Recording rules should be considered when:

- The query is expensive.
- The expression is reused in dashboards and alerts.
- The alert should be based on a stable operational signal.

Any alert built from a recording rule should still include severity, ownership labels, a clear description, and a runbook URL.

## Validation

Validate rules before deployment:

```bash
promtool check rules prometheus/rules/*.yml
```

Expected result:

```text
SUCCESS
```

## Operational Notes

Recording rules are part of the monitoring platform contract. Renaming a rule can break dashboards and alerts that depend on it.

Before changing a recording rule:

1. Search dashboards and alert rules for references.
2. Add the replacement rule before removing the old rule.
3. Deploy and confirm both rules evaluate.
4. Update dashboards or alerts.
5. Remove the old rule only after consumers are migrated.
