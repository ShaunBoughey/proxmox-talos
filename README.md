## Ansible Talos K8s Role

This role provisions a Talos-based Kubernetes cluster on Proxmox and bootstraps it using `talosctl`. It creates Control Plane and Worker VMs, applies Talos configs, bootstraps the cluster, and writes `kubeconfig` locally for immediate `kubectl` access.

## Features

- **Proxmox VM Provisioning**: Creates `talos-cp-*` and `talos-wrk-*` VMs with CPU/RAM/Disk/bridge settings
- **Deterministic IPs**: Static IPs constructed from a network prefix and starting octet for idempotence
- **Automated Bootstrap**: Generates Talos configs, applies to nodes, and performs `talosctl bootstrap`
- **Health-aware Idempotence**: Skips bootstrap if the cluster is already healthy
- **Local Artifacts**: Writes `talosconfig` and `kubeconfig` into `./<cluster_name>/`

## Quick Start

### 1) Requirements

- **Controller machine**
  - `ansible >= 2.12`
  - `talosctl` installed and on PATH
  - `kubectl` (optional, recommended)
  - Ansible collection `community.proxmox`:
    ```bash
    ansible-galaxy collection install community.proxmox
    ```
- **Proxmox**
  - API reachable from the controller (e.g. `https://<host>:8006`)
  - Talos ISO uploaded as `local:iso/talos-amd64.iso` (or adjust in the role)
  - Datastore for disks (e.g. `local-lvm`)
  - Cloud-Init storage available (e.g. `<datastore>:cloudinit`)

### 2) Minimal Playbook

```yaml
- name: Talos K8s bootstrap
  hosts: local
  gather_facts: false
  roles:
    - talos-k8s
```

### 3) Variables (example)

Override via inventory, `group_vars`, or `extra_vars`. Store secrets with Ansible Vault.

```yaml
# Proxmox
bootstrap_proxmox: true
proxmox_host: "192.168.0.200:8006"
proxmox_user: "root@pam"
# Use either API token (preferred) or password (one may be omitted)
proxmox_token_id: ""                    # e.g. ansible@pve!token_name
proxmox_token_secret: ""                # store with Ansible Vault
proxmox_password: ""                    # store with Ansible Vault
proxmox_node: "proxmox"
proxmox_datastore: "local-lvm"

# IP scheme
network: "192.168.0"                    # network prefix (no last octet)
start_octet: 100                         # first assigned octet
cidr: 24                                 # subnet mask length
gateway: "192.168.0.1"                  # default gateway

# Cluster layout
talos_cluster_name: "dev_k8s"
talos_install_disk: "/dev/vda"

# Control plane
talos_cp_count: 1
talos_cp_cores: 1
talos_cp_memory: 2048
talos_cp_disk: 20
talos_cp_network: "vmbr0"

# Workers
talos_worker_count: 2
talos_worker_cores: 2
talos_worker_memory: 4096
talos_worker_disk: 20
talos_worker_network: "vmbr0"

# Bootstrap toggle
bootstrap_talos_cluster: true

# VIP (optional)
vip_enabled: false
vip_address: ""             # e.g. 192.168.0.50
vip_interface: "eth0"
vip_arp: true
vip_namespace: "kube-system"
```

### 4) Run

```bash
ansible-playbook /path/to/site.yml
```

After completion, your artifacts will be in `./<talos_cluster_name>/`, including `kubeconfig`.

## Using the Cluster

```bash
# point kubectl at generated kubeconfig
export KUBECONFIG="$(pwd)/dev_k8s/kubeconfig"

# verify nodes
kubectl get nodes -o wide

# check talos health (uses talosconfig in ./dev_k8s)
talosctl --talosconfig=dev_k8s/talosconfig health
```

## Variable Reference

### Proxmox

- `bootstrap_proxmox` (bool): Create/start VMs in Proxmox (default: true)
- `proxmox_host` (str): `host:port` of Proxmox API
- `proxmox_user` (str): API username (e.g. `root@pam`)
- `proxmox_password` (str): API password (store with Vault)
- `proxmox_token_id` (str): API token identifier (alternative to password)
- `proxmox_token_secret` (str): API token secret (store with Vault)
- `proxmox_node` (str): Proxmox node name
- `proxmox_datastore` (str): Datastore for disks (e.g. `local-lvm`)

### IP Scheme

- `network` (str): Network prefix without last octet (e.g. `192.168.0`)
- `start_octet` (int): Starting host octet for IP allocation
- `cidr` (int): Subnet mask length (e.g. `24`)
- `gateway` (str): Default gateway IP (e.g. `192.168.0.1`)

### Talos Cluster

- `talos_cluster_name` (str): Output directory and cluster name (default: `dev_k8s`)
- `talos_install_disk` (str): Target install disk inside VMs (e.g. `/dev/vda`)
- `talos_cp_count` (int): Number of control plane nodes
- `talos_cp_cores` (int), `talos_cp_memory` (MiB), `talos_cp_disk` (GiB)
- `talos_cp_network` (str): Proxmox bridge (e.g. `vmbr0`)
- `talos_worker_count` (int)
- `talos_worker_cores` (int), `talos_worker_memory` (MiB), `talos_worker_disk` (GiB)
- `talos_worker_network` (str): Proxmox bridge
- `bootstrap_talos_cluster` (bool): Run Talos bootstrap flow (default: true)

### VIP (API High Availability)

- `vip_enabled` (bool): Enable Talos VIP on control-plane nodes
- `vip_address` (str): Shared virtual IP on CP subnet (required when enabled)
- `vip_interface` (str): Host interface name used for the VIP (default: `eth0`)
- `vip_arp` (bool): Use ARP mode (recommended; BGP not managed by this role)
- `vip_namespace` (str): Namespace for any VIP-related resources (default: `kube-system`)

## Outputs

- Directory `./<talos_cluster_name>/` containing:
  - `controlplane.yaml`, `worker.yaml`
  - `talosconfig`
  - `kubeconfig` (mode `0600`)
- `talosctl` endpoint set to the first CP IP
- `kubeconfig` is written directly to `./<talos_cluster_name>/kubeconfig` (not merged into `~/.kube/config`); when VIP is enabled, its `server` points at the VIP

## Monitoring

```bash
# Talos API health (requires talosconfig)
talosctl --talosconfig=dev_k8s/talosconfig health

# List machines
talosctl --talosconfig=dev_k8s/talosconfig -n 192.168.0.100 get machines

# Kubernetes node status
export KUBECONFIG="$(pwd)/dev_k8s/kubeconfig"
kubectl get nodes -o wide
```

## Troubleshooting

- **Proxmox module missing**
  - Install `community.proxmox`: `ansible-galaxy collection install community.proxmox`
- **Talos ISO not found**
  - Ensure `local:iso/talos-amd64.iso` exists (or update the role/template accordingly)
- **Wait for port 50000 times out**
  - Verify Talos VMs booted and have network connectivity with correct IP/gateway
  - Confirm bridge names (`vmbr0`, etc.) and VLANs are correct
- **Cluster re-bootstrap**
  - The role skips bootstrap if healthy; to force a fresh run, ensure the cluster is not healthy and/or remove `./<cluster_name>/talosconfig` (caution), or recreate VMs
- **Credentials**
  - Use Ansible Vault for `proxmox_password`. Do not commit secrets.

## Notes and Limitations

- VM names are fixed format: `talos-cp-*` and `talos-wrk-*`
- Single NIC per VM; adjust role if multiple NICs are required
- Does not delete or resize existing VMs; changes to counts/sizes may require manual reconciliation