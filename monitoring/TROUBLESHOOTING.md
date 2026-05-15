# Monitoring Troubleshooting Notes

This document records the main troubleshooting steps used while deploying `kube-prometheus-stack` in the homelab Kubernetes cluster.

## Environment

- Kubernetes OS: Talos Linux
- Monitoring namespace: `monitoring`
- Monitoring stack: `kube-prometheus-stack`
- Deployment method: Flux `HelmRelease`
- Ingress controller: Traefik
- Grafana hostname: `grafana.home.arpa`

## Issue 1: Helm Release Showed `failed`

After deploying `kube-prometheus-stack`, most monitoring pods were running, but Helm showed the release as failed.

```powershell
helm list -n monitoring
```

Example output:

```text
NAME                    NAMESPACE       REVISION        STATUS  CHART
kube-prometheus-stack   monitoring      5               failed  kube-prometheus-stack-85.0.2
```

The pods were mostly healthy:

```powershell
kubectl get pods -n monitoring
```

Example running components:

```text
alertmanager-kube-prometheus-stack-alertmanager-0
kube-prometheus-stack-grafana
kube-prometheus-stack-kube-state-metrics
kube-prometheus-stack-operator
prometheus-kube-prometheus-stack-prometheus-0
```

However, Helm history showed repeated failures related to `prometheus-node-exporter`.

```powershell
helm history kube-prometheus-stack -n monitoring
```

Example error:

```text
timeout waiting for: [DaemonSet/monitoring/kube-prometheus-stack-prometheus-node-exporter status: 'InProgress']
```

## Root Cause

The `prometheus-node-exporter` DaemonSet could not create pods because the `monitoring` namespace was using the Kubernetes `baseline` Pod Security level.

`node-exporter` needs host-level access to collect Linux node metrics such as:

- CPU usage
- Memory usage
- Disk usage
- Filesystem metrics
- Network usage
- Node uptime

The DaemonSet showed that Kubernetes wanted one node-exporter pod on each node, but none were being created.

```powershell
kubectl get ds -n monitoring
```

Problem state:

```text
NAME                                             DESIRED   CURRENT   READY   AVAILABLE
kube-prometheus-stack-prometheus-node-exporter   3         0         0       0
```

The DaemonSet events showed Pod Security blocking the pod creation:

```text
violates PodSecurity "baseline:latest": host namespaces, hostPath volumes, hostPort
```

## Fix

The `monitoring` namespace was updated to allow privileged workloads.

```powershell
kubectl label namespace monitoring pod-security.kubernetes.io/enforce=privileged --overwrite
kubectl label namespace monitoring pod-security.kubernetes.io/audit=privileged --overwrite
kubectl label namespace monitoring pod-security.kubernetes.io/warn=privileged --overwrite
```

Then the node-exporter DaemonSet was restarted.

```powershell
kubectl rollout restart ds kube-prometheus-stack-prometheus-node-exporter -n monitoring
```

After the fix:

```powershell
kubectl get ds -n monitoring
```

Working state:

```text
NAME                                             DESIRED   CURRENT   READY   AVAILABLE
kube-prometheus-stack-prometheus-node-exporter   3         3         3       3
```

Node-exporter was then running successfully on all three Talos nodes.

```powershell
kubectl get pods -n monitoring | findstr node
```

Example output:

```text
kube-prometheus-stack-prometheus-node-exporter-bkjkw   1/1   Running
kube-prometheus-stack-prometheus-node-exporter-g5p4w   1/1   Running
kube-prometheus-stack-prometheus-node-exporter-ldzm6   1/1   Running
```

## Issue 2: Manual Helm Upgrade Conflicted with Flux

After fixing node-exporter, a manual Helm upgrade was attempted.

```powershell
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --timeout 15m
```

The upgrade failed with conflicts similar to:

```text
conflicts with "helm-controller"
```

## Root Cause

The monitoring stack was being managed by Flux Helm Controller through a `HelmRelease`.

Because Flux owned the resources, running `helm upgrade` manually caused conflicts between the local Helm CLI and Flux's `helm-controller`.

The `HelmRelease` confirmed Flux ownership:

```powershell
kubectl describe helmrelease kube-prometheus-stack -n monitoring
```

Important status lines:

```text
Type: Ready
Reason: UpgradeSucceeded
Message: Helm upgrade succeeded for release monitoring/kube-prometheus-stack.v8
```

## Correct Management Method

Since Flux manages this stack, future changes should be made through GitOps.

Use these commands to check the Flux-managed release:

```powershell
kubectl get hr -n monitoring
kubectl describe hr kube-prometheus-stack -n monitoring
```

To force Flux to apply the current Git state:

```powershell
flux reconcile helmrelease kube-prometheus-stack -n monitoring
```

Avoid manually running:

```powershell
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

unless Flux ownership is intentionally removed first.

## Issue 3: Grafana Access Through Local DNS

Grafana was deployed successfully and exposed through Traefik.

Check the ingress:

```powershell
kubectl get ingress -n monitoring
```

Working output:

```text
NAME                            CLASS     HOSTS               ADDRESS         PORTS
kube-prometheus-stack-grafana   traefik   grafana.home.arpa   192.168.1.240   80
```

Check the Grafana service:

```powershell
kubectl get svc -n monitoring | findstr grafana
```

Example output:

```text
kube-prometheus-stack-grafana   LoadBalancer   10.106.213.238   192.168.1.242   80:30824/TCP
```

## Correct DNS Record

Because Grafana is exposed through Traefik, the local DNS record should point to the Traefik ingress IP.

```text
grafana.home.arpa -> 192.168.1.240
```

Access Grafana at:

```text
http://grafana.home.arpa
```

Do not point `grafana.home.arpa` to the Grafana LoadBalancer IP unless the goal is to bypass Traefik.

## Current Working State

The monitoring stack is now healthy and Flux-managed.

Working components:

- Grafana
- Prometheus
- Alertmanager
- Prometheus Operator
- kube-state-metrics
- node-exporter
- Default Grafana dashboards
- Default Prometheus rules
- ServiceMonitors
- Traefik ingress for Grafana

Final working status:

```text
Flux HelmRelease: Ready=True
Release status: deployed
node-exporter: 3/3 ready
Grafana ingress: grafana.home.arpa -> 192.168.1.240
```

## Useful Verification Commands

Check all monitoring pods:

```powershell
kubectl get pods -n monitoring
```

Check node-exporter DaemonSet:

```powershell
kubectl get ds -n monitoring
```

Check Flux HelmRelease:

```powershell
kubectl get hr -n monitoring
kubectl describe hr kube-prometheus-stack -n monitoring
```

Check Grafana ingress:

```powershell
kubectl get ingress -n monitoring
```

Check Grafana service:

```powershell
kubectl get svc -n monitoring | findstr grafana
```

Test local DNS:

```powershell
nslookup grafana.home.arpa
```

Expected DNS result:

```text
192.168.1.240
```

## Notes

- `node-exporter` requires privileged access because it reads host-level metrics from each Kubernetes node.
- The `monitoring` namespace must allow privileged workloads for node-exporter to run.
- Flux is the source of truth for the monitoring stack.
- Manual Helm upgrades can conflict with Flux-managed resources.
- Grafana should be accessed through Traefik at `grafana.home.arpa`.
- Local DNS should point `grafana.home.arpa` to the Traefik ingress IP.
