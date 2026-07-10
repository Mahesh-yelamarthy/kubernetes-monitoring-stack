# Alertmanager Routing Guide

This guide documents the baseline Alertmanager configuration for the Kubernetes monitoring stack.

The goal is to make alert delivery predictable before real notification integrations are added. The current configuration uses webhook receivers that represent production routing targets without committing secrets, personal contact details, or vendor-specific tokens.

## Configuration File

The Alertmanager configuration is stored at:

```text
alertmanager/alertmanager.yml
```

It defines:

- Default grouping for Kubernetes alerts.
- Separate routing for critical, warning, info, and heartbeat alerts.
- Inhibition rules to reduce duplicate pages.
- Placeholder webhook receivers for future integration with an alert router, incident management system, or chat notification service.

## Routing Model

| Alert type | Matcher | Receiver | Repeat interval | Intended response |
| --- | --- | --- | --- | --- |
| Critical platform alerts | `severity="critical"` | `platform-critical` | 30 minutes | Page the on-call responder and start incident triage. |
| Warning platform alerts | `severity="warning"` | `platform-warning` | 2 hours | Notify the owning team for investigation during working support hours. |
| Informational alerts | `severity="info"` | `platform-default` | 12 hours | Record low-priority operational context without interrupting responders. |
| Watchdog alerts | `alertname="Watchdog"` | `platform-heartbeat` | 5 minutes | Confirm that the alert delivery path is alive. |
| Unmatched alerts | None | `platform-default` | 4 hours | Keep unknown alerts visible while routing gaps are fixed. |

Critical alerts are matched before warning and info alerts. This keeps high-impact symptoms from being delayed by broad default routing.

## Grouping Strategy

Alerts are grouped by:

```text
cluster
namespace
severity
alertname
```

This grouping keeps related symptoms together while still separating different environments, namespaces, severities, and alert classes.

The baseline timing is:

| Setting | Value | Reason |
| --- | --- | --- |
| `group_wait` | 30 seconds | Allows related alerts to arrive before the first notification. |
| `group_interval` | 5 minutes | Avoids excessive notification churn during active incidents. |
| `repeat_interval` | 4 hours | Keeps unresolved unmatched alerts visible without spamming responders. |
| Critical `group_wait` | 15 seconds | Reduces time to page for high-severity issues. |
| Critical `repeat_interval` | 30 minutes | Repeats unresolved pages often enough for production incidents. |

## Inhibition Strategy

The baseline inhibition rule suppresses warning alerts when a critical alert with the same `cluster`, `namespace`, and `alertname` is active.

This prevents duplicate notifications where the critical alert already represents the highest-priority response path. It does not suppress alerts across different namespaces or clusters because those may represent separate incidents.

## Receiver Contract

The current receivers point to an internal alert router service:

```text
http://alert-router.monitoring.svc.cluster.local:8080
```

In a real environment, that service would forward alerts to approved destinations such as PagerDuty, Opsgenie, Slack, Microsoft Teams, email, or an internal incident workflow.

Do not commit real webhook tokens, API keys, or personal contact URLs into this repository. Store secret values in Kubernetes Secrets, an external secret manager, or the deployment system that renders the final Alertmanager configuration.

## Label Requirements

Alert rules should include the following labels when possible:

| Label | Purpose |
| --- | --- |
| `severity` | Drives routing priority. Expected values are `critical`, `warning`, and `info`. |
| `team` | Identifies the owning team for escalation and reporting. |
| `service` | Identifies the affected service or platform component. |
| `cluster` | Separates production, staging, and development clusters. |
| `namespace` | Separates Kubernetes ownership and blast radius. |

Missing labels should be fixed in the alert rule instead of worked around in Alertmanager whenever possible.

## Validation

Validate YAML syntax:

```bash
ruby -e 'require "yaml"; YAML.load_file("alertmanager/alertmanager.yml")'
```

Validate with Alertmanager tooling when available:

```bash
amtool check-config alertmanager/alertmanager.yml
```

Review routing behavior before production rollout:

```bash
amtool config routes --config.file=alertmanager/alertmanager.yml
```

## Deployment Notes

Before enabling this configuration in a cluster:

1. Confirm the alert router service exists in the `monitoring` namespace.
2. Replace placeholder receivers with environment-approved integrations.
3. Confirm critical alerts route to an actively monitored on-call path.
4. Confirm warning alerts do not page outside the intended support model.
5. Test a synthetic Watchdog alert from Prometheus through Alertmanager.
6. Confirm resolved notifications are delivered and understood by responders.

## Operational Review

Alertmanager routing should be reviewed after:

- Any missed page.
- Any noisy alert storm.
- Any incident where ownership was unclear.
- Any new production service or namespace launch.
- Any change to on-call coverage or notification tools.

Routing should stay simple until operational evidence proves that additional nesting or environment-specific routing is needed.
