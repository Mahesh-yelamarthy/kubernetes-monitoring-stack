# Prometheus Rules

This directory contains version-controlled Prometheus recording and alerting rules.

The base Prometheus configuration loads files matching:

```text
/etc/prometheus/rules/*.yml
```

The deployment mechanism must mount this repository directory at `/etc/prometheus/rules` inside the Prometheus container.

## Rule Files

| File | Purpose |
| --- | --- |
| `kubernetes-alerts.yml` | Kubernetes node, pod, deployment, and Prometheus target health alerts. |
| `recording-rules.yml` | Precomputed queries for dashboards and commonly reused expressions. |

## Rule Quality Requirements

Every alerting rule should include:

- A stable alert name
- A duration using `for` to avoid transient noise where appropriate
- A severity label
- Ownership or service context
- A concise summary
- An actionable description
- A runbook URL for page-worthy alerts

## Validation

Validate rule files before deployment:

```bash
promtool check rules prometheus/rules/*.yml
```

Validate the complete Prometheus configuration:

```bash
promtool check config prometheus/prometheus.yml
```

## Current Alerts

The baseline Kubernetes alert file is:

```text
prometheus/rules/kubernetes-alerts.yml
```

It includes alerts for:

- Prometheus scrape target failures
- Kubernetes nodes not ready
- Repeated pod container restarts
- Pods stuck pending
- Deployments with unavailable replicas
- Sustained node CPU saturation
- Sustained node memory saturation
- Low node filesystem space

The Kubernetes state alerts require kube-state-metrics. Resource saturation alerts require node-exporter metrics. The Prometheus target health alert uses the built-in `up` metric.

## Current Recording Rules

The baseline recording rule file is:

```text
prometheus/rules/recording-rules.yml
```

It includes reusable expressions for:

- Cluster and node CPU utilization
- Node memory utilization
- Node filesystem availability
- Namespace pod restart rate
- Namespace pending pod count
- Namespace unavailable deployment replicas
- Down Prometheus targets by job
