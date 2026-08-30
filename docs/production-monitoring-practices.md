# Production Monitoring Practices

This guide defines production monitoring standards for the Kubernetes monitoring stack. It is intended for SREs, platform engineers, and service owners who need monitoring that supports real incident response instead of passive metric collection.

## Purpose

Production monitoring should help responders answer three questions quickly:

1. Is there user or service impact?
2. What is the blast radius?
3. What action should happen next?

Prometheus, Grafana, Alertmanager, and runbooks should be managed as one operational system. Metrics without dashboards are hard to investigate. Alerts without runbooks create delay. Dashboards without ownership drift out of date.

## Monitoring Objectives

The monitoring stack should:

- Detect production-impacting symptoms before customers or dependent teams report them.
- Reduce mean time to acknowledge by routing actionable alerts to the right responder.
- Reduce mean time to understand by linking alerts, dashboards, and runbooks.
- Preserve useful evidence during incidents.
- Keep alert volume low enough that responders trust the signal.
- Make monitoring changes reviewable through version control.

## Signal Strategy

Use symptom-based monitoring for paging and cause-based monitoring for diagnosis.

| Signal Type | Primary Use | Examples |
| --- | --- | --- |
| User or service symptoms | Paging and incident declaration | Availability loss, high error rate, sustained latency, unavailable replicas. |
| Kubernetes platform symptoms | Platform paging or urgent triage | Node NotReady, pods stuck pending, filesystem pressure, monitoring target loss. |
| Cause indicators | Dashboard diagnosis and warning alerts | CPU saturation, memory pressure, rollout changes, container restarts. |
| Informational signals | Async review and trend analysis | Capacity growth, expected maintenance, noncritical target changes. |

Critical alerts should represent urgent operational risk. Warning alerts should represent actionable degradation that can be investigated during the current support window. Informational alerts should not interrupt an on-call responder.

## Alerting Standards

Every warning or critical alert should include:

- `severity` label.
- `team` or clear owner label.
- `cluster` and `environment` context.
- `namespace`, `workload`, `node`, or `component` context when applicable.
- Human-readable `summary`.
- Operational `description` that explains impact.
- `runbook_url` annotation.
- A `for` duration long enough to avoid paging on transient noise.

Avoid these alert patterns:

- Paging on a single failed scrape when redundant targets exist.
- Thresholds without operational justification.
- Alerts that describe a metric but not a risk.
- Alerts with no owner or no runbook.
- Dashboard curiosity promoted into an interrupting page.
- Duplicate alerts that fire for the same symptom at several abstraction layers.

## Severity Guidance

| Severity | Use When | Expected Response |
| --- | --- | --- |
| Critical | Availability, data safety, monitoring coverage, or cluster health is at immediate risk. | Interrupt on-call and begin incident workflow. |
| Warning | A responder should investigate soon, but immediate customer impact is unconfirmed or limited. | Notify the owning team and investigate in the current support window. |
| Info | The event is useful for awareness but does not require active response. | Review asynchronously or use for routing tests. |

Severity should be based on responder urgency, not metric importance.

## Dashboard Standards

Production dashboards should be designed for on-call use under pressure.

A dashboard is production-ready when it:

- Starts with health, impact, and blast-radius panels.
- Separates cluster, node, namespace, workload, and monitoring-system views.
- Uses recording rules for repeated or expensive PromQL.
- Shows rates, saturation, and availability in consistent time windows.
- Uses clear panel titles that describe the operational question.
- Avoids decorative panels and low-signal visual noise.
- Has an owner and review cadence.

The Kubernetes overview dashboard should remain a first triage surface, not a dumping ground for every metric. Add focused dashboards when a specific workflow becomes too detailed for the overview.

## Runbook Standards

Every runbook linked from an alert should include:

- Impact statement.
- First five minutes of commands or checks.
- Triage questions.
- Safe mitigation options.
- Escalation criteria.
- Validation steps after recovery.
- Follow-up items that improve detection, prevention, or response.

Runbooks should be updated after incidents when responders had to discover missing commands, missing ownership, unclear thresholds, or unsafe assumptions.

## Alertmanager Practices

Alertmanager should reduce cognitive load during incidents.

Recommended practices:

- Group alerts by `cluster`, `namespace`, `severity`, and `alertname`.
- Inhibit warning alerts when a matching critical alert is active for the same scope.
- Keep silences scoped, time-bound, and documented.
- Route critical alerts to actively monitored on-call destinations.
- Route warnings to owning teams or work queues.
- Test the Watchdog or heartbeat route after receiver changes.
- Never commit real webhook tokens, pager addresses, or personal contact details.

## Monitoring Change Review

Monitoring changes should be reviewed like production infrastructure changes.

Before merging a monitoring change, confirm:

- The change has a clear operational purpose.
- Alert thresholds and durations are justified.
- Critical and warning alerts have runbook links.
- New labels support routing and ownership.
- Prometheus rules pass `promtool check rules`.
- Dashboard JSON is valid.
- Alertmanager changes do not expose secrets.
- The change can be rolled back safely.

## Metrics Lifecycle

Treat metrics, rules, dashboards, and alerts as a lifecycle.

When adding a new signal:

1. Confirm the metric source is reliable.
2. Add or update a scrape job if needed.
3. Add a recording rule for repeated or complex queries.
4. Add dashboard coverage for investigation.
5. Add alerting only when a responder action is clear.
6. Add or update a runbook.
7. Review the signal after the first real incident or noisy period.

When removing a signal:

1. Search dashboards, rules, and runbooks for references.
2. Migrate consumers first.
3. Remove unused panels or alerts.
4. Record the reason in the pull request.

## Review Cadence

| Frequency | Review |
| --- | --- |
| Weekly | Noisy alerts, missed alerts, silences, and critical dashboard load health. |
| Monthly | Alert thresholds, runbook links, recording rule usefulness, and routing accuracy. |
| Quarterly | Monitoring coverage against current services, cluster architecture, and incident trends. |
| After incidents | Detection timing, alert quality, dashboard usefulness, runbook completeness, and escalation path. |

## Production Readiness Checklist

The monitoring practice is production-ready when:

- Critical alerts route to tested on-call receivers.
- Every critical alert has a current runbook.
- Dashboards answer impact and blast-radius questions quickly.
- Alert noise is reviewed and reduced regularly.
- Monitoring changes are validated before merge.
- Incident follow-up updates alerts, dashboards, runbooks, or routing when needed.
- Ownership is clear for Prometheus, Grafana, Alertmanager, and runbooks.

