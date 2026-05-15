\# Monitoring Stack



This folder contains Kubernetes configuration for the homelab monitoring stack.



The core monitoring stack is installed with Helm using `kube-prometheus-stack`, which deploys:



\- Grafana

\- Prometheus

\- Alertmanager

\- Prometheus Operator

\- kube-state-metrics

\- node-exporter



\## Install



```powershell

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update



kubectl apply -f monitoring\\namespace.yaml



helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --timeout 15m

