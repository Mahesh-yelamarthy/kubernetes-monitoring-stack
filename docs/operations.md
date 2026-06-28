# Operations Guide

This document defines the operational expectations for the Kubernetes monitoring stack. It is the starting point for how the stack should be maintained, reviewed, and used during production support.

## Operational Objectives

The monitoring stack should support the following SRE outcomes:

- Detect production-impacting issues quickly.
- Reduce time to understand incident scope.
- Provide reliable dashboards for investigation.
- Route alerts to the correct responder.
- Keep alert volume actionable and manageable.
- Maintain clear runbooks for common failure modes.

## Ownership Model

Recommended ownership for a production environment:

| Area | Owner | Responsibility |
| --- | --- | --- |
| Prometheus configuration | Platform or SRE team | Scrape jobs, rule files, retention settings, and target health. |
| Alert rules | SRE team with service owners | Alert thresholds, severity, labels, and runbook links. |
| Grafana dashboards | SRE team and service owners | Dashboard accuracy, usability, and review after incidents. |
| Alertmanager routing | SRE team | Notification routing, grouping, silences, and escalation policy. |
| Runbooks | SRE team and service owners | Triage steps, mitigation options, and incident learnings. |

## Change Management

Monitoring changes should be reviewed with the same care as application and infrastructure changes.

Recommended review checklist:

- Does the change improve detection, diagnosis, or response?
- Is the alert or dashboard tied to a real operational need?
- Could the change increase alert noise?
- Are labels, names, and descriptions clear?
- Does any critical alert have a runbook?
- Are thresholds justified by expected production behavior?
- Can the change be rolled back cleanly?

## Alert Quality Standards

An alert should be considered production-ready when it includes:

- Clear alert name
- Severity label
- Affected service, namespace, or component
- Human-readable summary
- Actionable description
- Runbook reference for warning or critical alerts
- Threshold based on operational risk

Avoid alerts that only say a metric changed without explaining why a responder should care.

## Baseline Alert Operations

The current baseline alert file is:

```text
prometheus/rules/kubernetes-alerts.yml
```

These rules cover early platform health signals: scrape target failures, nodes not ready, repeated pod restarts, pods stuck pending, and deployments below desired replica availability.

Before enabling these alerts in a cluster, confirm kube-state-metrics is deployed and Prometheus is scraping it. Without kube-state-metrics, Kubernetes state expressions such as pod phase, deployment replica status, and node readiness will not evaluate.

Use the Kubernetes alert runbook for response:

```text
runbooks/kubernetes-alerts.md
```

## Dashboard Quality Standards

A dashboard should be considered production-ready when it:

- Answers a specific operational question
- Uses clear panel titles
- Separates cluster, node, workload, and service views
- Shows rates and saturation where appropriate
- Avoids unnecessary visual noise
- Provides enough context for incident triage
- Can be understood by an on-call engineer under pressure

## Baseline Dashboard Operations

The current Grafana dashboard is:

```text
grafana/dashboards/kubernetes-overview.json
```

Use it as the first investigation surface for Kubernetes platform incidents. It is intentionally broad and should help responders decide whether to inspect scrape health, node health, workload restarts, pod scheduling, deployment availability, or resource saturation.

Before relying on it during on-call, confirm the target cluster exposes kube-state-metrics, node-exporter metrics, and container CPU metrics.

## Incident Response Expectations

During an incident, the monitoring stack should help responders:

1. Confirm whether there is user or service impact.
2. Identify the affected cluster, namespace, workload, or node.
3. Compare current behavior against recent baseline behavior.
4. Locate related alerts and dashboard panels.
5. Follow the relevant runbook.
6. Capture findings for post-incident review.

## Routine Maintenance

Recommended recurring maintenance:

| Frequency | Activity |
| --- | --- |
| Weekly | Review firing alerts and noisy alerts. |
| Weekly | Confirm critical dashboards still load and show expected data. |
| Monthly | Review alert thresholds against real incidents and operational trends. |
| Monthly | Validate runbook links and update stale commands. |
| Quarterly | Review monitoring coverage for new services and Kubernetes changes. |

## Baseline Workload Checks

Before monitoring is added, the existing NGINX workload can be verified with basic Kubernetes checks:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
kubectl apply -f manifests/nginx-service.yaml
kubectl get pods
kubectl get svc
kubectl describe deployment nginx-deployment
```

These checks confirm that the workload exists and gives future Prometheus and dashboard work a concrete target.

## Failure Modes to Plan For

The monitoring stack itself can fail. Future documentation should cover:

- Prometheus target scrape failures
- Missing Kubernetes metrics
- Grafana dashboard data gaps
- Alertmanager notification failures
- Excessive alert noise
- Runbook drift
- Monitoring namespace resource pressure

## Day 1 Scope

This Day 1 version defines the operational baseline and documentation structure.

Implementation-specific operations will continue to expand as dashboards, Alertmanager routing, additional rules, and incident response documents are introduced.
