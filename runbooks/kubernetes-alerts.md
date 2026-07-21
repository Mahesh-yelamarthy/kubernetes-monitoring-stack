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

## KubernetesNodeHighCPUUsage

### Impact

Sustained high node CPU usage can delay workload scheduling, increase application latency, and reduce headroom for failover during an incident.

### Triage

```bash
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu
kubectl describe node <node>
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node> -o wide
```

Check:

- One workload consuming most node CPU.
- Recent deployments, batch jobs, or cron jobs.
- Missing or unrealistic CPU requests and limits.
- HorizontalPodAutoscaler or Cluster Autoscaler behavior.
- Whether the node is also reporting memory, disk, or network pressure.

### Mitigation

Scale the responsible workload only when it is safe and supported by capacity. Move non-critical workloads, add cluster capacity, correct runaway jobs, or tune CPU requests after confirming real usage patterns.

Avoid immediately raising limits without understanding whether the workload is inefficient, overloaded, or incorrectly scheduled.

## KubernetesNodeHighMemoryUsage

### Impact

Sustained high node memory usage increases the risk of pod eviction, OOM kills, degraded application performance, and cascading workload restarts.

### Triage

```bash
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
kubectl describe node <node>
kubectl get events --all-namespaces --sort-by=.lastTimestamp | grep -i -E 'evict|oom|memory'
```

Check:

- Pods with high resident memory growth.
- Recent deployments with changed memory behavior.
- Containers missing memory requests or limits.
- Eviction events or OOMKilled container states.
- DaemonSets or node agents consuming unexpected memory.

### Mitigation

Restore headroom by scaling or moving workloads, rolling back a memory-regressing release, adding capacity, or adjusting requests and limits after confirming expected usage.

If evictions are already happening, prioritize user-facing workloads and avoid draining the node until disruption budgets and replacement capacity are confirmed.

## KubernetesNodeFilesystemSpaceLow

### Impact

Low filesystem space can break container image pulls, log writes, kubelet operation, and application writes. If the node reaches disk pressure, Kubernetes may evict pods.

### Triage

```bash
kubectl describe node <node>
kubectl get events --all-namespaces --sort-by=.lastTimestamp | grep -i -E 'disk|evict|image'
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node> -o wide
```

On the affected node, inspect large consumers:

```bash
df -h
sudo du -xh /var/lib/containerd /var/log /var/lib/kubelet 2>/dev/null | sort -h | tail -30
```

Check:

- Excess container images or old image layers.
- Unexpected application log growth.
- Large emptyDir volumes.
- Failed cleanup or log rotation.
- Disk pressure node condition.

### Mitigation

Free space using approved node maintenance procedures, repair log rotation, clean unused images through the container runtime, or cordon and replace the node if cleanup is unsafe.

Do not delete Kubernetes-managed directories manually unless the platform team has a documented procedure for that runtime and node image.

## Post-Incident Follow-Up

After resolving a page-worthy alert:

- Record the trigger, affected service, start time, end time, and customer impact.
- Link the incident or ticket to the commit, deployment, or infrastructure event that caused it.
- Tune alert thresholds only when evidence shows the alert was noisy or late.
- Add missing dashboard panels or runbook steps discovered during response.
