# Prometheus Rules

This directory will contain version-controlled Prometheus recording and alerting rules.

The base Prometheus configuration loads files matching:

```text
/etc/prometheus/rules/*.yml
```

The deployment mechanism must mount this repository directory at `/etc/prometheus/rules` inside the Prometheus container.

## Planned Rule Files

| File | Purpose |
| --- | --- |
| `kubernetes-alerts.yml` | Kubernetes node, pod, and workload availability alerts. |
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

No alert rules are added on Day 4. Kubernetes alert rules are scheduled for a later, separate change.
