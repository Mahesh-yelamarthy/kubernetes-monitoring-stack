# Kubernetes Monitoring Incident Response

This document defines a production-oriented incident response workflow for alerts and symptoms surfaced by the Kubernetes monitoring stack. It connects Prometheus alerts, Grafana dashboards, Alertmanager routing, architecture diagrams, and runbooks into a repeatable response process.

## Scope

Use this guide when:

- A Prometheus alert fires.
- A Grafana dashboard shows degraded Kubernetes health.
- Alertmanager routing or notification delivery appears broken.
- A responder needs to classify impact and decide whether to escalate.
- A recurring alert needs post-incident follow-up.

This guide does not replace service-specific runbooks. It provides the platform response wrapper around those runbooks.

## Incident Objectives

During an incident, the responder should:

1. Confirm whether there is user or service impact.
2. Identify the affected cluster, namespace, workload, node, or monitoring component.
3. Preserve evidence before making disruptive changes.
4. Use dashboards and Prometheus queries to confirm the failure mode.
5. Follow the relevant runbook.
6. Mitigate safely or escalate with clear context.
7. Record follow-up work that reduces recurrence or alert noise.

## Severity Model

| Severity | Definition | Response |
| --- | --- | --- |
| Critical | User-facing impact, reduced workload availability, unsafe node state, or monitoring failure that hides critical signals. | Page or interrupt the on-call responder. |
| Warning | Actionable degradation with limited or unclear impact. | Notify the owning team and investigate during the current support window. |
| Info | Low-risk visibility or expected operational state. | Review asynchronously. |
| Watchdog | Synthetic alert used to verify alert delivery path. | Investigate notification pipeline if missing or misrouted. |

Severity should reflect responder urgency, not technical curiosity.

## First Five Minutes

Capture context before changing anything:

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
kubectl get events --all-namespaces --sort-by=.lastTimestamp
kubectl -n monitoring get pods -o wide
```

Check whether Prometheus is healthy:

```bash
kubectl -n monitoring get svc,endpoints
kubectl -n monitoring logs deploy/prometheus --tail=100
```

Open the Kubernetes overview dashboard:

```text
grafana/dashboards/kubernetes-overview.json
```

Review architecture and alert flow if ownership or dependency boundaries are unclear:

```text
docs/diagrams/monitoring-architecture.md
```

## Alert Response Flow

```text
Alert fires
    |
    v
Confirm alert labels: cluster, namespace, severity, alertname
    |
    v
Check dashboard and related alerts
    |
    v
Classify impact: user-facing, workload-only, node-only, monitoring-only
    |
    v
Follow the matching runbook
    |
    +--> Mitigate if action is clear and safe
    |
    +--> Escalate if blast radius, ownership, or safety is unclear
    |
    v
Record evidence and follow-up
```

## Response Matrix

| Alert or Symptom | Primary Runbook | Initial Evidence |
| --- | --- | --- |
| `PrometheusTargetDown` | `runbooks/kubernetes-alerts.md` | Prometheus targets, service endpoints, Prometheus logs. |
| `KubernetesNodeNotReady` | `runbooks/kubernetes-alerts.md` | Node conditions, pods on node, cluster events. |
| `KubernetesPodCrashLooping` | `runbooks/kubernetes-alerts.md` | Pod describe output, current and previous logs, rollout history. |
| `KubernetesPodPending` | `runbooks/kubernetes-alerts.md` | Pod events, node capacity, scheduling constraints, PVC state. |
| `KubernetesDeploymentReplicasUnavailable` | `runbooks/kubernetes-alerts.md` | Deployment describe output, ReplicaSets, pods, rollout status. |
| `KubernetesNodeHighCPUUsage` | `runbooks/kubernetes-alerts.md` | Node CPU, top pods, recent deployments or batch jobs. |
| `KubernetesNodeHighMemoryUsage` | `runbooks/kubernetes-alerts.md` | Node memory, top pods, OOM or eviction events. |
| `KubernetesNodeFilesystemSpaceLow` | `runbooks/kubernetes-alerts.md` | Node filesystem, disk pressure condition, eviction events. |
| Missing notifications | `docs/alertmanager-routing.md` | Alertmanager routes, receiver status, grouping and inhibition state. |
| Dashboard has no data | `docs/grafana-dashboard.md` | Prometheus target health, recording rule availability, datasource query. |

## Evidence Capture

Record:

- Alert name and severity.
- Cluster and environment.
- Namespace, workload, pod, or node.
- Start time and detection method.
- Dashboard panels or Prometheus queries reviewed.
- Commands executed and exit codes.
- Mitigation taken.
- Escalation target.
- Follow-up owner.

Example incident note:

```text
Incident: KubernetesDeploymentReplicasUnavailable
Severity: critical
Cluster: dev-cluster
Namespace: payments
Workload: checkout-api
Detected: Prometheus alert via Alertmanager
Evidence: desired=4 available=1, new pods failing readiness, rollout started 10 minutes earlier
Action: Paused rollout and escalated to service owner
Follow-up: Add deployment rollback procedure to service runbook
```

## Escalation Criteria

Escalate immediately when:

- Customer impact is confirmed or likely.
- A critical workload is below safe replica count.
- Multiple nodes are NotReady or under pressure.
- The monitoring stack cannot observe critical services.
- The responder cannot identify ownership.
- Mitigation may delete data, evict critical pods, or drain nodes.
- The alert repeats after mitigation.

Escalation should include evidence, current impact, attempted actions, and the next decision needed.

## Mitigation Guidelines

Use the least disruptive action that restores safety.

Potential mitigations:

- Roll back a bad deployment.
- Scale a workload only when the service owner confirms it is safe.
- Cordon a bad node to prevent new scheduling.
- Drain a node only after checking disruption budgets and workload redundancy.
- Restore a missing metrics target or endpoint selector.
- Silence an alert only when the silence is scoped, time-bound, and documented.

Avoid:

- Deleting pods repeatedly without understanding why they fail.
- Increasing thresholds to hide noisy alerts.
- Draining nodes without checking impact.
- Silencing critical alerts without owner, reason, and expiration.
- Treating dashboard gaps as harmless when they hide monitoring coverage.

## Alertmanager Use During Incidents

Alertmanager should reduce noise, not hide risk.

During incidents:

- Check grouping to identify related alerts.
- Review inhibition to ensure warning alerts are suppressed only when appropriate.
- Create silences only for known, scoped, temporary noise.
- Include owner, reason, and expiration on every silence.
- Confirm resolved notifications are delivered for critical routes.

Routing details are documented in:

```text
docs/alertmanager-routing.md
```

## Post-Incident Review

After recovery:

1. Confirm alerts have resolved.
2. Confirm dashboards show healthy current state.
3. Confirm no related warning alerts remain active.
4. Record timeline, impact, root cause, and mitigation.
5. Identify whether detection was early, late, noisy, or missing.
6. Update alert thresholds, dashboards, runbooks, or routing as needed.
7. Create follow-up work with an owner.

## Follow-Up Categories

| Category | Example |
| --- | --- |
| Alert quality | Add missing runbook URL or tune threshold based on real signal. |
| Dashboard quality | Add panel for missing saturation or workload state. |
| Runbook quality | Add command that responders had to discover manually. |
| Routing quality | Fix receiver, grouping, or severity mapping. |
| Monitoring coverage | Add exporter, scrape job, or recording rule. |
| Reliability work | Fix workload probes, resources, scheduling, or release process. |

## Readiness Checklist

The monitoring response process is production-ready when:

- Critical alerts route to a tested receiver.
- Warning alerts reach an owning team.
- Every critical alert has a runbook link.
- The Kubernetes overview dashboard loads with real data.
- Alertmanager grouping and inhibition are understood by responders.
- Responders know where to record incident notes.
- Post-incident follow-up is tracked and reviewed.
