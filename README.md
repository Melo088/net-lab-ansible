# lab1-ansible-dns-dhcp

Idempotent Ansible playbook to install and configure **BIND (named)** and **dhcpd** on Rocky Linux 9.  
Built as a network infrastructure lab on KVM/Fedora, designed to serve as the foundation for a future Kubernetes cluster.

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
│   ├── dns_bind/
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── named.conf.j2
│   │       ├── zone.forward.j2
│   │       └── zone.reverse.j2
│   └── dhcpd/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/
│           └── dhcpd.conf.j2
├── .gitignore
├── .github/workflows/ci.yml
└── README.md
```

---

## Development Environment

| Component | Details |
|---|---|
| Host OS | Fedora 43 (KDE Plasma) — kernel 6.19.7 |
| Hardware | Lenovo IdeaPad Slim 3 15AMN8 |
| Hypervisor | KVM/QEMU + libvirt (`dnf install @virtualization`) |
| Guest OS | Rocky Linux 9 Minimal |
| Ansible | Installed in an isolated `.venv` |

---

## Step-by-Step Setup

### 1. Install Ansible in a Virtual Environment (Fedora)

The `.venv` is created one level above the repository directory to keep it separate from project files:

```bash
cd ~/University/InfraIII/lab1-ansible-dns-dhcp   # parent directory

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install ansible ansible-lint

# Required collections
ansible-galaxy collection install ansible.posix community.general

ansible --version
```

To activate the virtual environment in each new session:

```bash
source ~/University/InfraIII/lab1-ansible-dns-dhcp/.venv/bin/activate
```

> The actual repository lives in the subdirectory `lab1-ansible-dns-dhcp/lab1-ansible-dns-dhcp/`.  
> All `ansible-playbook` commands must be run from within that subdirectory.

---

### 2. Create an Isolated NAT Network in KVM

Define a `lab-net` network (`192.168.50.0/24`) **without** libvirt's built-in DHCP, as it will be managed by the `dhcpd` role:

```bash
cat > /tmp/lab-net.xml <<'XMLEOF'
<network>
  <n>lab-net</n>
  <forward mode="nat"/>
  <bridge name="virbr50" stp="on" delay="0"/>
  <ip address="192.168.50.1" netmask="255.255.255.0"/>
</network>
XMLEOF

virsh net-define /tmp/lab-net.xml
virsh net-autostart lab-net
virsh net-start lab-net
virsh net-list --all     # expected output: lab-net  active  yes
```

---

### 3. Install the Base VM (Rocky 9 Minimal)

**ISO location:** `~/University/InfraIII/VM-ISO/Rocky-9-minimal*.iso`

In **virt-manager**: new VM → local ISO → 2048 MB RAM, 2 vCPUs, 20 GB disk, network `lab-net`.

**During Anaconda installation:**
- Software selection: Minimal Install
- Network: enable the interface with a provisional static IP `192.168.50.50/24`, gateway `192.168.50.1`
  > Note: Set `8.8.8.8` as a temporary DNS server during installation so `dnf` can resolve repository URLs. This can be reverted to `127.0.0.1` once BIND is operational.
- Hostname: `rocky9-base`
- Create user `rocky` in the `wheel` group

---

### 4. Prepare the Base VM for Ansible

Inside the VM (via virt-manager console or provisional SSH):

```bash
sudo dnf update -y
sudo dnf install -y python3 openssh-server
sudo systemctl enable --now sshd
```

**Enable passwordless sudo** (required for unattended Ansible execution):

```bash
echo "rocky ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/rocky
sudo chmod 440 /etc/sudoers.d/rocky
```

From the Fedora host, copy the SSH key:

```bash
ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/ansible_lab
ssh-copy-id -i ~/.ssh/ansible_lab.pub rocky@192.168.50.50
```

> The private key path is already configured in `ansible.cfg` as `private_key_file=~/.ssh/ansible_lab`.

---

### 5. Clone the Base VM

```bash
virsh shutdown rocky9-base

virt-clone --original rocky9-base --name dns-dhcp-server --auto-clone
virt-clone --original rocky9-base --name client01         --auto-clone
```

**Set a static IP on the server VM** (inside `dns-dhcp-server`):

```bash
virsh start dns-dhcp-server
# Access via virt-manager console

ip a     # identify the interface name, e.g. enp1s0

sudo nmcli con mod enp1s0 \
  ipv4.method manual \
  ipv4.addresses 192.168.50.10/24 \
  ipv4.gateway   192.168.50.1 \
  ipv4.dns       8.8.8.8 \
  connection.autoconnect yes
sudo nmcli con up enp1s0
sudo hostnamectl set-hostname ns1.jmelol.lab
```

Copy the SSH key to the server's definitive IP (from Fedora):

```bash
ssh-copy-id -i ~/.ssh/ansible_lab.pub rocky@192.168.50.10
```

> The interface name in this lab is `enp1s0`, which is set in `group_vars/all.yml` under `dhcp_interface`.  
> Once BIND is running, change the VM's DNS setting to `127.0.0.1`.

**Client VM** — configure for automatic DHCP:

```bash
sudo nmcli con mod enp1s0 ipv4.method auto
sudo hostnamectl set-hostname client01.jmelol.lab
```

If the cloned VM retains a static IP from the base image, remove it:

```bash
sudo nmcli con mod enp1s0 ipv4.addresses "" ipv4.gateway ""
sudo nmcli con up enp1s0
```

---

### 6. Retrieve MAC Addresses and Update DHCP Reservations

```bash
# From Fedora
virsh domiflist dns-dhcp-server
virsh domiflist client01
```

Edit `group_vars/all.yml` under `dhcp_reservations` with each VM's actual MAC address.  
The `master` and `worker` nodes are commented out until their VMs are created:

```yaml
dhcp_reservations:
#  - hostname: "master"
#    mac:  "52:54:00:AA:BB:20"
#    ip:   "192.168.50.20"
#  - hostname: "worker"
#    mac:  "52:54:00:AA:BB:21"
#    ip:   "192.168.50.21"
  - hostname: "client01"
    mac:  "52:54:00:2a:6a:bd"    # real MAC obtained via virsh domiflist
    ip:   "192.168.50.100"
```

---

### 7. Run the Playbook

```bash
cd ~/University/InfraIII/lab1-ansible-dns-dhcp/lab1-ansible-dns-dhcp
source ../../.venv/bin/activate

# Verify connectivity
ansible all -m ping

# Dry run
ansible-playbook site.yml --check --diff

# Apply
ansible-playbook site.yml
```

Expected result on first run: **22 tasks executed, approximately 14 changes applied**.  
Subsequent runs should be fully idempotent (0 changes).

---

## Network Topology

```
192.168.50.0/24  (lab-net / KVM NAT)
|
|-- 192.168.50.1    virbr50 — KVM gateway (libvirt)
|-- 192.168.50.10   ns1.jmelol.lab     <- DNS + DHCP server  [active]
|-- 192.168.50.20   master.jmelol.lab  <- future k8s master  [pending]
|-- 192.168.50.21   worker.jmelol.lab  <- future k8s worker  [pending]
`-- 192.168.50.100-200  dynamic DHCP pool
        `-- .100  client01 (fixed reservation by MAC)
```

---

## Verification Checklist

### From the Fedora host

```bash
# Static analysis
ansible-lint site.yml

# Check mode (no changes applied)
ansible-playbook site.yml --check --diff

# Connectivity test
ansible all -m ping
```

### From the server (192.168.50.10)

```bash
# Service status
sudo systemctl status named dhcpd

# Validate named.conf
sudo named-checkconf /etc/named.conf

# Validate forward zone
sudo named-checkzone jmelol.lab /var/named/jmelol.lab.zone

# Validate reverse zone
sudo named-checkzone 50.168.192.in-addr.arpa /var/named/50.168.192.in-addr.arpa.zone

# Validate dhcpd.conf
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Open ports
sudo ss -ulnp | grep -E '53|67'
sudo firewall-cmd --list-services

# DNS resolution tests
dig @192.168.50.10 ns1.jmelol.lab
dig @192.168.50.10 client01.jmelol.lab
dig @192.168.50.10 -x 192.168.50.100

# Active DHCP leases
sudo cat /var/lib/dhcpd/dhcpd.leases
```

---

## Issues Encountered and Resolutions

### SSH Permission Denied
The public key was installed on the provisional IP (`.50`) but not on the server's final IP (`.10`) after reconfiguration.  
**Resolution:** Re-run `ssh-copy-id` targeting `192.168.50.10`.

### Privilege Escalation Failure (Missing sudo password)
Ansible required an interactive sudo password, which is incompatible with unattended automation.  
**Resolution:** Create `/etc/sudoers.d/rocky` with `NOPASSWD:ALL` on the VM (see step 4).

### Deprecated Output Plugin (`community.general.yaml`)
Newer versions of `ansible-core` do not support this external callback plugin, causing the playbook to fail at startup.  
**Resolution applied in `ansible.cfg`:**
```ini
stdout_callback = ansible.builtin.default

[callback_default]
result_format = yaml
```

### Invalid Jinja2 Filter (`.ljust()`)
The `.ljust()` method is a Python string method and not a valid Jinja2 filter, causing template rendering to fail.  
**Resolution in `zone.forward.j2` and `zone.reverse.j2`:**
```jinja2
# Before
{{ record.name | ljust(16) }}
# After
{{ "%-16s" | format(record.name) }}
```

### BIND Package Installation Failure (DNS Bootstrap)
The server VM had `127.0.0.1` configured as its DNS resolver, but BIND was not yet installed, preventing `dnf` from resolving repository hostnames.  
**Resolution:** Configure `8.8.8.8` as a temporary DNS resolver on the VM before running the playbook. Revert to `127.0.0.1` once `named` is active.

### Incorrect `bind_service` Variable Value
An erroneous default value caused systemd to attempt managing a non-existent service unit.  
**Resolution:** Set `bind_service: "named"` in `roles/dns_bind/defaults/main.yml`.

### Client VM Retained Static IP After Cloning
The cloned VM inherited the static network configuration from the base image, resulting in two simultaneous IP addresses on the same interface.  
**Resolution:**
```bash
sudo nmcli con mod enp1s0 ipv4.addresses "" ipv4.gateway ""
sudo nmcli con up enp1s0
```

### Client Received Reserved IP Regardless of MAC Address
The dynamic DHCP pool started at `.100`, which was also the fixed reservation for `client01`. When tested with a different MAC, the server assigned `.100` from the dynamic pool rather than matching the reservation.  
**Lesson learned:** Do not overlap reservation IPs with the start of the dynamic range. Consider starting the dynamic pool at `.150` or higher to maintain a clear boundary.

---

## SELinux Notes

Rocky 9 runs SELinux in `enforcing` mode by default. The expected security contexts are:

| Resource | Expected context |
|---|---|
| `/etc/named.conf` | `system_u:object_r:named_conf_t:s0` |
| `/var/named/*.zone` | `system_u:object_r:named_zone_t:s0` |
| `/etc/dhcp/dhcpd.conf` | `system_u:object_r:dhcp_etc_t:s0` |

To investigate AVC denials:
```bash
sudo ausearch -m avc -ts recent | audit2why
```

---

## DHCP Interface Configuration

`dhcpd` requires an explicit interface name for its listener. The role writes this value to `/etc/sysconfig/dhcpd` as `DHCPDARGS="<interface>"`.

- **Value used in this lab:** `dhcp_interface: "enp1s0"` (set explicitly in `group_vars/all.yml`)
- If set to `auto`, the role reads `ansible_default_ipv4.interface` from Ansible facts, which may be unreliable on multi-interface hosts.
- **Recommendation:** always define the interface name explicitly. Verify the correct name with `ip -br a` inside the VM.

---

## Roadmap — Kubernetes

DNS records and DHCP reservations for the Kubernetes nodes are already defined in the variables.

| Task | Status |
|---|---|
| Create `master` VM by cloning `rocky9-base` | Pending |
| Create `worker` VM by cloning `rocky9-base` | Pending |
| Retrieve MACs and uncomment entries in `dhcp_reservations` | Pending |
| Uncomment `[k8s_master]` / `[k8s_workers]` in `inventory/hosts.ini` | Pending |
| Develop `k8s_base` role and add corresponding play in `site.yml` | Pending |