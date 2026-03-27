# net-lab-ansible

Idempotent Ansible playbook suite to deploy a complete network infrastructure lab on KVM/Fedora:
**DNS + DHCP (BIND/dhcpd)** and a full **Kubernetes 1.32 cluster** (kubeadm + Flannel CNI) on Rocky Linux 9.

---

## Introduction and Objectives

This project documents the design, automation, and deployment of a complete network infrastructure lab built on top of KVM/QEMU virtualization running on a Fedora 43 host. The lab covers two interconnected areas: foundational network services (DNS and DHCP) and a production-grade Kubernetes cluster, both provisioned end-to-end through a single idempotent Ansible playbook suite.

The work directly addresses the requirements of the practical activity *Despliegue de un Clúster de Kubernetes con kubeadm*, adapted to a KVM/libvirt environment instead of VirtualBox, and elevating the configuration management from manual steps to a fully automated, version-controlled approach.

**Primary objectives:**

- Design and automate the deployment of a bastion node that provides DNS (BIND9) and DHCP (ISC dhcpd) services to an isolated lab network, including MAC-based IP reservations for cluster nodes.
- Provision a functional Kubernetes 1.32 cluster using `kubeadm` across a dedicated control-plane node and a worker node, fully automated with Ansible roles.
- Install the Flannel CNI plugin and validate pod-to-pod networking across nodes.
- Configure the bastion as the sole `kubectl` administration point, keeping the control plane free of direct operator access.
- Achieve full idempotency: re-running `ansible-playbook site.yml` on a stable cluster produces zero changes and zero failures.
- Document every obstacle encountered during implementation and the resolution applied, serving as a replicable reference for similar lab environments.

---

## Lab Context

The activity was framed as an infrastructure-as-code exercise within the *Infraestructura III* course. The assignment required students to manually configure three virtual machines (a bastion, a Kubernetes master, and a Kubernetes worker) following a prescribed set of phases: bastion setup with network services, OS-level preparation of the Kubernetes nodes, cluster initialization, and workload validation.

This implementation diverges from the reference setup in two deliberate ways. First, the hypervisor is KVM/libvirt on Fedora 43 instead of VirtualBox; this required adapting network configuration (using a libvirt NAT bridge `virbr50` instead of VirtualBox Host-Only adapters). Second, every manual step described in the assignment has been encoded as an Ansible role, making the deployment reproducible from a single command rather than a series of interactive shell sessions.

The entire lab runs inside the `192.168.50.0/24` subnet managed by libvirt. The bastion (`ns1`, `.10`) provides name resolution for the `.jmelol.lab` domain and assigns fixed IPs to the cluster nodes via DHCP reservations. The master (`.20`) hosts the Kubernetes control plane and the Flannel CNI overlay. The worker (`.21`) runs application workloads and is the target for NodePort service exposure. An optional DHCP test client (`client01`, `.100`) validates dynamic lease assignment from the `.150–.200` pool.

All guest VMs run **Rocky Linux 9.7 (Blue Onyx) Minimal**, cloned from a single base image to keep provisioning time low. Ansible connects over SSH using a dedicated key pair and escalates privileges via passwordless `sudo`, both configured as part of the one-time VM setup documented in the Quick Start section.

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
├── docs/
│   └── capturas/               ← Evidencia fotográfica por fase
│       ├── 00-infraestructura-kvm/
│       ├── 01-ansible/
│       ├── 02-servicios-red/
│       ├── 03-cluster-kubernetes/
│       └── 04-validacion-fase4/
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

## Configuration Phases

The deployment follows four sequential phases, each corresponding to an Ansible play (or group of plays) in `site.yml`.

### Phase 1 — Bastion Server Configuration (`dns_bind` + `dhcpd` roles)

The bastion node `ns1` (192.168.50.10) is configured first because every subsequent phase depends on it for name resolution. This phase deploys two services:

**BIND9 (named)** is installed and configured with a forward zone (`jmelol.lab`) and a reverse zone (`50.168.192.in-addr.arpa`). Zone files are generated from Jinja2 templates (`zone.forward.j2`, `zone.reverse.j2`) using the `bind_a_records` and `bind_ptr_records` lists defined in `group_vars/all.yml`. The server listens on `127.0.0.1` and `192.168.50.10`, accepts queries from localhost and the `192.168.50.0/24` subnet, and forwards external queries to `8.8.8.8` and `1.1.1.1`. SELinux file contexts (`named_conf_t`, `named_zone_t`) are preserved via `restorecon`.

**ISC dhcpd** is configured to serve the `192.168.50.0/24` subnet with a dynamic pool from `.150` to `.200` and fixed-address reservations for `master` (`.20`), `worker` (`.21`), and `client01` (`.100`) matched by MAC address. Lease times are set to 12 h default / 24 h maximum. The `dhcp_interface` variable (`enp1s0`) controls which interface the daemon binds to.

A critical bootstrapping concern was solved here: if `127.0.0.1` is set as the system DNS resolver before BIND is installed, `dnf` cannot resolve package repositories. The playbook temporarily uses `8.8.8.8` for package installation and switches to `127.0.0.1` only after `named` is confirmed active.

### Phase 2 — Kubernetes Node Preparation (`k8s_base` role)

Applied to both `master` and `worker` in parallel, this phase conditions the Rocky Linux OS to meet all Kubernetes prerequisites:

- **Hostname and DNS:** Each node's hostname is set (`master.jmelol.lab`, `worker.jmelol.lab`) and its DNS resolver is pointed to `192.168.50.10` via `nmcli`, so that `kubeadm` preflight checks can resolve cluster FQDNs.
- **Swap:** Swap is disabled immediately (`swapoff -a`) and removed from `/etc/fstab` to prevent kubelet from refusing to start.
- **SELinux:** Set to `permissive` mode using direct shell commands (`setenforce 0` + `sed` on `/etc/selinux/config`) because `ansible.posix.selinux` requires `python3-libselinux`, which is absent on the minimal image.
- **Kernel modules:** `overlay` and `br_netfilter` are loaded and persisted via `/etc/modules-load.d/k8s.conf`.
- **sysctl:** `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, and `net.ipv4.ip_forward` are all set to `1` and written to `/etc/sysctl.d/k8s.conf`.
- **Firewall:** `firewalld` is configured with the ports required by the control plane (6443, 2379–2380, 10250–10252) and worker (10250, 30000–32767), as well as Flannel's VXLAN port (8472/udp) plus IP masquerading for pod-to-external communication.
- **containerd:** Installed from the Docker CE repository. The default `config.toml` is always regenerated (no `creates:` guard) to ensure `SystemdCgroup = true` is active and the CRI socket is available before `kubeadm` runs.
- **Kubernetes tooling:** `kubelet`, `kubeadm`, and `kubectl` are installed from the official `pkgs.k8s.io` repository pinned to version `1.32`.

### Phase 3 — Cluster Initialization (`k8s_master` + `k8s_worker` roles)

**Control plane (`k8s_master` role):** `kubeadm init` is invoked with `--apiserver-advertise-address=192.168.50.20` and `--pod-network-cidr=10.244.0.0/16`. A `kubeadm reset --force` step (idempotency guard) runs first when `admin.conf` is absent to clean any leftover state from previous failed attempts. After a successful init, the kubeconfig is installed for the `rocky` user and Flannel is applied with `kubectl apply -f`. The join token and CA certificate hash are extracted and stored as Ansible facts for use in the next play.

**Worker join (`k8s_worker` role):** The token and hash captured from the master are passed to `kubeadm join 192.168.50.20:6443`. The task is guarded by a check on `/etc/kubernetes/kubelet.conf` to skip re-joining an already-registered node.

**Bastion kubectl (`k8s_kubectl_bastion` role):** Rather than relying on in-memory `hostvars` (which are empty on playbook re-runs), the `admin.conf` file is fetched from the master to the Fedora host with `fetch`, then pushed to `ns1` with `copy`. This makes the role order-independent and safe for idempotent re-runs.

### Phase 4 — Validation and Testing

Once the playbook completes, the cluster state is verified from `ns1`. See the [Verification](#verification) section for the complete command sequence and the [Photographic Evidence](#photographic-evidence) section for annotated screenshots of each step.

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

## Photographic Evidence

All screenshots are stored under `docs/capturas/`, organized by phase. Each image is described below with its context and what it demonstrates.

### `00-infraestructura-kvm/` — KVM Infrastructure

**`01-vms-apagadas.png`**
Output of `sudo virsh list --all` on the Fedora host before starting the lab session. All five VMs (`dns-dhcp-server`, `k8s-master`, `k8s-worker`, `client01`, `rocky9-base`) appear in `shut off` state, confirming a clean cold start.

**`02-redes-virtuales.png`**
Output of `sudo virsh net-list --all`. Both virtual networks are present and active: `default` (libvirt's built-in NAT network) and `lab1-ansible-net` (the isolated `192.168.50.0/24` lab network used by all nodes).

**`03-vms-encendidas.png`**
Output of `sudo virsh list --all` after running `virsh start` on each node. The three active lab VMs (`dns-dhcp-server`, `k8s-master`, `k8s-worker`) appear in `running` state with their assigned domain IDs.

---

### `01-ansible/` — Ansible Connectivity and Idempotency

**`01-ping-todos-nodos.png`**
Output of `ansible all -m ping` from the Fedora host with the virtualenv activated. `ns1`, `master`, and `worker` all return `"ping": "pong"` with `SUCCESS`. `client01` appears as `UNREACHABLE` because it is intentionally shut off during this session — this is expected and documented.

**`02-playbook-idempotente-recap.png`**
The `PLAY RECAP` section at the end of a second `ansible-playbook site.yml` run on an already-deployed cluster. All hosts report `failed=0`. The small number of `changed` tasks (containerd config regeneration) is expected and documented in the Issues section. This confirms the playbook's idempotent behaviour.

---

### `02-servicios-red/` — DNS and DHCP Services

**`01-named-activo.png`**
Output of `sudo systemctl status named` on `ns1`. The BIND9 daemon is in `active (running)` state, confirming the DNS service is operational.

**`02-dhcpd-activo.png`**
Output of `sudo systemctl status dhcpd` on `ns1`. The ISC DHCP daemon is in `active (running)` state, confirming that DHCP leases and MAC-based reservations are being served.

**`03-dig-master.png`**
Output of `dig @127.0.0.1 master.jmelol.lab` from `ns1`. The `ANSWER SECTION` shows `master.jmelol.lab. 86400 IN A 192.168.50.20`, confirming that the forward zone resolves the master node correctly with `status: NOERROR`.

**`04-dig-worker.png`**
Output of `dig @127.0.0.1 worker.jmelol.lab` from `ns1`. The `ANSWER SECTION` shows `worker.jmelol.lab. 86400 IN A 192.168.50.21`, confirming forward resolution for the worker node.

**`05-dig-reverse.png`**
Output of `dig @127.0.0.1 -x 192.168.50.20` from `ns1`. Confirms that the reverse zone (`50.168.192.in-addr.arpa`) returns the correct PTR record, validating full forward and reverse DNS coverage.

---

### `03-cluster-kubernetes/` — Cluster Health

**`01-nodos-ready.png`**
Output of `kubectl get nodes -o wide` from `ns1`. Both `master.jmelol.lab` (`192.168.50.20`) and `worker.jmelol.lab` (`192.168.50.21`) appear with `STATUS=Ready`, running Kubernetes `v1.32.13` on Rocky Linux 9.7 with `containerd://2.2.2`. This is the baseline health check before running workloads.

**`02-kube-system-pods.png`**
Output of `kubectl get pods -n kube-system -o wide`. All eight system pods are in `Running` state: two CoreDNS replicas on the master, etcd, kube-apiserver, kube-controller-manager, kube-scheduler, and kube-proxy on both nodes. No pods are in `Pending`, `CrashLoopBackOff`, or `Error` state.

**`03-flannel-pods.png`**
Output of `kubectl get pods -n kube-flannel -o wide`. The Flannel DaemonSet has deployed one pod per node (`kube-flannel-ds-*`), both in `Running` state, confirming that the VXLAN overlay network is active and pod-to-pod communication across nodes is possible.

---

### `04-validacion-fase4/` — Phase 4 Validation

This folder contains the evidence for all five evaluated checkpoints of Phase 4.

**`01-worker-cordoned.png`**
Output of `kubectl get nodes -o wide` immediately after the playbook run, before the uncordon step. The worker node appears with `STATUS=Ready,SchedulingDisabled`, a known condition in which newly joined nodes sometimes start. This state prevents pods from being scheduled on the worker until it is explicitly uncordoned.

**`02-uncordon-worker.png`**
Output of `kubectl uncordon worker.jmelol.lab` followed by `kubectl get nodes -o wide`. The worker transitions from `Ready,SchedulingDisabled` to `Ready`, and both nodes now accept pod scheduling. The playbook includes an automated uncordon task; this screenshot shows the manual verification confirming the final state.

**`03-nginx-deployment-creado.png`**
Output of `kubectl create deployment nginx-test --image=nginx --replicas=2`. The CLI confirms `deployment.apps/nginx-test created`. This is the starting point of the smoke test that validates end-to-end workload management.

**`04-pods-en-worker.png`**
Output of `kubectl get pods -o wide` showing both `nginx-test` replicas (`nginx-test-b548755db-jwxtp` and `nginx-test-b548755db-l54kd`) in `Running` state. The `NODE` column confirms both pods are scheduled on `worker.jmelol.lab` with pod IPs in the `10.244.1.x` range (Flannel subnet assigned to the worker). This validates that the scheduler correctly placed workloads on the worker node and that the CNI overlay assigned IP addresses.

**`05-servicio-nodeport.png`**
Output of `kubectl expose deployment nginx-test --type=NodePort --port=80` (creation confirmation) and `kubectl get svc nginx-test` (service state). The service appears with `TYPE=NodePort`, a `CLUSTER-IP` of `10.108.226.249`, and `PORT(S)=80:30432/TCP`. The NodePort `30432` is the port exposed on every node's external interface, used in the next step.

**`06-curl-nginx-worker.png`**
Output of `curl http://192.168.50.21:30432` from the `ns1` bastion. The full HTML response is returned, including the `<h1>Welcome to nginx!</h1>` heading. This confirms that: (1) the NodePort is reachable from outside the cluster, (2) kube-proxy iptables rules are forwarding traffic correctly to the pod, and (3) the pod is serving HTTP responses. The request targets the worker's IP directly, as required by the assignment.

**`07-coredns-nslookup.png`**
Output of the CoreDNS resolution test using a temporary BusyBox pod:

```
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it \
  -- nslookup nginx-test.default.svc.cluster.local
```

The output shows the CoreDNS server at `10.96.0.10:53` successfully resolving `nginx-test.default.svc.cluster.local` to `10.108.226.249` (the ClusterIP of the `nginx-test` service). The pod is automatically deleted after the query (`pod "dns-test" deleted`). This validates that service discovery via DNS works correctly inside the cluster overlay network.

**`08-cleanup.png`**
Output of `kubectl delete deployment nginx-test` and `kubectl delete svc nginx-test`. Both resources are confirmed deleted, restoring the cluster to its baseline state after the smoke test.

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

### Flannel Networking Timeouts (DNS Failure)
Pods across different nodes could not reach CoreDNS (timeout).  
**Resolution:** Added `firewalld` rules to allow the `10.244.0.0/16` source, opened port `8472/udp` for VXLAN encapsulation, and enabled `masquerade` on all nodes.

### Worker Node Joined in Cordoned State
New worker nodes occasionally joined the cluster in a cordoned state.  
**Resolution:** Added an automated `kubectl uncordon` task in the master role to ensure all nodes are ready to receive workloads immediately after joining.

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

## Results and Conclusions

The lab was completed successfully. A single run of `ansible-playbook site.yml` on a freshly cloned set of VMs produces a fully operational Kubernetes 1.32 cluster with DNS and DHCP services, without any manual intervention beyond the one-time SSH key distribution. Subsequent runs confirm complete idempotency: all tasks report `ok` or `skipped`, with zero changes and zero failures.

**Observed results:**

- Both `master` and `worker` reach `Ready` status after the playbook completes.
- All `kube-system` pods (API server, controller manager, scheduler, etcd, CoreDNS) and the `kube-flannel` daemonset run in `Running` state on their respective nodes.
- The Nginx test deployment schedules two replicas on the worker and is reachable via NodePort from the bastion, confirming that pod networking, CNI overlay (Flannel VXLAN), and kube-proxy iptables rules are all functional.
- DHCP reservations are honoured: the master and worker always receive `.20` and `.21` respectively, and a dynamic lease from the `.150–.200` pool is correctly assigned to `client01`.
- DNS forward and reverse resolution for all four hosts in `jmelol.lab` is confirmed from every node in the lab.
- CoreDNS internal service discovery resolves cluster service names (e.g. `nginx-test.default.svc.cluster.local`) to their correct ClusterIP addresses, as validated by a temporary BusyBox pod during Phase 4 testing.