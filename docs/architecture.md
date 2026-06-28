# Monitoring Architecture

This document describes the architecture for the Kubernetes monitoring stack. The current implementation includes the Prometheus configuration foundation, baseline Kubernetes alert rules, and a Kubernetes overview Grafana dashboard. It will be expanded as Alertmanager, additional dashboards, additional runbooks, and diagrams are added.

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

## Components

| Component | Role | Production Purpose |
| --- | --- | --- |
| Prometheus | Implemented configuration and rules | Discovers Kubernetes targets, scrapes platform and annotated workload metrics, and evaluates version-controlled alert rules. |
| Grafana | Implemented dashboard | Provides dashboards for cluster health, workload behavior, resource saturation, and incident investigation. |
| Alertmanager | Alert routing | Groups, deduplicates, silences, and routes alerts to the correct notification channel. |
| Kubernetes metrics sources | Observability inputs | Provide node, pod, deployment, service, and control plane metrics for Prometheus to scrape. |
| Runbooks | Response guidance | Convert alerts into repeatable investigation and mitigation steps. |

## Prometheus Discovery

Prometheus uses Kubernetes service discovery rather than fixed node or pod addresses.

| Scrape job | Discovery role | Metrics source |
| --- | --- | --- |
| `kubernetes-apiservers` | Endpoints | Kubernetes API server |
| `kubernetes-nodes` | Nodes | Kubelet metrics through the API server proxy |
| `kubernetes-cadvisor` | Nodes | Container resource metrics through the API server proxy |
| `kubernetes-pods` | Pods | Application metrics from explicitly annotated pods |

Pod scraping is opt-in. A workload must provide a metrics endpoint and annotations such as:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
prometheus.io/path: "/metrics"
```

The current NGINX workload does not expose Prometheus metrics and is therefore not annotated.

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

## Authentication and Authorization

The configuration assumes Prometheus runs inside the cluster. It uses:

- `/var/run/secrets/kubernetes.io/serviceaccount/token`
- `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
- The internal `kubernetes.default.svc` API endpoint

The Prometheus service account will require read access to Kubernetes discovery resources and access to node proxy metrics. The exact RBAC manifests will be added with the deployment layer rather than embedded in the scrape configuration.

TLS verification remains enabled using the Kubernetes cluster CA. The configuration does not use `insecure_skip_verify`.

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

Current baseline alert categories:

- Prometheus target health
- Pod crash loops
- Deployment replica mismatch
- Node readiness
- Pods stuck pending

Future alert categories:

- Node resource saturation
- Persistent volume pressure
- API server availability problems
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

Grafana dashboards should support fast visual diagnosis. The current `Kubernetes Overview` dashboard focuses on:

- Cluster overview
- Node health
- Workload health
- Deployment and pod reliability

Planned dashboard expansion will add:

- Namespace resource usage
- Alert status
- Persistent volume health
- API server health

Dashboards should not replace alerts. Dashboards are for investigation; alerts are for interruption.

## Design Assumptions

This repository assumes:

- Kubernetes is the primary workload platform.
- Prometheus is the primary metrics backend.
- Monitoring configuration is managed through Git.
- Operators need documentation that explains both how the stack works and how to respond when it fails.

## Future Expansion

Planned architecture additions:

- Prometheus alert rules under `prometheus/rules/`
- Grafana dashboard JSON under `grafana/dashboards/`
- Alertmanager routing under `alertmanager/`
- Alert runbooks under `runbooks/`
- Mermaid architecture diagrams under `docs/diagrams/`
