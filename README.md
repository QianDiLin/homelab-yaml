# Kubernetes Homelab

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

### Traefik

Traefik is used as the Kubernetes ingress controller. It receives traffic on the shared LoadBalancer IP and routes requests to the correct application based on hostname.

Current Traefik LoadBalancer IP:

```text
192.168.1.240
```

Applications are not normally accessed by the raw IP address. Instead, they are accessed through internal hostnames such as:

```text
homepage.home.arpa
longhorn.home.arpa
grafana.home.arpa
uptime.home.arpa
vaultwarden.home.arpa
forgejo.home.arpa
homeassistant.home.arpa
```

This allows multiple applications to share the same ingress IP while still routing to the correct backend service.

### Traefik Dashboard

The Traefik dashboard is not exposed publicly by default. For safety, it can be accessed locally using a temporary port-forward:

```powershell
kubectl port-forward -n traefik deployment/traefik 8080:8080
```

Then open:

```text
http://localhost:8080/dashboard/#/
```

This keeps the dashboard from being exposed as a normal network service while still allowing local administration when needed.

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
