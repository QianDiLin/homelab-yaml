# Qian's Homelab

A self-hosted homelab built to practice real-world infrastructure, Kubernetes administration, GitOps, networking, monitoring, and DevOps workflows. This environment is designed as both a personal home server stack and a portfolio project for entry-level IT, help desk, NOC, networking, and DevOps-focused roles.

## Overview

This homelab runs a Kubernetes-based application platform on top of a virtualized home server environment. The cluster is used to host self-managed services such as a homepage dashboard, password management, uptime monitoring, Git hosting, home automation, persistent storage, and observability tools.

The goal of this project is to show practical experience with:

* Kubernetes cluster administration
* Talos Linux node management
* GitOps with Flux CD
* Ingress routing with Traefik
* Persistent storage with Longhorn
* Monitoring with Grafana and Prometheus
* Self-hosted application deployment
* YAML-based infrastructure management
* Home network services and internal DNS-style hostnames

## Architecture

The homelab is built around a Kubernetes cluster running Talos Linux nodes. Applications are deployed into separate Kubernetes namespaces and exposed through Traefik using internal `.home.arpa` hostnames.

```text
Windows Admin PC
        |
        | kubectl / flux / helm / git
        |
Talos Kubernetes Cluster
        |
        |-- Traefik Ingress Controller
        |-- Longhorn Storage
        |-- Flux CD GitOps
        |-- Homepage Dashboard
        |-- Vaultwarden
        |-- Uptime Kuma
        |-- Forgejo
        |-- Home Assistant
        |-- Grafana / Prometheus Stack
```

## Core Infrastructure

### Virtualization

The homelab is hosted on a Proxmox-based environment. Proxmox is used to create and manage virtual machines that form the Kubernetes cluster.

### Kubernetes

The cluster is built with Talos Linux, an immutable operating system designed specifically for Kubernetes. Talos is used to reduce manual server configuration and keep the cluster focused on declarative management.

Current cluster design:

* Talos Linux Kubernetes nodes
* One control-plane node
* Multiple worker nodes
* Kubernetes-managed application workloads
* Cluster access from a Windows administration workstation using `kubectl`

Example cluster access command:

```powershell
kubectl get nodes
```

### GitOps

Flux CD is used to synchronize Kubernetes manifests from this GitHub repository into the cluster. Changes are made locally, committed to Git, pushed to GitHub, and then reconciled by Flux.

Typical update workflow:

```powershell
git add .
git commit -m "Update homelab configuration"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system --timeout=5m
```

This allows the cluster state to be managed from version-controlled YAML files instead of one-off manual commands.

## Networking and Ingress

The homelab network is built around OPNsense, Pi-hole, a managed switch, VLANs, Proxmox virtualization, and a Talos Kubernetes cluster. OPNsense and Pi-hole run as virtual machines on the Proxmox host, while the Kubernetes applications run on Talos VMs.

```text
Network Diagram

                [ Internet ]
                     │
              [ Proxmox Host ]
                     │
        ┌────────────┴────────────┐
        │   OPNsense VM           │
        │   Router / Firewall     │
        │   VLANs / DHCP          │
        └────────────┬────────────┘
                     │
              [ Managed Switch ]
          ┌──────────┼──────────┐
          │          │          │
      Main PC     VLAN 10    VLAN 20
   Admin PC      Servers     Lab / Test
                     │
              [ Proxmox Host ]
          ┌──────────┼──────────────────────────┐
          │          │                          │
       Pi-hole   Talos Kubernetes Cluster       Lab VMs
       DNS       │                              Test Apps
       Adblock   │                              Temporary VMs
                 │
          ┌──────┴──────────────────┐
          │  Traefik Ingress         │
          │  192.168.1.240           │
          │                          │
          ├─ Homepage
          ├─ Longhorn
          ├─ Vaultwarden
          ├─ Uptime Kuma
          ├─ Forgejo
          ├─ Home Assistant
          └─ Grafana / Prometheus
```

### OPNsense

OPNsense runs as a VM on the Proxmox host and acts as the main router and firewall for the homelab. It handles internet access, firewall rules, VLAN routing, DHCP, and network segmentation between the main network, server network, and lab/test network.

This gives the homelab a more realistic network design because traffic is routed and filtered through a dedicated firewall instead of relying only on a basic consumer router.

### Pi-hole

Pi-hole also runs as a VM on the Proxmox host, separate from Kubernetes. It provides DNS and ad-blocking for the homelab network.

Pi-hole is used for internal DNS records so services can be accessed with readable hostnames instead of IP addresses. These hostnames point to the Traefik LoadBalancer IP.

Example internal DNS records:

```text
homepage.home.arpa      -> 192.168.1.240
longhorn.home.arpa      -> 192.168.1.240
grafana.home.arpa       -> 192.168.1.240
uptime.home.arpa        -> 192.168.1.240
vaultwarden.home.arpa   -> 192.168.1.240
forgejo.home.arpa       -> 192.168.1.240
homeassistant.home.arpa -> 192.168.1.240
```

### Traefik

Traefik runs inside the Talos Kubernetes cluster as the ingress controller. It receives traffic on the shared LoadBalancer IP:

```text
192.168.1.240
```

Applications are not normally accessed by the raw IP address. Instead, Pi-hole resolves internal hostnames like `homepage.home.arpa` or `grafana.home.arpa` to `192.168.1.240`. Traefik then uses the hostname in the request to route traffic to the correct Kubernetes service.

This allows multiple applications to share the same ingress IP while still routing correctly by hostname.

### Traefik Dashboard

The Traefik dashboard is not exposed as a normal public service. For safer access, it is opened only when needed through a local port-forward:

```powershell
kubectl port-forward -n traefik deployment/traefik 8080:8080
```

Then open:

```text
http://localhost:8080/dashboard/#/
```

This keeps the dashboard from being permanently exposed on the network while still allowing local administration from the Windows admin PC.

## Storage

### Longhorn

Longhorn provides persistent storage for Kubernetes workloads. It allows applications to use PersistentVolumeClaims instead of relying only on temporary container storage.

Longhorn is used for services that need durable data, such as application configuration, databases, and stateful workloads.

Longhorn UI:

```text
http://longhorn.home.arpa
```

Example storage check:

```powershell
kubectl get pvc -A
```

## Monitoring and Observability

### Grafana and Prometheus

The monitoring stack is based on `kube-prometheus-stack`, which provides Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter components.

Grafana is exposed internally through Traefik:

```text
http://grafana.home.arpa
```

The monitoring stack is intended to provide visibility into:

* Kubernetes node health
* Pod status
* Application uptime
* Cluster resource usage
* Persistent volume status
* Service availability
* Alerting and troubleshooting data

### Uptime Kuma

Uptime Kuma is used for service availability monitoring. It provides a simple dashboard for checking whether internal applications and services are reachable.

Uptime Kuma URL:

```text
http://uptime.home.arpa
```

## Applications

### Homepage

Homepage is used as the central dashboard for the homelab. It provides quick links to infrastructure tools, self-hosted apps, and monitoring services.

Homepage URL:

```text
http://homepage.home.arpa
```

Homepage is managed through Kubernetes YAML and updated through the GitOps workflow.

### Vaultwarden

Vaultwarden is a lightweight self-hosted password manager compatible with Bitwarden clients.

Vaultwarden URL:

```text
http://vaultwarden.home.arpa
```

This service demonstrates hosting a security-focused application with persistent storage and ingress routing.

### Forgejo

Forgejo is a self-hosted Git service used to practice source control, repository hosting, and DevOps workflows.

Forgejo URL:

```text
http://forgejo.home.arpa
```

This can be used for internal projects, automation files, and future CI/CD experiments.

### Home Assistant

Home Assistant is used for home automation and smart home management.

Home Assistant URL:

```text
http://homeassistant.home.arpa
```

This service allows the homelab to connect infrastructure practice with real home automation use cases.

## Screenshots

Screenshots can be added to show the homelab running in a real environment. Store screenshots in an `images/` folder inside this repository.

```text
homelab-yaml/
├── images/
│   ├── homepage-dashboard.png
│   ├── kubernetes-nodes.png
│   ├── traefik-dashboard.png
│   ├── longhorn-dashboard.png
│   ├── grafana-dashboard.png
│   └── proxmox-vms.png
```

### Homepage Dashboard

![Homepage Dashboard](images/homepage-dashboard.png)

The Homepage dashboard provides a central landing page for the homelab and links to the main self-hosted services.

### Kubernetes Cluster Nodes

![Kubernetes Nodes](images/kubernetes-nodes.png)

This screenshot shows the Talos Kubernetes nodes running and reachable from the Windows administration PC using `kubectl`.

### Traefik Ingress Dashboard

![Traefik Dashboard](images/traefik-dashboard.png)

The Traefik dashboard shows ingress routing, routers, services, and middleware used to expose Kubernetes applications through internal hostnames.

### Longhorn Storage Dashboard

![Longhorn Dashboard](images/longhorn-dashboard.png)

Longhorn provides persistent storage for Kubernetes workloads and allows storage health, volumes, replicas, and nodes to be monitored from a web dashboard.

### Grafana Monitoring

![Grafana Dashboard](images/grafana-dashboard.png)

Grafana is used to visualize Kubernetes and homelab monitoring data from the Prometheus stack.

### Proxmox Virtual Machines

![Proxmox VMs](images/proxmox-vms.png)

The Proxmox view shows the virtual machines used to run OPNsense, Pi-hole, Talos Kubernetes nodes, and lab/test systems.

Before uploading screenshots, sensitive information such as passwords, tokens, private keys, public IP addresses, and personal email addresses should be hidden or cropped out.

## Repository Structure

This repository stores the YAML configuration used to manage the Kubernetes homelab.

Example structure:

```text
homelab-yaml/
├── apps/
│   ├── homepage/
│   ├── vaultwarden/
│   ├── uptime-kuma/
│   ├── forgejo/
│   ├── homeassistant/
│   └── monitoring/
├── infrastructure/
│   ├── traefik/
│   ├── longhorn/
│   └── flux-system/
├── README.md
└── kustomization.yaml
```

Actual folder names may vary as the project develops, but the goal is to keep application configuration and infrastructure configuration organized and easy to maintain.

## Common Commands

### Check cluster nodes

```powershell
kubectl get nodes
```

### Check all pods

```powershell
kubectl get pods -A
```

### Check ingress routes

```powershell
kubectl get ingress -A
```

### Check services

```powershell
kubectl get svc -A
```

### Reconcile Flux

```powershell
flux reconcile source git flux-system
flux reconcile kustomization flux-system --timeout=5m
```

### Restart Homepage after a ConfigMap update

```powershell
kubectl rollout restart deployment homepage -n homepage
```

### Access Traefik dashboard locally

```powershell
kubectl port-forward -n traefik deployment/traefik 8080:8080
```

Then open:

```text
http://localhost:8080/dashboard/#/
```

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

## Future Improvements

Planned improvements for this homelab include:

* Move more Helm-installed infrastructure into Flux-managed HelmReleases
* Add HTTPS certificates for internal services
* Improve monitoring dashboards in Grafana
* Add alerting rules for node, pod, and service failures
* Create a cleaner folder structure for all Kubernetes manifests
* Add backup strategy for Longhorn volumes
* Add network segmentation and VLAN documentation
* Add CI/CD workflows for application deployment
* Document recovery steps for rebuilding the cluster
* Add screenshots and diagrams for portfolio presentation

## Purpose

This homelab is more than a collection of self-hosted apps. It is a practical learning environment for building, breaking, fixing, and documenting real infrastructure. The project helps develop the same troubleshooting habits used in IT support, NOC, systems administration, networking, and DevOps roles.

The main goal is to continue improving the environment over time while keeping the configuration documented, version-controlled, and reproducible.
