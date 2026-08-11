# Monitoring Architecture Diagrams

This document captures the Kubernetes monitoring stack architecture in reviewable Mermaid diagrams. The goal is to make data flow, alert flow, and responder workflow clear enough for another SRE to operate the stack without reverse-engineering every configuration file.

## System Context

```mermaid
flowchart TD
    user["Users and services"] --> svc["Kubernetes Service"]
    svc --> app["NGINX workload"]

    app --> kubelet["Kubelet and cAdvisor metrics"]
    kube["Kubernetes API server"] --> ksm["kube-state-metrics"]
    node["Kubernetes nodes"] --> nodeExporter["node-exporter"]

    prometheus["Prometheus"] --> apiScrape["API server scrape"]
    prometheus --> kubelet
    prometheus --> ksm
    prometheus --> nodeExporter
    prometheus --> annotatedPods["Annotated workload metrics"]

    prometheus --> rules["Alert and recording rules"]
    rules --> alertmanager["Alertmanager"]
    rules --> grafana["Grafana dashboards"]

    alertmanager --> oncall["On-call notification path"]
    grafana --> responder["SRE responder"]
    oncall --> responder
    responder --> runbooks["Runbooks and operations docs"]
```

### Operational Meaning

- Kubernetes exposes platform and workload state through the API server, kubelet, cAdvisor, kube-state-metrics, and node-exporter.
- Prometheus discovers targets through Kubernetes service discovery and evaluates alerting and recording rules.
- Grafana uses raw metrics and recording rules for incident investigation.
- Alertmanager receives firing alerts, groups related symptoms, inhibits lower-priority duplicates, and routes notifications.
- Runbooks convert alerts into repeatable investigation and mitigation steps.

## Prometheus Data Flow

```mermaid
flowchart LR
    subgraph sources["Metrics sources"]
        api["API server"]
        kubelet["Kubelet"]
        cadvisor["cAdvisor"]
        ksm["kube-state-metrics"]
        nodeExporter["node-exporter"]
        pods["Annotated pods"]
    end

    subgraph prometheus["Prometheus"]
        discovery["Kubernetes service discovery"]
        scrape["Scrape jobs"]
        storage["Time series storage"]
        recording["Recording rules"]
        alerts["Alert rules"]
    end

    api --> discovery
    kubelet --> scrape
    cadvisor --> scrape
    ksm --> scrape
    nodeExporter --> scrape
    pods --> discovery
    discovery --> scrape
    scrape --> storage
    storage --> recording
    storage --> alerts
    recording --> dashboards["Grafana dashboards"]
    alerts --> alertmanager["Alertmanager"]
```

### Implemented Configuration

| Concern | File | Notes |
| --- | --- | --- |
| Scrape configuration | `prometheus/prometheus.yml` | Defines Prometheus, API server, node, cAdvisor, and pod discovery jobs. |
| Alert rules | `prometheus/rules/kubernetes-alerts.yml` | Covers target health, node readiness, pod failures, deployment availability, and node saturation. |
| Recording rules | `prometheus/rules/recording-rules.yml` | Precomputes reusable cluster, node, namespace, and target health signals. |
| Dashboard | `grafana/dashboards/kubernetes-overview.json` | Uses Kubernetes and Prometheus metrics for first-response triage. |
| Alert routing | `alertmanager/alertmanager.yml` | Groups, inhibits, and routes alerts by severity and alert type. |

## Alert Lifecycle

```mermaid
sequenceDiagram
    participant P as Prometheus
    participant R as Rule evaluation
    participant A as Alertmanager
    participant N as Notification receiver
    participant S as SRE responder
    participant B as Runbook

    P->>R: Evaluate alert rules on interval
    R->>R: Apply threshold and for-duration
    R->>A: Send firing alert with labels and annotations
    A->>A: Group, deduplicate, and apply inhibition rules
    A->>N: Route by severity and alert name
    N->>S: Notify responder
    S->>B: Follow linked runbook
    S->>P: Query metrics and inspect related signals
    S->>A: Silence only when justified and time-bound
```

### Alert Routing Expectations

- Critical alerts should reach an actively monitored on-call path.
- Warning alerts should reach the responsible team without unnecessary paging.
- Heartbeat or watchdog alerts should verify the notification path itself.
- A silence should include owner, reason, scope, and expiration.
- Recurring alerts should produce runbook or threshold follow-up work.

## Incident Investigation Workflow

```mermaid
flowchart TD
    alert["Alert fires"] --> classify["Classify symptom and severity"]
    classify --> scope["Identify cluster, namespace, workload, or node"]
    scope --> dashboard["Open Kubernetes overview dashboard"]
    dashboard --> related["Check related alerts and recording-rule signals"]
    related --> runbook["Follow matching runbook"]
    runbook --> impact{"User or service impact?"}
    impact -- "yes" --> mitigate["Mitigate or escalate"]
    impact -- "no" --> monitor["Monitor and gather more evidence"]
    mitigate --> record["Record commands, findings, and action"]
    monitor --> record
    record --> review["Create follow-up for prevention or noise reduction"]
```

### First Response Questions

| Question | Primary Surface |
| --- | --- |
| Is Prometheus missing data? | Target health panels and `PrometheusTargetDown`. |
| Is the node unhealthy? | Node readiness and resource saturation panels. |
| Is the workload unavailable? | Deployment availability, pod status, and restart panels. |
| Is this capacity related? | CPU, memory, filesystem, and pending pod signals. |
| Is this alert noisy or actionable? | Alertmanager route, runbook quality, and recent alert history. |

## Deployment Boundary

```mermaid
flowchart TB
    subgraph monitoring["monitoring namespace"]
        prometheus["Prometheus"]
        grafana["Grafana"]
        alertmanager["Alertmanager"]
        rules["Rule files"]
        dashboards["Dashboard JSON"]
    end

    subgraph workloads["application namespaces"]
        nginx["NGINX demo workload"]
        futureApps["Future annotated workloads"]
    end

    subgraph cluster["cluster-level metrics"]
        api["API server"]
        kubelet["Kubelet and cAdvisor"]
        nodes["node-exporter"]
        ksm["kube-state-metrics"]
    end

    prometheus --> api
    prometheus --> kubelet
    prometheus --> nodes
    prometheus --> ksm
    prometheus --> nginx
    prometheus --> futureApps
    rules --> prometheus
    dashboards --> grafana
```

### Boundary Notes

- Monitoring components should run in a dedicated namespace, normally `monitoring`.
- Application workloads should remain in their own namespaces.
- Prometheus needs RBAC permissions for discovery and node proxy metrics.
- Secrets for notification receivers or external integrations must not be committed to this repository.
- Dashboard JSON and rule files should be reviewed like application code because they affect incident response behavior.

## Failure Modes Shown by the Architecture

| Failure Mode | Visible Symptom | Likely File or Surface |
| --- | --- | --- |
| Bad scrape configuration | Targets down or missing metrics | `prometheus/prometheus.yml` |
| Missing kube-state-metrics | Kubernetes state alerts never evaluate | Cluster add-on and Prometheus targets |
| Missing node-exporter | Resource saturation alerts never evaluate | Cluster add-on and Prometheus targets |
| Broken alert route | Alert fires but no notification arrives | `alertmanager/alertmanager.yml` |
| Dashboard query drift | Grafana panels show no data | `grafana/dashboards/kubernetes-overview.json` |
| Runbook drift | Responder cannot map alert to action | `runbooks/kubernetes-alerts.md` |

## Review Checklist

Before changing the monitoring architecture:

1. Confirm which production question the change answers.
2. Confirm the required metrics are available in Prometheus.
3. Confirm dashboards and alerts use stable labels.
4. Confirm critical alerts route to the intended receiver.
5. Confirm runbooks include triage commands and escalation criteria.
6. Validate Prometheus rule syntax before commit.
7. Update diagrams when data flow, routing, or ownership changes.
