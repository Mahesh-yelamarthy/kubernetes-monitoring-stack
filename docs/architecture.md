# Monitoring Architecture

This document describes the intended architecture for the Kubernetes monitoring stack. It starts with the Day 1 design foundation and will be expanded as Prometheus, Grafana, Alertmanager, dashboards, rules, runbooks, and diagrams are added.

## Architecture Goals

The monitoring stack should help an SRE answer four production questions quickly:

1. Is the cluster healthy?
2. Are workloads available and performing normally?
3. Are users or services impacted?
4. What should the responder check first during an incident?

The design emphasizes operational clarity over tool sprawl. Each component should have a clear responsibility, documented ownership, and a direct connection to incident response.

## Current Workload

The repository currently includes a small NGINX workload:

| Resource | File | Role |
| --- | --- | --- |
| Deployment | `manifests/nginx-deployment.yaml` | Runs NGINX pods with a defined replica count. |
| Service | `manifests/nginx-service.yaml` | Exposes the NGINX pods through a Kubernetes service. |

This workload gives the project an initial application target for future monitoring configuration, dashboards, and alert rules.

## Planned Components

| Component | Role | Production Purpose |
| --- | --- | --- |
| Prometheus | Metrics collection and rule evaluation | Scrapes Kubernetes and workload metrics, evaluates alert rules, and stores time series data for troubleshooting. |
| Grafana | Visualization | Provides dashboards for cluster health, workload behavior, resource saturation, and incident investigation. |
| Alertmanager | Alert routing | Groups, deduplicates, silences, and routes alerts to the correct notification channel. |
| Kubernetes metrics sources | Observability inputs | Provide node, pod, deployment, service, and control plane metrics for Prometheus to scrape. |
| Runbooks | Response guidance | Convert alerts into repeatable investigation and mitigation steps. |

## High-Level Flow

```text
User traffic
    |
    v
Kubernetes Service
    |
    v
NGINX Deployment
    |
    v
Kubernetes metrics endpoints and exporters
    |
    v
Prometheus scrape jobs
    |
    +--> Prometheus rules evaluate alert conditions
    |         |
    |         v
    |     Alertmanager routes notifications
    |
    v
Grafana dashboards support investigation
    |
    v
Runbooks guide incident response
```

## Namespace Strategy

Monitoring components should run in a dedicated namespace such as:

```text
monitoring
```

Using a dedicated namespace improves separation of concerns, access control, troubleshooting, and cleanup.

Application workloads may run in separate namespaces such as:

```text
production
staging
platform
```

The monitoring stack should be able to discover workloads across namespaces while still respecting Kubernetes RBAC.

## Metrics Strategy

The stack will prioritize metrics that are useful during production incidents:

- Node CPU, memory, disk, and network saturation
- Pod restarts, crash loops, and pending pods
- Deployment replica availability
- Container resource usage and limits
- Service availability and endpoint health
- API server and control plane health where available
- Application-level request rate, error rate, and latency when exposed

Metrics should be collected because they support reliability decisions, not simply because they are available.

## Alerting Strategy

Alerts should be actionable and tied to symptoms or high-risk infrastructure conditions.

Examples of future alert categories:

- Node resource saturation
- Pod crash loops
- Deployment replica mismatch
- Persistent volume pressure
- API server availability problems
- Missing scrape targets
- Service endpoint unavailability

Every page-worthy alert should eventually include:

- Severity
- Owner
- Business or service impact
- Triage commands
- Mitigation options
- Escalation path
- Link to a runbook

## Dashboard Strategy

Grafana dashboards should support fast visual diagnosis. Planned dashboards will focus on:

- Cluster overview
- Node health
- Workload health
- Namespace resource usage
- Alert status
- Deployment and pod reliability

Dashboards should not replace alerts. Dashboards are for investigation; alerts are for interruption.

## Design Assumptions

This repository assumes:

- Kubernetes is the primary workload platform.
- Prometheus is the primary metrics backend.
- Monitoring configuration is managed through Git.
- Operators need documentation that explains both how the stack works and how to respond when it fails.

## Future Expansion

Planned architecture additions:

- Prometheus configuration under `prometheus/`
- Grafana dashboard JSON under `grafana/dashboards/`
- Alertmanager routing under `alertmanager/`
- Alert runbooks under `runbooks/`
- Mermaid architecture diagrams under `docs/diagrams/`
