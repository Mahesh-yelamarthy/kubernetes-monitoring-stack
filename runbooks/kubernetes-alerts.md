# Kubernetes Alert Runbook

This runbook supports the baseline Prometheus alerts in `prometheus/rules/kubernetes-alerts.yml`.

## Metric Dependencies

These alerts assume Prometheus can scrape:

- Native target health through the `up` metric
- Kubernetes node, pod, and deployment state metrics from kube-state-metrics
- Kubelet and cAdvisor metrics for deeper resource investigation

If an alert never fires when expected, first verify that the required metrics exist in Prometheus.

## General Triage

Start every alert investigation by capturing context:

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
kubectl get events --all-namespaces --sort-by=.lastTimestamp
```

Then identify whether the problem is isolated to one workload, one namespace, one node, or the full cluster.

## PrometheusTargetDown

### Impact

Prometheus cannot scrape a configured target. Monitoring data may be missing, and dependent alerts or dashboards may be unreliable.

### Triage

```bash
kubectl -n monitoring get pods -o wide
kubectl -n monitoring get svc,endpoints
kubectl -n monitoring logs deploy/prometheus --tail=100
```

Check:

- Target pod or service exists.
- Endpoint has ready addresses.
- NetworkPolicy is not blocking Prometheus.
- TLS or service account authentication is valid.
- The target metrics path responds.

### Mitigation

Restore the target, fix the service or endpoint selector, repair credentials, or roll back the scrape configuration change that introduced the failure.

## KubernetesNodeNotReady

### Impact

A NotReady node cannot reliably run workloads. Pods may be evicted, stuck terminating, or rescheduled to other nodes.

### Triage

```bash
kubectl describe node <node>
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node>
kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -50
```

Check:

- Kubelet status and node conditions.
- Disk pressure, memory pressure, PID pressure, or network unavailable conditions.
- Recent node maintenance or autoscaler activity.
- Cloud provider or infrastructure health.

### Mitigation

Cordon the node if workloads are at risk:

```bash
kubectl cordon <node>
```

Drain only after confirming disruption budgets and workload redundancy:

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

Escalate to the platform or infrastructure owner when the node does not recover quickly.

## KubernetesPodCrashLooping

### Impact

A workload container is repeatedly restarting. The service may be degraded or unavailable.

### Triage

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> -c <container> --previous
kubectl -n <namespace> get deploy,rs,pod -o wide
```

Check:

- Recent image or configuration changes.
- Failed liveness or startup probes.
- Missing secrets, config maps, or permissions.
- OOMKilled status or resource limits.
- Application startup errors.

### Mitigation

Roll back a bad release, restore missing configuration, adjust probes when they are incorrect, or scale known-good capacity while the failed version is investigated.

## KubernetesPodPending

### Impact

The scheduler cannot place a pod. Capacity, constraints, image pulls, storage, or admission controls may be blocking workload startup.

### Triage

```bash
kubectl -n <namespace> describe pod <pod>
kubectl get nodes -o wide
kubectl describe nodes
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Check:

- Insufficient CPU or memory.
- Node selectors, affinity, anti-affinity, or taints.
- PersistentVolumeClaim binding failures.
- Image pull errors.
- Admission webhook failures.

### Mitigation

Add capacity, correct scheduling constraints, repair storage claims, fix image references or registry access, or roll back the workload change.

## KubernetesDeploymentReplicasUnavailable

### Impact

A deployment has fewer available replicas than desired. If redundancy is reduced, a single additional failure may become user visible.

### Triage

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> get rs,pods -l app=<app-label> -o wide
kubectl -n <namespace> rollout status deployment/<deployment>
```

Check:

- Rollout is paused or stuck.
- Pods are crash looping, pending, or failing readiness.
- The new ReplicaSet is using the expected image.
- Resource requests cannot be scheduled.
- Readiness probes are too strict or genuinely failing.

### Mitigation

If a rollout caused the issue, roll back to the last known-good revision:

```bash
kubectl -n <namespace> rollout undo deployment/<deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=120s
```

If the issue is capacity-related, restore enough healthy replicas before continuing the release.

## Post-Incident Follow-Up

After resolving a page-worthy alert:

- Record the trigger, affected service, start time, end time, and customer impact.
- Link the incident or ticket to the commit, deployment, or infrastructure event that caused it.
- Tune alert thresholds only when evidence shows the alert was noisy or late.
- Add missing dashboard panels or runbook steps discovered during response.
