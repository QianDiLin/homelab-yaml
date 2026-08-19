\# NOC Technician Lab



\## Overview



This project simulates a small Network Operations Center environment using my existing homelab infrastructure.



The lab focuses on the core responsibilities of an entry-level NOC Technician, including infrastructure monitoring, alert triage, network troubleshooting, service validation, incident documentation, escalation, and recovery verification.



Instead of building an isolated environment, the NOC lab uses my existing Proxmox, OPNsense, Talos Linux, Kubernetes, monitoring, DNS, storage, and application infrastructure as monitored production-style systems.



The goal is to practice identifying failures from alerts, determining the affected layer, restoring service safely, validating recovery, and documenting the incident clearly.



\---



\# Lab Goals



The lab was designed to practice:



\* Network monitoring

\* Alert triage

\* Infrastructure monitoring

\* SNMP concepts

\* Syslog concepts

\* ICMP and service availability checks

\* DNS troubleshooting

\* DHCP troubleshooting

\* Layer 1/2/3 troubleshooting

\* VLAN troubleshooting

\* Routing troubleshooting

\* Firewall troubleshooting

\* Linux troubleshooting

\* Kubernetes service monitoring

\* Incident prioritization

\* Escalation

\* Root cause documentation

\* Recovery validation



\---



\# Environment



\## Existing Infrastructure



The NOC lab monitors systems already running in my homelab.



Example environment:



```text

Internet

&#x20;  |

&#x20;  |

OPNsense

&#x20;  |

&#x20;  |

Managed Network

&#x20;  |

&#x20;  +-------------------+

&#x20;  |                   |

Proxmox Cluster     Network Devices

&#x20;  |

&#x20;  +-----------------------------+

&#x20;  |                             |

Talos Kubernetes              Other VMs

&#x20;  |

&#x20;  +-----------------------------+

&#x20;  |

Applications / Services

```



Core infrastructure includes:



\* 3-node Proxmox cluster

\* OPNsense firewall/router

\* Pi-hole DNS

\* Talos Linux

\* Kubernetes

\* Longhorn

\* Traefik

\* MetalLB

\* Prometheus

\* Grafana

\* Alertmanager

\* Blackbox Exporter

\* Uptime Kuma

\* Home Assistant

\* Vaultwarden

\* Forgejo

\* n8n

\* Speedtest Tracker



\---



\# NOC Monitoring Architecture



The lab uses several monitoring layers.



```text

Infrastructure

&#x20;    |

&#x20;    +-------- Prometheus

&#x20;    |

&#x20;    +-------- Blackbox Exporter

&#x20;    |

&#x20;    +-------- Uptime Kuma

&#x20;    |

&#x20;    +-------- Kubernetes Metrics

&#x20;    |

&#x20;    +-------- System Logs

&#x20;    |

&#x20;    v

&#x20;  Grafana

&#x20;    |

&#x20;    v

Alertmanager / Alerts

&#x20;    |

&#x20;    v

NOC Investigation

```



Each tool provides a different perspective.



| Tool              | Purpose                              |

| ----------------- | ------------------------------------ |

| Prometheus        | Metrics collection                   |

| Grafana           | Visualization and dashboards         |

| Alertmanager      | Alert routing and notification logic |

| Blackbox Exporter | Endpoint/service probing             |

| Uptime Kuma       | Availability monitoring              |

| kubectl           | Kubernetes health validation         |

| Linux CLI         | Host and service troubleshooting     |

| OPNsense          | Firewall/routing investigation       |



\---



\# 1. Verify Kubernetes Monitoring



The first step is confirming that the monitoring stack is healthy.



```powershell

kubectl get pods -n monitoring

```



Check services:



```powershell

kubectl get svc -n monitoring

```



Check Prometheus:



```powershell

kubectl get pods -n monitoring | findstr prometheus

```



Check Grafana:



```powershell

kubectl get pods -n monitoring | findstr grafana

```



Check Alertmanager:



```powershell

kubectl get pods -n monitoring | findstr alertmanager

```



Check Blackbox Exporter:



```powershell

kubectl get pods -n monitoring | findstr blackbox

```



\---



\# 2. Verify Monitoring Helm Releases



Because the monitoring stack is managed with Flux and Helm, release state can be checked using:



```powershell

flux get helmreleases -A

```



Expected monitoring releases include:



```text

monitoring/kube-prometheus-stack

monitoring/blackbox-exporter

```



Check reconciliation:



```powershell

flux get kustomizations -A

```



\---



\# 3. Verify Grafana Access



Check service:



```powershell

kubectl get svc -n monitoring

```



Check ingress:



```powershell

kubectl get ingress -n monitoring

```



Example:



```text

grafana.home.arpa

```



Test name resolution:



```powershell

nslookup grafana.home.arpa

```



Test application response:



```powershell

curl http://grafana.home.arpa

```



\---



\# 4. Verify Prometheus Targets



Prometheus should be collecting metrics from configured targets.



From Kubernetes:



```powershell

kubectl get servicemonitors -A

```



Check Prometheus resources:



```powershell

kubectl get prometheus -A

```



Inspect Prometheus:



```powershell

kubectl describe prometheus -n monitoring

```



Check logs if required:



```powershell

kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -c prometheus

```



\---



\# 5. Blackbox Monitoring



Blackbox Exporter is used to perform active probes against services.



It can monitor:



\* HTTP

\* HTTPS

\* ICMP

\* TCP endpoints



Example monitored services:



```text

homeassistant.home.arpa

grafana.home.arpa

vaultwarden.home.arpa

forgejo.home.arpa

uptime.home.arpa

```



Check probes:



```powershell

kubectl get probes -A

```



Describe the probe:



```powershell

kubectl describe probe homelab-app-uptime -n monitoring

```



\---



\# 6. Uptime Kuma Monitoring



Uptime Kuma is used as a second availability-monitoring layer.



Typical monitor types:



```text

HTTP

HTTPS

Ping

TCP port

DNS

```



Example monitored systems:



```text

OPNsense

Pi-hole

Proxmox

Home Assistant

Grafana

Vaultwarden

Forgejo

Traefik

```



This provides a simple NOC-style view of whether critical services are reachable.



\---



\# 7. Basic Network Troubleshooting Workflow



When an alert occurs, troubleshooting starts with simple connectivity checks.



Example:



```text

Alert received

&#x20;     |

&#x20;     v

Confirm affected service

&#x20;     |

&#x20;     v

Ping endpoint

&#x20;     |

&#x20;     v

Check DNS

&#x20;     |

&#x20;     v

Check route

&#x20;     |

&#x20;     v

Check port

&#x20;     |

&#x20;     v

Check firewall

&#x20;     |

&#x20;     v

Check application/service

&#x20;     |

&#x20;     v

Restore

&#x20;     |

&#x20;     v

Validate

```



\---



\# 8. ICMP Testing



Test local infrastructure:



```powershell

ping 192.168.1.1

```



Test another host:



```powershell

ping 192.168.1.210

```



Continuous ping:



```powershell

ping -t 192.168.1.210

```



This helps detect:



\* Packet loss

\* Latency

\* Unreachable hosts

\* Intermittent failures



\---



\# 9. Trace Network Path



Windows:



```powershell

tracert 8.8.8.8

```



Linux:



```bash

traceroute 8.8.8.8

```



This helps identify where traffic stops along a route.



\---



\# 10. DNS Troubleshooting



Resolve service:



```powershell

nslookup homeassistant.home.arpa

```



Resolve external domain:



```powershell

nslookup google.com

```



PowerShell:



```powershell

Resolve-DnsName homeassistant.home.arpa

```



Test a specific DNS server:



```powershell

nslookup google.com 192.168.1.10

```



Check current DNS:



```powershell

Get-DnsClientServerAddress

```



\---



\# 11. TCP Port Testing



PowerShell:



```powershell

Test-NetConnection homeassistant.home.arpa -Port 80

```



Test HTTPS:



```powershell

Test-NetConnection vaultwarden.home.arpa -Port 443

```



Test SSH:



```powershell

Test-NetConnection 192.168.1.158 -Port 22

```



This helps separate:



```text

Host reachable

vs.

Service reachable

```



\---



\# 12. HTTP Service Testing



Use:



```powershell

curl http://homeassistant.home.arpa

```



Headers only:



```powershell

curl -I http://homeassistant.home.arpa

```



HTTPS:



```powershell

curl -I https://vaultwarden.home.arpa

```



This helps identify:



\* 200 OK

\* 301/302 redirects

\* 404 errors

\* 502 errors

\* TLS errors

\* Connection failures



\---



\# 13. Kubernetes Health Checks



List pods:



```powershell

kubectl get pods -A

```



Find unhealthy pods:



```powershell

kubectl get pods -A | findstr /V Running

```



Check services:



```powershell

kubectl get svc -A

```



Check ingress:



```powershell

kubectl get ingress -A

```



Check nodes:



```powershell

kubectl get nodes

```



Detailed node status:



```powershell

kubectl describe node <node-name>

```



\---



\# 14. Kubernetes Pod Troubleshooting



Describe failing pod:



```powershell

kubectl describe pod <pod-name> -n <namespace>

```



Logs:



```powershell

kubectl logs <pod-name> -n <namespace>

```



Previous logs:



```powershell

kubectl logs <pod-name> -n <namespace> --previous

```



Check deployment:



```powershell

kubectl get deployment -n <namespace>

```



Describe deployment:



```powershell

kubectl describe deployment <deployment-name> -n <namespace>

```



\---



\# 15. Check Kubernetes Endpoints



A service can exist while having no healthy backend.



```powershell

kubectl get endpoints -A

```



Specific service:



```powershell

kubectl get endpoints homeassistant -n homeassistant

```



If the endpoint list is empty, the NOC investigation should continue into pod health and label selection.



\---



\# 16. Ingress Troubleshooting



Check ingress:



```powershell

kubectl get ingress -A

```



Describe:



```powershell

kubectl describe ingress homeassistant -n homeassistant

```



Check Traefik:



```powershell

kubectl get pods -n traefik

```



Logs:



```powershell

kubectl logs -n traefik deployment/traefik

```



\---



\# 17. Linux Service Troubleshooting



Check system services:



```bash

systemctl status <service>

```



Restart:



```bash

sudo systemctl restart <service>

```



Enable:



```bash

sudo systemctl enable <service>

```



Check logs:



```bash

journalctl -u <service>

```



Recent logs:



```bash

journalctl -u <service> -n 50

```



Follow logs:



```bash

journalctl -u <service> -f

```



\---



\# 18. Linux Resource Monitoring



Processes:



```bash

ps aux

```



CPU and memory:



```bash

top

```



Memory:



```bash

free -h

```



Disk:



```bash

df -h

```



Block devices:



```bash

lsblk

```



Disk usage:



```bash

du -sh /\*

```



This helps investigate alerts such as:



```text

High CPU

High memory

Disk almost full

Service unresponsive

```



\---



\# 19. Linux Network Troubleshooting



Interfaces:



```bash

ip addr

```



Routes:



```bash

ip route

```



Open ports:



```bash

ss -tulpn

```



Connectivity:



```bash

ping 192.168.1.1

```



DNS:



```bash

nslookup google.com

```



HTTP:



```bash

curl -I http://example.com

```



\---



\# 20. OPNsense Investigation



When several systems lose connectivity at the same time, the firewall/router becomes a key investigation point.



Typical items to inspect:



```text

Interfaces

Routes

Gateway status

DHCP

DNS

Firewall rules

NAT

ARP

Logs

```



Useful OPNsense locations:



```text

Interfaces > Overview

Interfaces > Diagnostics

System > Gateways > Status

Firewall > Log Files > Live View

Services > DHCP

Diagnostics > Routes

Diagnostics > ARP Table

```



\---



\# 21. Layer 1 Troubleshooting



Physical network failures should be ruled out before deeper troubleshooting.



Check:



```text

Cable connected

Link light present

Switch port active

Correct patch cable

Correct physical port

MoCA/coax link if applicable

Power

Transceiver seated

```



Basic workflow:



```text

No connectivity

&#x20;    |

&#x20;    v

Check cable

&#x20;    |

&#x20;    v

Check link LEDs

&#x20;    |

&#x20;    v

Check switch port

&#x20;    |

&#x20;    v

Replace known-good cable

&#x20;    |

&#x20;    v

Retest

```



\---



\# 22. Layer 2 Troubleshooting



Areas include:



\* VLAN assignment

\* Access ports

\* Trunks

\* MAC learning

\* STP

\* Link aggregation



Cisco-style troubleshooting commands practiced in Packet Tracer:



```text

show interfaces status

show vlan brief

show interfaces trunk

show mac address-table

show spanning-tree

```



\---



\# 23. Layer 3 Troubleshooting



Areas include:



\* IP addressing

\* Subnet masks

\* Default gateway

\* Routing

\* ACLs

\* NAT



Cisco commands:



```text

show ip interface brief

show ip route

show running-config

show access-lists

```



\---



\# 24. Cisco Packet Tracer NOC Lab



A separate logical network was created in Cisco Packet Tracer to practice common NOC network troubleshooting.



Example:



```text

&#x20;                Router

&#x20;                  |

&#x20;                  |

&#x20;             Core Switch

&#x20;                  |

&#x20;      +-----------+-----------+

&#x20;      |           |           |

&#x20;    VLAN10      VLAN20      VLAN30

&#x20;     Users       Servers      IT

```



Example networks:



```text

VLAN10   10.10.10.0/24

VLAN20   10.20.20.0/24

VLAN30   10.30.30.0/24

```



\---



\# 25. Configure Cisco VLANs



Example:



```text

enable

configure terminal



vlan 10

name USERS



vlan 20

name SERVERS



vlan 30

name MANAGEMENT

```



Verify:



```text

show vlan brief

```



\---



\# 26. Configure Access Port



```text

configure terminal



interface gigabitEthernet0/1

switchport mode access

switchport access vlan 10

```



Verify:



```text

show interfaces status

```



\---



\# 27. Configure Trunk



```text

configure terminal



interface gigabitEthernet0/24

switchport mode trunk

```



Verify:



```text

show interfaces trunk

```



\---



\# 28. Configure Router Interface



Example:



```text

configure terminal



interface gigabitEthernet0/0

ip address 10.10.10.1 255.255.255.0

no shutdown

```



Verify:



```text

show ip interface brief

```



\---



\# 29. Configure Static Route



```text

ip route 10.20.20.0 255.255.255.0 10.10.10.2

```



Verify:



```text

show ip route

```



\---



\# 30. Configure OSPF



Example:



```text

router ospf 1



network 10.10.10.0 0.0.0.255 area 0

network 10.20.20.0 0.0.0.255 area 0

```



Verify:



```text

show ip ospf neighbor

```



Routes:



```text

show ip route ospf

```



\---



\# 31. Cisco Interface Troubleshooting



Useful commands:



```text

show interfaces

show interfaces status

show ip interface brief

```



Check errors:



```text

show interfaces counters errors

```



Things to look for:



```text

Administratively down

Line protocol down

CRC errors

Duplex mismatch

Input errors

Output errors

```



\---



\# 32. Monitoring Alert Workflow



Example:



```text

Alert generated

&#x20;    |

&#x20;    v

Acknowledge alert

&#x20;    |

&#x20;    v

Determine affected system

&#x20;    |

&#x20;    v

Determine impact

&#x20;    |

&#x20;    v

Check monitoring history

&#x20;    |

&#x20;    v

Check connectivity

&#x20;    |

&#x20;    v

Check infrastructure

&#x20;    |

&#x20;    v

Identify root cause

&#x20;    |

&#x20;    v

Restore service

&#x20;    |

&#x20;    v

Validate

&#x20;    |

&#x20;    v

Document

```



\---



\# 33. Incident Scenario — Website Down



\## Alert



```text

CRITICAL

homeassistant.home.arpa unreachable

```



\## Initial Check



```powershell

ping homeassistant.home.arpa

```



Check DNS:



```powershell

nslookup homeassistant.home.arpa

```



Check HTTP:



```powershell

curl -I http://homeassistant.home.arpa

```



Check ingress:



```powershell

kubectl get ingress -n homeassistant

```



Check pod:



```powershell

kubectl get pods -n homeassistant

```



Check service:



```powershell

kubectl get svc -n homeassistant

```



Check endpoints:



```powershell

kubectl get endpoints -n homeassistant

```



\---



\# 34. Incident Scenario — 502 Bad Gateway



\## Alert



Application monitor reports:



```text

502 Bad Gateway

```



\## Investigation



Check ingress:



```powershell

kubectl get ingress -A

```



Check Traefik:



```powershell

kubectl get pods -n traefik

```



Check service:



```powershell

kubectl get svc -n homeassistant

```



Check endpoints:



```powershell

kubectl get endpoints homeassistant -n homeassistant

```



Check application pod:



```powershell

kubectl get pods -n homeassistant

```



Logs:



```powershell

kubectl logs <homeassistant-pod> -n homeassistant

```



Possible failure chain:



```text

Traefik

&#x20;  |

Service

&#x20;  |

Endpoints

&#x20;  |

Pod

```



\---



\# 35. Incident Scenario — DNS Failure



\## Alert



Multiple services become unreachable by hostname.



Test IP directly:



```powershell

ping 192.168.1.240

```



Then DNS:



```powershell

nslookup homeassistant.home.arpa

```



Check Pi-hole:



```powershell

ping 192.168.1.10

```



Query Pi-hole directly:



```powershell

nslookup homeassistant.home.arpa 192.168.1.10

```



Possible root cause:



```text

Application healthy

Network healthy

DNS failed

```



This prevents wasting time restarting working applications.



\---



\# 36. Incident Scenario — High CPU



\## Alert



```text

CPU usage > 90%

```



Linux investigation:



```bash

top

```



Or:



```bash

ps aux --sort=-%cpu | head

```



Kubernetes:



```powershell

kubectl top pods -A

```



Nodes:



```powershell

kubectl top nodes

```



Check logs if one process or service is responsible.



\---



\# 37. Incident Scenario — High Memory



Linux:



```bash

free -h

```



Processes:



```bash

ps aux --sort=-%mem | head

```



Kubernetes:



```powershell

kubectl top pods -A

```



Determine whether memory usage is:



```text

Expected

Increasing

Leaking

Causing OOM kills

```



Check pod events:



```powershell

kubectl describe pod <pod-name> -n <namespace>

```



\---



\# 38. Incident Scenario — Disk Full



Check filesystem:



```bash

df -h

```



Identify large directories:



```bash

du -sh /\* 2>/dev/null

```



More detailed:



```bash

du -ah /var | sort -rh | head

```



Check logs:



```bash

du -sh /var/log/\*

```



Possible remediation includes:



```text

Rotate logs

Delete safe temporary data

Expand storage

Escalate storage issue

```



\---



\# 39. Incident Scenario — Kubernetes Pod CrashLoopBackOff



Check:



```powershell

kubectl get pods -A

```



Describe:



```powershell

kubectl describe pod <pod-name> -n <namespace>

```



Current logs:



```powershell

kubectl logs <pod-name> -n <namespace>

```



Previous crash:



```powershell

kubectl logs <pod-name> -n <namespace> --previous

```



Check:



```text

Environment variables

Secrets

ConfigMaps

Storage

Dependencies

Image

Recent changes

```



\---



\# 40. Incident Scenario — Kubernetes Pod Pending



Describe:



```powershell

kubectl describe pod <pod-name> -n <namespace>

```



Look for:



```text

Insufficient CPU

Insufficient memory

PVC issue

Node selector

Taint

Scheduling issue

```



Check nodes:



```powershell

kubectl get nodes

```



PVCs:



```powershell

kubectl get pvc -A

```



\---



\# 41. Incident Scenario — Storage Failure



Check PVC:



```powershell

kubectl get pvc -A

```



Check Longhorn pods:



```powershell

kubectl get pods -n longhorn-system

```



Check volumes:



```powershell

kubectl get volumes.longhorn.io -n longhorn-system

```



Check volume attachments:



```powershell

kubectl get volumeattachments.longhorn.io -n longhorn-system

```



Investigate:



```powershell

kubectl describe volumeattachments.longhorn.io <volume> -n longhorn-system

```



\---



\# 42. Incident Scenario — Node Down



Check:



```powershell

kubectl get nodes

```



If a node shows:



```text

NotReady

```



Describe:



```powershell

kubectl describe node <node>

```



Check which workloads were running there:



```powershell

kubectl get pods -A -o wide

```



Determine:



```text

Node powered off?

VM problem?

Network issue?

Talos problem?

Storage issue?

```



\---



\# 43. Incident Scenario — Service Port Unreachable



Test:



```powershell

Test-NetConnection <host> -Port <port>

```



Example:



```powershell

Test-NetConnection 192.168.1.158 -Port 8006

```



If ping works but port fails:



```text

Host reachable

&#x20;    |

Service/firewall issue likely

```



Investigate:



```text

Application status

Listening socket

Firewall

NAT

ACL

Service configuration

```



\---



\# 44. Incident Scenario — Packet Loss



Windows:



```powershell

ping -n 100 192.168.1.1

```



Linux:



```bash

ping -c 100 192.168.1.1

```



Trace:



```powershell

tracert 8.8.8.8

```



Investigate:



```text

Cable

Switch port

MoCA link

Wireless link

Duplex

Interface errors

Router

ISP

```



\---



\# 45. SNMP Lab



SNMP can be used to monitor network devices.



Key concepts:



```text

SNMP Manager

SNMP Agent

OID

Polling

Trap

Community string

SNMPv2c

SNMPv3

```



Example Linux package installation:



```bash

sudo apt update

sudo apt install snmp snmpd

```



Test local SNMP:



```bash

snmpwalk -v2c -c public localhost

```



Example query:



```bash

snmpget -v2c -c public <device-ip> 1.3.6.1.2.1.1.1.0

```



For real environments, SNMPv3 should be preferred when supported.



\---



\# 46. Syslog Lab



Centralized logging is useful for correlating incidents.



Linux service logs:



```bash

journalctl

```



Kernel logs:



```bash

dmesg

```



Follow:



```bash

journalctl -f

```



Network-device syslog concepts include forwarding:



```text

Interface changes

Authentication events

Routing events

Hardware warnings

Configuration changes

```



to a central log server.



\---



\# 47. Alert Severity Model



Example severity levels:



| Severity      | Example                                       |

| ------------- | --------------------------------------------- |

| P1 / Critical | Core network or many services unavailable     |

| P2 / High     | Important service unavailable                 |

| P3 / Medium   | Degraded performance or single-system failure |

| P4 / Low      | Informational or minor issue                  |



\---



\# 48. Incident Ticket Example



```text

Incident ID:

NOC-001



Severity:

P2



Alert:

Home Assistant endpoint unavailable



Affected Service:

homeassistant.home.arpa



Impact:

Home automation application unavailable to users



Detection:

Blackbox HTTP probe failure



Initial Checks:

\- DNS resolution successful

\- Traefik reachable

\- Service object present

\- Endpoint unavailable



Investigation:

kubectl get pods -n homeassistant

kubectl get svc -n homeassistant

kubectl get endpoints -n homeassistant

kubectl logs <pod> -n homeassistant



Root Cause:

Application pod was unavailable, leaving the service without a healthy backend.



Resolution:

Application workload was restored and endpoint health returned.



Validation:

\- Pod Running

\- Endpoint populated

\- HTTP probe successful

\- Application reachable



Status:

Resolved

```



\---



\# 49. Escalation Example



```text

Incident:

NOC-004



Severity:

P1



Affected Systems:

Multiple internal services



Symptoms:

\- High packet loss

\- Intermittent gateway connectivity

\- Multiple monitoring alerts



Troubleshooting Completed:

\- Confirmed issue affects multiple endpoints

\- Verified DNS not primary cause

\- Confirmed packet loss to gateway

\- Checked local endpoint NIC

\- Checked switch connection

\- Replaced patch cable

\- Issue persisted



Evidence:

Ping loss and timestamps documented.



Escalation:

Escalated for network infrastructure investigation.

```



\---



\# 50. NOC Shift Workflow



A simplified NOC workflow:



```text

Start shift

&#x20;  |

&#x20;  v

Review active alerts

&#x20;  |

&#x20;  v

Review open incidents

&#x20;  |

&#x20;  v

Check critical dashboards

&#x20;  |

&#x20;  v

Prioritize P1/P2 issues

&#x20;  |

&#x20;  v

Investigate

&#x20;  |

&#x20;  v

Resolve or escalate

&#x20;  |

&#x20;  v

Update ticket

&#x20;  |

&#x20;  v

Validate monitoring recovery

&#x20;  |

&#x20;  v

Document handoff

```



\---



\# 51. NOC Handoff Notes



Example:



```text

Shift Handoff



Open Incident:

NOC-007



Service:

Grafana



Current Status:

Application reachable but showing intermittent latency.



Work Completed:

\- Confirmed DNS healthy

\- Confirmed ingress healthy

\- Checked Grafana pod

\- Reviewed node CPU/memory

\- No restart performed



Next Steps:

Review Prometheus query latency and storage performance.



Priority:

P3

```



\---



\# 52. Useful NOC Command Reference



\## Windows



```powershell

ping

tracert

nslookup

ipconfig /all

route print

arp -a

netstat -ano

Test-NetConnection

Resolve-DnsName

```



\## Linux



```bash

ping

traceroute

ip addr

ip route

ss -tulpn

curl

nslookup

dig

ps aux

top

free -h

df -h

journalctl

systemctl

dmesg

```



\## Kubernetes



```powershell

kubectl get nodes

kubectl get pods -A

kubectl get svc -A

kubectl get ingress -A

kubectl get endpoints -A

kubectl get pvc -A

kubectl describe

kubectl logs

kubectl top nodes

kubectl top pods -A

```



\## Flux



```powershell

flux get kustomizations -A

flux get helmreleases -A

```



\## Cisco



```text

show ip interface brief

show interfaces

show interfaces status

show interfaces trunk

show vlan brief

show mac address-table

show spanning-tree

show ip route

show access-lists

show running-config

show cdp neighbors

```



\---



\# 53. Skills Demonstrated



\## Network Operations



\* Network availability monitoring

\* Layer 1/2/3 troubleshooting

\* TCP/IP

\* DNS

\* DHCP

\* VLANs

\* Routing

\* NAT

\* Firewall troubleshooting

\* ICMP

\* TCP port testing



\## Monitoring



\* Prometheus

\* Grafana

\* Alertmanager

\* Blackbox Exporter

\* Uptime Kuma

\* Availability checks

\* Infrastructure metrics

\* Alert triage



\## Systems



\* Linux

\* Proxmox

\* Talos Linux

\* Kubernetes

\* OPNsense

\* Pi-hole

\* Longhorn

\* Traefik



\## Networking Practice



\* Cisco IOS concepts

\* VLAN configuration

\* Trunks

\* STP

\* Static routing

\* OSPF

\* Interface troubleshooting

\* MAC tables



\## Operations



\* Incident prioritization

\* Root cause investigation

\* Escalation

\* Ticket documentation

\* Shift handoff

\* Recovery validation

\* Change awareness



\---



\# 54. Troubleshooting Methodology



The NOC lab uses a layered troubleshooting approach.



```text

Alert

&#x20; |

&#x20; v

Scope

&#x20; |

&#x20; v

Layer 1

Physical connectivity

&#x20; |

&#x20; v

Layer 2

VLAN / switch / MAC

&#x20; |

&#x20; v

Layer 3

IP / gateway / routing

&#x20; |

&#x20; v

DNS

&#x20; |

&#x20; v

Port / Firewall

&#x20; |

&#x20; v

Operating System

&#x20; |

&#x20; v

Application

&#x20; |

&#x20; v

Restore

&#x20; |

&#x20; v

Validate

&#x20; |

&#x20; v

Document

```



This avoids immediately restarting systems before identifying the actual failure domain.



\---



\# 55. Resume-Relevant Experience



This project provides practical experience applicable to:



\* NOC Technician

\* Network Operations Technician

\* Infrastructure Operations Technician

\* Data Center Operations Technician

\* Junior Network Technician

\* Junior Linux Administrator

\* Production Support Analyst

\* IT Operations Technician



Example resume bullet:



> Built and operated a NOC-style monitoring lab across a Proxmox and Kubernetes homelab using Prometheus, Grafana, Alertmanager, Blackbox Exporter, and Uptime Kuma; investigated simulated incidents involving DNS, routing, packet loss, firewall rules, Linux services, Kubernetes workloads, storage, and application availability.



Another possible bullet:



> Practiced end-to-end network operations workflows including alert triage, Layer 1/2/3 troubleshooting, TCP port validation, DNS analysis, Linux diagnostics, Kubernetes health checks, incident prioritization, escalation, recovery verification, and shift-style documentation.



\---



\# Future Improvements



Potential additions include:



\* Zabbix

\* LibreNMS

\* PRTG

\* SNMPv3

\* Centralized Syslog

\* Loki

\* Splunk

\* NetFlow

\* sFlow

\* Cisco physical switches

\* BGP

\* Advanced OSPF

\* LACP

\* STP failure scenarios

\* Interface utilization monitoring

\* Automated alert notifications

\* PagerDuty-style workflows

\* SLA tracking

\* MTTR tracking

\* Incident dashboards



\---



\# Project Status



```text

Monitoring Stack                 Complete

Prometheus                       Complete

Grafana                          Complete

Alertmanager                     Complete

Blackbox Exporter                Complete

Uptime Kuma                      Complete

Linux Monitoring                 Complete

Kubernetes Monitoring            Complete

DNS Troubleshooting              Complete

Layer 1 Troubleshooting          Complete

Layer 2 Practice                 Complete

Layer 3 Practice                 Complete

Cisco Packet Tracer Lab          Complete

Incident Scenarios               Complete

Ticket Documentation             Complete

Escalation Workflow              Complete

Shift Handoff Practice           Complete

```



\---



\# Conclusion



The NOC Technician Lab was designed to simulate the operational workflow of an entry-level Network Operations Center.



Rather than only practicing individual networking commands, the lab combines monitoring, alerts, Linux, networking, Kubernetes, DNS, routing, firewalls, applications, and documentation into full incident-response workflows.



The project demonstrates the ability to move from:



```text

Alert

&#x20;  ↓

Investigation

&#x20;  ↓

Failure-domain isolation

&#x20;  ↓

Resolution

&#x20;  ↓

Validation

&#x20;  ↓

Documentation

```



and provides a repeatable environment for practicing NOC, network operations, infrastructure support, and production-support skills.



