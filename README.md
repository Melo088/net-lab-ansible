# net-lab-ansible

Idempotent Ansible playbook suite to deploy a complete network infrastructure lab on KVM/Fedora:
**DNS + DHCP (BIND/dhcpd)** and a full **Kubernetes 1.32 cluster** (kubeadm + Flannel CNI) on Rocky Linux 9.

---

## Architecture

```
192.168.50.0/24  (lab1-ansible-net / KVM NAT)
│
├── 192.168.50.1    virbr50 — KVM gateway (libvirt)
├── 192.168.50.10   ns1.jmelol.lab      ← Bastion: DNS + DHCP + kubectl  [active]
├── 192.168.50.20   master.jmelol.lab   ← Kubernetes control plane        [active]
├── 192.168.50.21   worker.jmelol.lab   ← Kubernetes worker / workloads   [active]
└── 192.168.50.100  client01.jmelol.lab ← Test DHCP client (fixed MAC)    [optional]
     (dynamic pool: 192.168.50.150–200)
```

---

## Development Environment

| Component    | Details                                          |
|---|---|
| Host OS      | Fedora 43 (KDE Plasma) — kernel 6.19.7           |
| Hardware     | Lenovo IdeaPad Slim 3 15AMN8                     |
| Hypervisor   | KVM/QEMU + libvirt (`dnf install @virtualization`)|
| Guest OS     | Rocky Linux 9.7 (Blue Onyx) Minimal              |
| Kubernetes   | 1.32.13                                          |
| Runtime      | containerd 2.2.2                                 |
| CNI          | Flannel (latest)                                 |
| Ansible      | Installed in an isolated `.venv`                 |

---

## Repository Structure

```
lab1-ansible-dns-dhcp/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── site.yml
├── roles/
│   ├── dns_bind/               ← BIND (named) on ns1
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── named.conf.j2
│   │       ├── zone.forward.j2
│   │       └── zone.reverse.j2
│   ├── dhcpd/                  ← ISC dhcpd on ns1
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       └── dhcpd.conf.j2
│   ├── k8s_base/               ← Common k8s prerequisites (all nodes)
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   └── tasks/main.yml
│   ├── k8s_master/             ← kubeadm init + Flannel CNI
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── k8s_worker/             ← kubeadm join
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   └── k8s_kubectl_bastion/    ← kubectl + kubeconfig on ns1
│       ├── defaults/main.yml
│       └── tasks/main.yml
├── startup-guide.txt           ← Lab1 (DNS/DHCP) startup & verification
├── startup-guide-k8s.txt       ← Lab2 (Kubernetes) startup & verification
├── .gitignore
├── .github/workflows/ci.yml
└── README.md
```

---

## Playbook Plays (site.yml)

| # | Play | Hosts | What it does |
|---|---|---|---|
| 1 | Configure DNS & DHCP | `dns_dhcp_servers` | Deploys BIND zones and dhcpd reservations |
| 2 | Kubernetes base | `k8s_master:k8s_workers` | kernel modules, swap off, containerd, kubelet/kubeadm/kubectl |
| 3 | Control plane init | `k8s_master` | `kubeadm init`, kubeconfig, Flannel CNI, join token |
| 4 | Worker join | `k8s_workers` | `kubeadm join` using token from play 3 |
| 5 | Bastion kubectl | `dns_dhcp_servers` | installs kubectl + kubeconfig on ns1 |

---

## Quick Start

### Prerequisites

```bash
# Fedora host — install virtualization stack
sudo dnf install @virtualization virt-manager

# Python venv for Ansible (created one level above the repo)
cd ~/University/InfraIII/lab1
python3 -m venv .venv
source .venv/bin/activate
pip install ansible ansible-lint
ansible-galaxy collection install ansible.posix community.general
```

### VM Setup (one-time)

```bash
# Clone base image for each node
sudo virsh shutdown rocky9-base
sudo virt-clone --original rocky9-base --name k8s-master --auto-clone
sudo virt-clone --original rocky9-base --name k8s-worker  --auto-clone

# Get real MAC addresses and update group_vars/all.yml dhcp_reservations
sudo virsh domiflist k8s-master
sudo virsh domiflist k8s-worker

# Copy SSH key to all nodes
ssh-copy-id -i ~/.ssh/ansible_lab.pub rocky@192.168.50.10
ssh-copy-id -i ~/.ssh/ansible_lab.pub rocky@192.168.50.20
ssh-copy-id -i ~/.ssh/ansible_lab.pub rocky@192.168.50.21
```

### Run

```bash
cd ~/University/InfraIII/lab1/lab1-ansible-dns-dhcp
source ../.venv/bin/activate

ansible all -m ping                        # verify connectivity
ansible-playbook site.yml                  # deploy everything
```

Expected first-run result: **~38 tasks on master, ~28 on worker, 0 failed.**  
Subsequent runs are fully idempotent (0 changes on stable state).

---

## Key Variables (`group_vars/all.yml`)

| Variable | Default | Description |
|---|---|---|
| `lab_domain` | `jmelol.lab` | DNS zone name |
| `lab_network` | `192.168.50.0` | Lab subnet |
| `dns_server_ip` | `192.168.50.10` | ns1 static IP |
| `k8s_master_ip` | `192.168.50.20` | Control plane advertise address |
| `k8s_pod_cidr` | `10.244.0.0/16` | Flannel pod network CIDR |
| `k8s_version_major_minor` | `1.32` | Kubernetes repo version |
| `dhcp_range_start` | `192.168.50.150` | Dynamic pool start (above reservations) |

---

## Verification

```bash
# From ns1 (bastion)
ssh -i ~/.ssh/ansible_lab rocky@192.168.50.10

kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get pods -n kube-flannel -o wide

# Quick smoke test
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl expose deployment nginx-test --type=NodePort --port=80
kubectl get svc nginx-test          # note the NodePort (3XXXX)
curl http://192.168.50.21:3XXXX     # should return nginx welcome page
kubectl delete deployment nginx-test && kubectl delete svc nginx-test
```

---

## Issues Encountered and Resolutions

### SSH Permission Denied
Key was installed on the provisional IP (`.50`) but not on the final IP (`.10`) after reconfiguration.  
**Resolution:** Re-run `ssh-copy-id` targeting the definitive IP.

### Privilege Escalation Failure
Ansible required an interactive sudo password, incompatible with unattended automation.  
**Resolution:** Create `/etc/sudoers.d/rocky` with `NOPASSWD:ALL` on each VM.

### Deprecated Output Plugin (`community.general.yaml`)
Newer `ansible-core` versions dropped this external callback.  
**Resolution in `ansible.cfg`:**
```ini
stdout_callback = ansible.builtin.default
[callback_default]
result_format = yaml
```

### Invalid Jinja2 Filter (`.ljust()`)
Python string method not valid in Jinja2 templates.  
**Resolution:**
```jinja2
{{ "%-16s" | format(record.name) }}
```

### BIND Bootstrap DNS Loop
Server had `127.0.0.1` as DNS but BIND was not yet installed — `dnf` could not resolve repos.  
**Resolution:** Set `8.8.8.8` temporarily during first playbook run; revert to `127.0.0.1` after named is active.

### Client VM Retained Static IP After Cloning
Cloned VM inherited static config from base image, causing duplicate IPs.  
**Resolution:**
```bash
sudo nmcli con mod enp1s0 ipv4.addresses "" ipv4.gateway ""
sudo nmcli con up enp1s0
```

### DHCP Pool Overlap with Reservations
Dynamic range started at `.100`, same as the `client01` fixed reservation.  
**Resolution:** Moved `dhcp_range_start` to `192.168.50.150`.

### containerd CRI Socket Unimplemented
`kubeadm init` failed with `unknown service runtime.v1.RuntimeService` because containerd config had stale defaults.  
**Resolution:** Always regenerate `config.toml` (removed `creates:` guard) and restart containerd explicitly with `wait_for` on the socket before continuing.

### SELinux Module Missing `python3-libselinux`
`ansible.posix.selinux` requires the SELinux Python bindings not present on Rocky 9 minimal.  
**Resolution:** Replaced the module with direct `getenforce`/`setenforce 0` shell commands.

### Master DNS Resolving via 8.8.8.8
`kubeadm init` preflight failed: `lookup master.jmelol.lab on 8.8.8.8:53: no such host`.  
**Resolution:** Added `nmcli con mod` task in `k8s_base` to point each k8s node's DNS to `192.168.50.10` before init runs.

### kubeadm Partial State After Failed Init
Re-running `kubeadm init` on a node with leftover state from a failed attempt caused errors.  
**Resolution:** Added `kubeadm reset --force` (with `failed_when: false`) before init when `admin.conf` does not exist.

### kubectl Bastion Role Did Not Execute
The bastion play ran but only gathered facts — `hostvars` fact from master was empty on re-runs.  
**Resolution:** Replaced in-memory `hostvars` transfer with `fetch` (master → Fedora host) + `copy` (Fedora host → ns1), making the role independent of play execution order.

---

## SELinux Notes

Rocky 9 runs SELinux in `enforcing` by default. Kubernetes nodes are set to `permissive` by the playbook. Expected security contexts on ns1:

| Resource | Context |
|---|---|
| `/etc/named.conf` | `system_u:object_r:named_conf_t:s0` |
| `/var/named/*.zone` | `system_u:object_r:named_zone_t:s0` |
| `/etc/dhcp/dhcpd.conf` | `system_u:object_r:dhcp_etc_t:s0` |

```bash
sudo ausearch -m avc -ts recent | audit2why
```

---
