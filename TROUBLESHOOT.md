## Troubleshooting Notes

### `localhost:8080` Kubernetes error

If `kubectl` or `flux` shows an error like:

```text
Get "http://localhost:8080/api": dial tcp [::1]:8080: connectex: No connection could be made
```

It usually means the Windows PC cannot find the Kubernetes kubeconfig file. The fix is to set or copy the kubeconfig file:

```powershell
$env:KUBECONFIG="C:\Users\qiand\homelab-yaml\kubeconfig"
kubectl get nodes
```

Or copy it to the default location:

```powershell
mkdir $env:USERPROFILE\.kube -Force
copy C:\Users\qiand\homelab-yaml\kubeconfig $env:USERPROFILE\.kube\config
```

### Raw Traefik IP shows NGINX page

Opening this may show an NGINX welcome page:

```text
http://192.168.1.240
```

That does not mean Traefik is broken. The IP is shared by all ingress routes. Traefik needs the correct hostname to route traffic to the right application.

Use hostnames like:

```text
http://homepage.home.arpa
http://grafana.home.arpa
http://longhorn.home.arpa
```

instead of the raw IP address.

### Homepage does not update after Git push

If Homepage does not change after editing the YAML, run:

```powershell
flux reconcile source git flux-system
flux reconcile kustomization flux-system --timeout=5m
kubectl rollout restart deployment homepage -n homepage
```

Then refresh the browser with `Ctrl + F5`.

### Prometheus stuck during startup with Longhorn PVC

While setting up the monitoring stack, I deployed `kube-prometheus-stack` into the `monitoring` namespace. This installed Grafana, Prometheus, Alertmanager, kube-state-metrics, node-exporter, and the Prometheus Operator.

The goal was to monitor both the Kubernetes cluster and the rest of the homelab, including Proxmox, OPNsense, Windows Server, Linux VMs, storage, and application uptime.

During the first deployment, most of the monitoring components started correctly. However, Prometheus did not fully start.

#### Problem observed

After checking the pods in the `monitoring` namespace, Grafana, Alertmanager, kube-state-metrics, and the Prometheus Operator were running. The Prometheus pod was stuck during initialization.

Command used:

```powershell
kubectl get pods -n monitoring
```

Example output:

```text
NAME                                                        READY   STATUS     RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running    0          48m
kube-prometheus-stack-grafana-8b45c57df-rdxnk               3/3     Running    0          48m
kube-prometheus-stack-kube-state-metrics-857ccb48f8-8c7fb   1/1     Running    0          48m
kube-prometheus-stack-operator-849fc5f997-p6zbq             1/1     Running    0          48m
prometheus-kube-prometheus-stack-prometheus-0               0/2     Init:0/1   0          3m38s
```

The important part was:

```text
prometheus-kube-prometheus-stack-prometheus-0    0/2    Init:0/1
```

This showed that the Prometheus pod was not reaching the normal `Running` state. Since the other monitoring components were running, the issue appeared to be isolated to Prometheus rather than the entire Helm deployment.

#### Troubleshooting process

The first step was to confirm whether the whole monitoring stack was broken or whether the issue was limited to Prometheus.

```powershell
kubectl get pods -n monitoring
```

Most of the stack was running successfully, which showed that the Helm chart installed and that the namespace was functional.

Next, I narrowed the output to Prometheus-related pods.

```powershell
kubectl get pods -n monitoring | findstr prometheus
```

This confirmed that the Prometheus StatefulSet pod was the main issue.

Because Prometheus was configured to use persistent storage, I checked the PVCs in the `monitoring` namespace.

```powershell
kubectl get pvc -n monitoring
```

The goal was to verify whether the Prometheus volume was created, bound, or stuck. Since Prometheus stores its time-series database on disk, a storage issue can prevent the pod from starting correctly.

Next, I inspected the Prometheus pod for Kubernetes events, volume mount issues, init container problems, or permission errors.

```powershell
kubectl describe pod prometheus-kube-prometheus-stack-prometheus-0 -n monitoring
```

This command is useful because it shows:

* init container status
* volume mount errors
* scheduling issues
* failed attach or mount events
* container restart information
* Kubernetes events related to the pod

Since the pod was stuck at `Init:0/1`, I focused on the init container and storage mounting process.

I also checked logs to see whether Prometheus or one of its init containers was failing.

```powershell
kubectl logs prometheus-kube-prometheus-stack-prometheus-0 -n monitoring
```

If the main Prometheus container has not started yet, the init container logs can be checked instead:

```powershell
kubectl logs prometheus-kube-prometheus-stack-prometheus-0 -n monitoring -c <container-name>
```

This helped separate whether the issue was caused by Prometheus itself, the config reloader, or the storage volume.

#### Likely cause

The issue appeared to be related to Prometheus using a Longhorn-backed persistent volume during startup.

Prometheus is storage-heavy because it constantly writes time-series data. If the persistent volume has an attach, mount, permission, or initialization issue, the Prometheus pod may remain stuck before the main containers start.

Since Grafana, Alertmanager, kube-state-metrics, and the operator were already running, the issue was not the entire monitoring stack. The problem was most likely tied to Prometheus storage configuration.

#### Decision made

Instead of continuing to troubleshoot Longhorn storage before the monitoring stack was functional, I decided to simplify the deployment.

The revised plan was:

```text
1. Run the base monitoring stack first
2. Confirm Grafana and Prometheus work without Longhorn persistence
3. Verify Kubernetes dashboards
4. Add application uptime checks
5. Add external homelab monitoring targets
6. Revisit Prometheus persistent storage later
```

This allowed me to separate the issue into two questions:

```text
Is the monitoring stack working?
Is Longhorn storage working correctly for Prometheus?
```

By separating those problems, the environment became easier to troubleshoot.

#### Workaround

The short-term workaround was to avoid using Longhorn-backed persistent storage for Prometheus during the first deployment.

Instead of forcing Prometheus onto Longhorn immediately, I kept the initial monitoring setup lightweight so the base stack could start successfully.

The key lesson was:

```text
Get the application working first, then add persistent storage.
```

For Prometheus, this means starting with a simple deployment first, then later tuning:

* retention time
* storage size
* Longhorn PVC settings
* backup strategy
* alerting rules
* dashboard persistence

#### Why I did not keep troubleshooting Longhorn immediately

Longhorn was not removed from the homelab. The issue was that Prometheus did not need to be the first service used to test Longhorn persistence.

Prometheus constantly writes metrics, so it can expose storage problems faster than simpler applications. For the first version of the monitoring stack, I prioritized getting visibility into the cluster over keeping long-term metrics history.

This made the setup easier to manage because I could first confirm that Grafana, Prometheus, Alertmanager, node-exporter, and kube-state-metrics worked correctly before adding more complex storage requirements.

#### Commands used

Check all monitoring pods:

```powershell
kubectl get pods -n monitoring
```

Check Prometheus pods:

```powershell
kubectl get pods -n monitoring | findstr prometheus
```

Check services:

```powershell
kubectl get svc -n monitoring
```

Check persistent volume claims:

```powershell
kubectl get pvc -n monitoring
```

Describe the Prometheus pod:

```powershell
kubectl describe pod prometheus-kube-prometheus-stack-prometheus-0 -n monitoring
```

Check Prometheus logs:

```powershell
kubectl logs prometheus-kube-prometheus-stack-prometheus-0 -n monitoring
```

Check logs for a specific container:

```powershell
kubectl logs prometheus-kube-prometheus-stack-prometheus-0 -n monitoring -c <container-name>
```

Restart the Prometheus pod:

```powershell
kubectl delete pod prometheus-kube-prometheus-stack-prometheus-0 -n monitoring
```

#### Lessons learned

This issue showed that Kubernetes troubleshooting should be done in layers. Installing the full monitoring stack with persistent storage, ingress, dashboards, uptime checks, and external exporters all at once makes troubleshooting harder.

A better approach is:

```text
1. Confirm the Helm chart installs correctly
2. Confirm the pods start
3. Confirm Grafana is accessible
4. Confirm Prometheus can scrape Kubernetes metrics
5. Add ingress
6. Add uptime checks
7. Add external exporters
8. Add persistent storage
9. Add alerts
```

The main troubleshooting lesson was that a pod stuck in `Init:0/1` should be investigated by checking:

* pod description
* Kubernetes events
* init container status
* init container logs
* PVC status
* storage backend health

This also showed why persistent storage should be added carefully. Longhorn is useful for Kubernetes storage, but Prometheus should only be placed on Longhorn after the base monitoring stack is confirmed to work.

#### Future improvements

A future improvement is to retry Prometheus persistence with a more controlled Longhorn configuration.

Planned improvements include:

* Create a dedicated Longhorn storage class for monitoring
* Set a reasonable Prometheus retention period
* Test with a smaller PVC first
* Confirm volume attach and mount behavior
* Add alerts for Longhorn volume health
* Add dashboards for Longhorn storage usage
* Back up Grafana dashboards and Prometheus configuration
* Add Alertmanager notifications for critical homelab services

## Skills Demonstrated

This project demonstrates hands-on experience with:

* Kubernetes administration
* Talos Linux
* Proxmox virtualization
* Git and GitHub
* GitOps with Flux CD
* Helm-based application deployment
* Kubernetes ingress with Traefik
* Persistent storage with Longhorn
* Monitoring with Grafana and Prometheus
* YAML configuration management
* Basic service troubleshooting
* Internal DNS-style routing
* Self-hosted application operations
