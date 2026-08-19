\# Automated Proxmox Cluster Updates with Ansible



\## Overview



I built an automated update workflow for my three-node Proxmox VE cluster using a Raspberry Pi as a dedicated Ansible control node.



The goal was to move away from manually logging into each Proxmox node and running package updates individually. Instead, the Raspberry Pi now connects to each node over SSH, performs health checks, updates the nodes sequentially, and can be scheduled to run automatically every two weeks.



Reboots are intentionally \*\*not automated\*\*. Package updates and host reboots are treated as separate maintenance operations so I can control when workloads such as OPNsense and Kubernetes are taken offline.



\---



\## Environment



\### Proxmox Cluster



My Proxmox cluster contains three nodes:



```text

MainCluster

├── pve

├── pve02

└── pve03

```



The cluster uses Corosync for clustering and has three votes:



```text

Expected votes: 3

Total votes:    3

Quorum:         2

Quorate:        Yes

```



The nodes were originally running:



```text

Proxmox VE: 8.4.0

Kernel:     6.8.12-9-pve

```



One of the nodes later updated to:



```text

pve-manager/8.4.20

```



\### Workload Distribution



The hosts are not equally critical.



```text

pve

├── OPNsense VM

└── Pi-hole LXC



pve02

├── Talos Kubernetes control plane

├── Talos worker 1

├── Talos worker 2

├── Talos worker 3

├── Tailscale

└── Cloudflare Tunnel



pve03

├── Game Server

└── Test VM

```



Because `pve` hosts my router and `pve02` hosts my Kubernetes cluster, I decided that automatic reboots would be too risky.



\---



\# Architecture



The Raspberry Pi acts as the Ansible control node.



```text

&#x20;                  Raspberry Pi

&#x20;                       │

&#x20;                    Ansible

&#x20;                       │

&#x20;                SSH key authentication

&#x20;                       │

&#x20;           ┌───────────┼───────────┐

&#x20;           │           │           │

&#x20;           ▼           ▼           ▼

&#x20;          pve        pve02        pve03

```



The Raspberry Pi runs Debian 13:



```text

Debian GNU/Linux 13 (trixie)

Python 3.13

```



Ansible communicates with the Proxmox servers over SSH.



\---



\# Installing Ansible



I installed Ansible on the Raspberry Pi using `pipx`.



```bash

sudo apt update

sudo apt install -y openssh-client pipx

pipx ensurepath

```



Then:



```bash

pipx install --include-deps ansible

```



I verified the installation using:



```bash

ansible --version

```



\---



\# Configuring SSH Authentication



To allow unattended Ansible jobs, I created an ED25519 SSH key on the Raspberry Pi.



```bash

ssh-keygen -t ed25519 -C "ansible-proxmox"

```



The public key was then copied to each Proxmox node:



```bash

ssh-copy-id root@<pve>

ssh-copy-id root@<pve02>

ssh-copy-id root@<pve03>

```



Passwordless SSH was verified with:



```bash

ssh root@<node> hostname

```



\---



\# Ansible Inventory



The Ansible project is stored at:



```text

/home/qiandi/ansible-proxmox/

```



The inventory contains the three Proxmox nodes.



```ini

\[proxmox]

pve ansible\_host=<PVE\_IP>

pve02 ansible\_host=<PVE02\_IP>

pve03 ansible\_host=<PVE03\_IP>



\[proxmox:vars]

ansible\_user=root

ansible\_python\_interpreter=/usr/bin/python3.11

```



Connectivity was verified using:



```bash

ansible proxmox -i inventory.ini -m ping

```



All three nodes returned:



```text

SUCCESS

ping: pong

```



\---



\# Building a Proxmox Health Check



Before automating updates, I created a read-only health-check playbook.



The playbook checks:



\* Proxmox version

\* Cluster quorum

\* Storage status

\* Running VMs

\* Running LXC containers

\* System uptime

\* Root filesystem usage

\* Available package updates



Example command:



```bash

ansible-playbook -i inventory.ini proxmox-check.yml

```



The health check completed against all three nodes with:



```text

changed=0

unreachable=0

failed=0

```



This gave me a safe way to validate cluster health before allowing Ansible to make changes.



\---



\# Repository Issue



The first real update attempt failed during:



```text

Refresh APT package cache

```



The Proxmox nodes returned:



```text

401 Unauthorized

```



for:



```text

enterprise.proxmox.com/debian/pve

enterprise.proxmox.com/debian/ceph-quincy

```



The servers were configured to use the Proxmox Enterprise repositories even though the cluster does not have an enterprise subscription.



All three nodes contained:



```text

/etc/apt/sources.list.d/pve-enterprise.list

/etc/apt/sources.list.d/ceph.list

```



The Ceph repository was also unnecessary because the cluster does not use Ceph.



Running:



```bash

pveceph status

```



returned:



```text

binary not installed: /usr/bin/ceph-mon

```



\---



\# Automating Repository Configuration



Instead of manually fixing each node, I created another Ansible playbook to standardize the repository configuration across the entire cluster.



The playbook:



1\. Removes the Enterprise Proxmox repository.

2\. Removes the unused Ceph Enterprise repository.

3\. Adds the Proxmox `pve-no-subscription` repository.

4\. Refreshes the APT package cache.



Example:



```yaml

\---

\- name: Configure Proxmox repositories

&#x20; hosts: proxmox

&#x20; serial: 1

&#x20; gather\_facts: false



&#x20; tasks:

&#x20;   - name: Disable Proxmox enterprise repository

&#x20;     ansible.builtin.file:

&#x20;       path: /etc/apt/sources.list.d/pve-enterprise.list

&#x20;       state: absent



&#x20;   - name: Disable unused Ceph enterprise repository

&#x20;     ansible.builtin.file:

&#x20;       path: /etc/apt/sources.list.d/ceph.list

&#x20;       state: absent



&#x20;   - name: Configure Proxmox no-subscription repository

&#x20;     ansible.builtin.copy:

&#x20;       dest: /etc/apt/sources.list.d/pve-no-subscription.list

&#x20;       owner: root

&#x20;       group: root

&#x20;       mode: "0644"

&#x20;       content: |

&#x20;         deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription



&#x20;   - name: Refresh APT package cache

&#x20;     ansible.builtin.apt:

&#x20;       update\_cache: true

&#x20;       lock\_timeout: 300

```



After running the playbook, all three nodes used the correct repository.



\---



\# Handling a Stuck Proxmox APT Task



While configuring the repositories, `pve03` repeatedly failed because another process was holding:



```text

/var/lib/apt/lists/lock

```



Inspection showed:



```text

apt-get update

```



had been running for several hours.



I traced the process tree:



```text

Proxmox task

└── apt-get update

&#x20;   ├── https

&#x20;   ├── http

&#x20;   ├── gpgv

&#x20;   └── store

```



The parent was a Proxmox task identified by a UPID:



```text

UPID:pve03:...:aptupdate::root@pam:

```



Instead of deleting the APT lock file, I safely stopped the stuck task through the Proxmox API using `pvesh`.



```bash

pvesh delete "/nodes/pve03/tasks/<UPID>"

```



Afterward:



```bash

apt-get update

```



completed normally.



This was an important lesson: \*\*APT lock files should not simply be deleted when another package process is still running.\*\*



The process using the lock should be identified first.



\---



\# Proxmox Update Playbook



The final update playbook updates the cluster sequentially.



```yaml

\---

\- name: Update Proxmox cluster packages

&#x20; hosts: proxmox

&#x20; serial: 1

&#x20; gather\_facts: true



&#x20; tasks:

&#x20;   - name: Check cluster quorum

&#x20;     ansible.builtin.command: pvecm status

&#x20;     register: quorum

&#x20;     changed\_when: false

&#x20;     check\_mode: false



&#x20;   - name: Stop if cluster is not quorate

&#x20;     ansible.builtin.fail:

&#x20;       msg: "Cluster is not quorate. Aborting updates."

&#x20;     when: "'Quorate:          Yes' not in quorum.stdout"



&#x20;   - name: Check storage status

&#x20;     ansible.builtin.command: pvesm status

&#x20;     register: storage

&#x20;     changed\_when: false

&#x20;     check\_mode: false



&#x20;   - name: Refresh APT package cache

&#x20;     ansible.builtin.apt:

&#x20;       update\_cache: true

&#x20;       lock\_timeout: 300



&#x20;   - name: Perform full package upgrade

&#x20;     ansible.builtin.apt:

&#x20;       upgrade: full

&#x20;       lock\_timeout: 300

&#x20;     register: upgrade\_result



&#x20;   - name: Check whether reboot is required

&#x20;     ansible.builtin.stat:

&#x20;       path: /var/run/reboot-required

&#x20;     register: reboot\_required



&#x20;   - name: Get current Proxmox version

&#x20;     ansible.builtin.command: pveversion

&#x20;     register: pve\_version

&#x20;     changed\_when: false



&#x20;   - name: Display update result

&#x20;     ansible.builtin.debug:

&#x20;       msg:

&#x20;         - "===== {{ inventory\_hostname }} ====="

&#x20;         - "Proxmox: {{ pve\_version.stdout }}"

&#x20;         - "Packages changed: {{ upgrade\_result.changed }}"

&#x20;         - "Reboot required: {{ reboot\_required.stat.exists }}"

```



The important setting is:



```yaml

serial: 1

```



This prevents Ansible from modifying all three Proxmox nodes simultaneously.



The update sequence becomes:



```text

pve

&#x20;↓

update complete

&#x20;↓

pve02

&#x20;↓

update complete

&#x20;↓

pve03

&#x20;↓

update complete

```



If one node encounters an error, the playbook stops instead of continuing blindly.



\---



\# Separating Updates from Reboots



One design decision was to \*\*not reboot Proxmox nodes automatically\*\*.



The update playbook only:



```text

Check health

&#x20;    ↓

Refresh packages

&#x20;    ↓

Install updates

&#x20;    ↓

Determine if reboot is required

&#x20;    ↓

STOP

```



It does not contain an Ansible reboot task.



This is intentional because:



```text

pve reboot

&#x20;   ↓

OPNsense offline

&#x20;   ↓

Router offline

```



and:



```text

pve02 reboot

&#x20;    ↓

All Talos VMs offline

&#x20;    ↓

Kubernetes cluster offline

```



Host reboots therefore remain a separate maintenance procedure.



\---



\# Handling APT Locks



Another issue appeared when `pve02` already had an `apt-get` process running.



Ansible waited for:



```text

/var/lib/dpkg/lock-frontend

```



but eventually timed out.



To reduce failures from short-lived package jobs, the playbook uses:



```yaml

lock\_timeout: 300

```



This allows Ansible to wait up to five minutes for APT or dpkg locks to clear.



Before manually intervening, I check for package processes with:



```bash

pgrep -af 'apt|dpkg'

```



If no real APT or dpkg process remains, the update can safely be retried.



\---



\# Scheduling Updates with systemd



Once the Ansible playbook worked reliably, I moved the update execution to the Raspberry Pi's `systemd`.



This means my desktop does not need to be powered on for maintenance to run.



\## systemd Service



File:



```text

/etc/systemd/system/proxmox-update.service

```



Configuration:



```ini

\[Unit]

Description=Update Proxmox Cluster with Ansible

After=network-online.target

Wants=network-online.target



\[Service]

Type=oneshot

User=qiandi

WorkingDirectory=/home/qiandi/ansible-proxmox

ExecStart=/home/qiandi/.local/bin/ansible-playbook -i inventory.ini proxmox-update.yml

```



The service executes the same Ansible command I previously ran manually.



\---



\# systemd Timer



File:



```text

/etc/systemd/system/proxmox-update.timer

```



Configuration:



```ini

\[Unit]

Description=Update Proxmox Cluster Every Two Weeks



\[Timer]

OnUnitActiveSec=2w

Unit=proxmox-update.service



\[Install]

WantedBy=timers.target

```



The timer is enabled with:



```bash

sudo systemctl daemon-reload

sudo systemctl enable --now proxmox-update.timer

```



The schedule can be viewed using:



```bash

systemctl list-timers --all | grep proxmox

```



Logs from the Ansible update can be viewed with:



```bash

journalctl -u proxmox-update.service

```



or followed live with:



```bash

journalctl -u proxmox-update.service -f

```



\---



\# Final Workflow



The completed automation looks like this:



```text

&#x20;                   Raspberry Pi

&#x20;                        │

&#x20;                 systemd timer

&#x20;                        │

&#x20;                    Every 2 weeks

&#x20;                        │

&#x20;                        ▼

&#x20;             proxmox-update.service

&#x20;                        │

&#x20;                        ▼

&#x20;                  Ansible Playbook

&#x20;                        │

&#x20;                        ▼

&#x20;                Check Cluster Quorum

&#x20;                        │

&#x20;                        ▼

&#x20;                 Check Storage

&#x20;                        │

&#x20;                        ▼

&#x20;              Update Proxmox Nodes

&#x20;                        │

&#x20;            ┌───────────┼───────────┐

&#x20;            ▼           ▼           ▼

&#x20;           pve        pve02        pve03

&#x20;            │           │           │

&#x20;            └───────────┴───────────┘

&#x20;                        │

&#x20;                        ▼

&#x20;               Report Update Status

&#x20;                        │

&#x20;                        ▼

&#x20;             Report Reboot Requirement

&#x20;                        │

&#x20;                        ▼

&#x20;                    NO REBOOT

```



\---



\# What I Learned



This project gave me hands-on experience with several infrastructure and operations concepts:



\* Ansible inventory management

\* SSH key-based authentication

\* Ansible playbooks

\* Idempotent infrastructure automation

\* Sequential cluster maintenance with `serial`

\* Debian APT package management

\* Troubleshooting APT and dpkg locks

\* Proxmox task management

\* Proxmox API interaction with `pvesh`

\* Proxmox cluster quorum

\* Repository management

\* Linux systemd services

\* systemd timers

\* Infrastructure health checks

\* Automated maintenance workflows

\* Failure handling and safe automation



One of the biggest lessons was that infrastructure automation should not simply execute commands automatically. A good automation workflow needs to understand when it is \*\*not safe to continue\*\*.



For example, the playbook verifies cluster quorum before performing updates and stops if a node encounters an error.



\---



\# Future Improvements



There are several improvements I plan to make to this automation.



\### Better APT Lock Detection



Instead of waiting until the update task encounters a lock, the playbook could perform a pre-flight check for:



```text

apt

apt-get

dpkg

```



and wait or skip the affected node.



\### Notifications



The Raspberry Pi could send a notification after each maintenance run containing:



```text

pve    - Updated / Current

pve02  - Updated / Current

pve03  - Updated / Current



Reboot required:

pve    - No

pve02  - Yes

pve03  - No

```



Potential notification methods include:



\* Discord webhook

\* Slack

\* Email

\* Home Assistant notification

\* ntfy



\### Controlled Reboot Automation



A separate playbook could eventually perform rolling reboots.



The reboot workflow would remain separate from package updates:



```text

Verify cluster

&#x20;     ↓

Reboot one node

&#x20;     ↓

Wait for node

&#x20;     ↓

Verify services

&#x20;     ↓

Continue

```



However, `pve` and `pve02` would require additional safeguards because they host critical networking and Kubernetes workloads.



\### Better Workload Distribution



Currently all Talos Kubernetes VMs are hosted on `pve02`.



Distributing Kubernetes nodes across multiple Proxmox hosts would improve resilience and allow safer rolling Proxmox maintenance.



\---



\## Result



The Proxmox cluster can now be maintained using a single automation workflow instead of updating each node manually.



What originally required:



```text

SSH into pve

Update packages



SSH into pve02

Update packages



SSH into pve03

Update packages

```



is now:



```text

systemd

&#x20;  ↓

Ansible

&#x20;  ↓

Automated Proxmox cluster maintenance

```



The Raspberry Pi acts as a lightweight infrastructure automation server, allowing Proxmox package maintenance to run consistently every two weeks while preserving manual control over disruptive host reboots.



