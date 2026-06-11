# Kubernetes Monitoring Stack

Production-oriented Kubernetes monitoring and observability project built around Prometheus, Grafana, Alertmanager, and Kubernetes operational practices.

This repository is part of a 30-day SRE / DevOps portfolio build. The project is intentionally developed in realistic daily increments to show how a monitoring platform evolves from a basic Kubernetes workload into a production-ready observability system with dashboards, alerts, runbooks, and incident response documentation.

## Purpose

The goal of this repository is to demonstrate how an SRE would design, document, and operate a Kubernetes monitoring stack that supports production reliability work.

This project will gradually cover:

- Kubernetes workload and cluster monitoring
- Prometheus scrape configuration
- Prometheus alerting rules
- Grafana dashboards
- Alertmanager routing
- Operational runbooks
- Troubleshooting guides
- Incident response documentation
- Production monitoring practices

## Current Status

Day 4 Prometheus configuration foundation is complete.

The repository contains a baseline NGINX Kubernetes workload and an in-cluster Prometheus configuration. Prometheus can discover the Kubernetes API server, node metrics, cAdvisor metrics, and pods explicitly enabled through annotations.

Grafana, Alertmanager, alert rules, runbooks, and architecture diagrams will be added in future commits.

## Repository Structure

```text
kubernetes-monitoring-stack/
├── README.md
├── docs/
│   ├── architecture.md
│   └── operations.md
├── manifests/
│   ├── nginx-deployment.yaml
│   └── nginx-service.yaml
└── prometheus/
    ├── prometheus.yml
    └── rules/
        └── README.md
```

Planned future directories:

```text
grafana/
runbooks/
docs/diagrams/
alertmanager/
```

## Prometheus Configuration

The base configuration is stored at:

```text
prometheus/prometheus.yml
```

Configured scrape jobs:

| Job | Purpose |
| --- | --- |
| `prometheus` | Monitors the Prometheus server itself. |
| `kubernetes-apiservers` | Scrapes the Kubernetes API server metrics endpoint. |
| `kubernetes-nodes` | Discovers nodes and scrapes kubelet metrics through the API server proxy. |
| `kubernetes-cadvisor` | Collects container resource metrics from cAdvisor through the API server proxy. |
| `kubernetes-pods` | Discovers pods with `prometheus.io/scrape: "true"` annotations. |

The configuration assumes Prometheus runs inside Kubernetes with:

- A mounted service account token
- The cluster CA certificate
- RBAC permission to discover nodes, pods, services, and endpoints
- Permission to access node proxy metrics

Validate the configuration before deployment:

```bash
promtool check config prometheus/prometheus.yml
```

The `cluster` and `environment` external labels are development placeholders and should be overridden for each deployed environment.

## Existing Kubernetes Workload

The current workload is a small NGINX application deployed with Kubernetes manifests.

| File | Purpose |
| --- | --- |
| `manifests/nginx-deployment.yaml` | Defines the NGINX deployment and replica count. |
| `manifests/nginx-service.yaml` | Exposes the NGINX deployment through a Kubernetes service. |

Apply the workload:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
kubectl apply -f manifests/nginx-service.yaml
```

Verify the workload:

```bash
kubectl get pods
kubectl get svc
```

## Monitoring Goals

This repository will grow into a monitoring stack that helps answer production questions quickly:

- Are Kubernetes nodes healthy?
- Are workloads available and correctly replicated?
- Are pods restarting or stuck in pending state?
- Are CPU, memory, disk, or network resources saturated?
- Are alert notifications actionable and routed correctly?
- Does every critical alert have a runbook?

## Operating Principles

This project follows production monitoring principles used by SRE and platform teams:

- Monitor symptoms before causes where possible.
- Prefer actionable alerts over noisy alert volume.
- Use dashboards for investigation and alerts for interruption.
- Keep monitoring configuration under version control.
- Connect critical alerts to runbooks.
- Document ownership, escalation, and response expectations.
- Review alert quality after incidents and recurring noise.

## Documentation

- [Architecture](docs/architecture.md)
- [Operations](docs/operations.md)

## Recruiter Signal

This repository is designed to show practical SRE capabilities:

- Kubernetes observability knowledge
- Production monitoring design
- Alerting and incident response discipline
- Infrastructure-as-documentation habits
- Operational thinking beyond simple tool installation

## Day 4 Commit

```text
feat: add base prometheus configuration
```
