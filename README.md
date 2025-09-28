## Ansible Talos K8s Role

This role provisions a Talos-based Kubernetes cluster on Proxmox and bootstraps it using `talosctl`. It creates Control Plane and Worker VMs, applies Talos configs, bootstraps the cluster, and writes `kubeconfig` locally for immediate `kubectl` access.

## Features

- **Proxmox VM Provisioning**: Creates `talos-cp-*` and `talos-wrk-*` VMs with CPU/RAM/Disk/bridge settings
- **Deterministic IPs**: Static IPs constructed from a network prefix and starting octet for idempotence
- **Automated Bootstrap**: Generates Talos configs, applies to nodes, and performs `talosctl bootstrap`
- **Health-aware Idempotence**: Skips bootstrap if the cluster is already healthy
- **Reproducible Inputs-Only Generation**: Machine configs are generated from `secrets.yaml` + patches + version inputs and not stored; see [Reproducible Machine Configuration](https://www.talos.dev/v1.11/talos-guides/configuration/reproducible-machine-config/)
- **Local Artifacts**: Persists `talosconfig` and `kubeconfig` in `./<cluster_name>/`; generated machine configs are removed after apply

## Upgrades

Talos OS upgrades are API-driven and safe (A/B images with automatic rollback). This role provides a dedicated upgrade flow that serializes upgrades across nodes and is idempotent.

Reference: [Talos: Upgrading Talos Linux](https://www.talos.dev/v1.11/talos-guides/upgrading-talos/)

### Enable and run upgrades

1) Set the target Talos version (used both for generation and upgrades):

```yaml
talos_version: "v1.11.1"
```

2) Either enable the flag or use tags:

```yaml
talos_upgrade_enabled: true
```

Or via tags:

```bash
ansible-playbook -i inventory.ini site.yml --tags upgrade
```

3) Optional behaviors:

- `talos_upgrade_stage: true` to use staged upgrade when files are held open
- `talos_upgrade_wait: true` to wait and stream progress

### Idempotency

Before upgrading, the role runs `talosctl version` per node and compares against `talos_version`. If they match, the upgrade step for that node is skipped.

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
# ---------------------------------------------------------------------
# Proxmox Configuration
# ---------------------------------------------------------------------
proxmox_host: "192.168.0.200:8006"
proxmox_node: "proxmox"
proxmox_datastore: "local-lvm"
proxmox_user: "root@pam"
# Use either API token (preferred) or password (one may be omitted)
proxmox_token_id: ""          # e.g. ansible@pve!token_name
proxmox_token_secret: ""      # store with Ansible Vault
proxmox_password: ""          # store with Ansible Vault

# ---------------------------------------------------------------------
# Talos Cluster
# Core settings and software versions.
# ---------------------------------------------------------------------
talos_cluster_name: "dev_k8s"
talos_version: "v1.11.1"            # REQUIRED: Talos version contract for gen/upgrade
talos_kubernetes_version: "v1.34.0" # REQUIRED: Current Kubernetes version
talos_install_disk: "/dev/vda"

# ---------------------------------------------------------------------
# Network Configuration
# ---------------------------------------------------------------------
network: "192.168.0"
start_octet: 100
cidr: 24
gateway: "192.168.0.1"

# ---------------------------------------------------------------------
# Virtual IP
# ---------------------------------------------------------------------
vip_enabled: true
vip_address: "192.168.0.150"
vip_interface: "eth0"

# ---------------------------------------------------------------------
# Control Plane Node Specification
# ---------------------------------------------------------------------
talos_cp_count: 3
talos_cp_cores: 1
talos_cp_memory: 2048
talos_cp_disk: 20
talos_cp_network: "vmbr0"

# ---------------------------------------------------------------------
# Worker Node Specification
# ---------------------------------------------------------------------
talos_worker_count: 2
talos_worker_cores: 2
talos_worker_memory: 4096
talos_worker_disk: 20
talos_worker_network: "vmbr0"

# ---------------------------------------------------------------------
# Automation & Lifecycle Flags
# ---------------------------------------------------------------------
bootstrap_proxmox: true
bootstrap_talos_cluster: true
talos_upgrade_enabled: false
talos_upgrade_stage: false
talos_upgrade_wait: true
```
## Reproducible Config Generation

This role follows Talos' recommended inputs-only workflow to avoid drift:

- Generate or reuse a `secrets.yaml` bundle once per cluster. If missing, the role runs `talosctl gen secrets --talos-version <talos_version> -o <cluster>/secrets.yaml`. See [Generate Secrets Bundle](https://www.talos.dev/v1.11/introduction/prodnotes/#step-5-generate-secrets-bundle).
- Always generate machine configs from inputs: `secrets.yaml`, `talos_version`, `talos_kubernetes_version`, and any patches (VIP patch is applied automatically when enabled). See [Reproducible Machine Configuration](https://www.talos.dev/v1.11/talos-guides/configuration/reproducible-machine-config/).
- Generated `controlplane.yaml` and `worker.yaml` are applied to nodes and then deleted locally.


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

### Proxmox Configuration

- `proxmox_host` (str): `host:port` of Proxmox API
- `proxmox_node` (str): Proxmox node name
- `proxmox_datastore` (str): Datastore for disks (e.g. `local-lvm`)
- `proxmox_user` (str): API username (e.g. `root@pam`)
- `proxmox_token_id` (str), `proxmox_token_secret` (str): API token auth (preferred)
- `proxmox_password` (str): API password (store with Vault)

### Talos Cluster

- `talos_cluster_name` (str): Output directory and cluster name (default: `dev_k8s`)
- `talos_version` (str): REQUIRED Talos version contract used for gen/upgrade
- `talos_kubernetes_version` (str): REQUIRED current Kubernetes version
- `talos_install_disk` (str): Target install disk inside VMs (e.g. `/dev/vda`)

### Network Configuration

- `network` (str): Network prefix without last octet (e.g. `192.168.0`)
- `start_octet` (int): Starting host octet for IP allocation
- `cidr` (int): Subnet mask length (e.g. `24`)
- `gateway` (str): Default gateway IP (e.g. `192.168.0.1`)

### Virtual IP

- `vip_enabled` (bool): Enable Talos VIP on control-plane nodes
- `vip_address` (str): Shared virtual IP on CP subnet (required when enabled)
- `vip_interface` (str): Host interface name used for the VIP (default: `eth0`)

### Control Plane Node Specification

- `talos_cp_count` (int)
- `talos_cp_cores` (int), `talos_cp_memory` (MiB), `talos_cp_disk` (GiB)
- `talos_cp_network` (str): Proxmox bridge (e.g. `vmbr0`)

### Worker Node Specification

- `talos_worker_count` (int)
- `talos_worker_cores` (int), `talos_worker_memory` (MiB), `talos_worker_disk` (GiB)
- `talos_worker_network` (str): Proxmox bridge

### Automation & Lifecycle Flags

- `bootstrap_proxmox` (bool): Create/start VMs in Proxmox (default: true)
- `bootstrap_talos_cluster` (bool): Run Talos bootstrap flow (default: true)
- `talos_upgrade_enabled` (bool): Import and run upgrade tasks (default: false)
- `talos_upgrade_stage` (bool): Use `--stage` upgrade mode (default: false)
- `talos_upgrade_wait` (bool): Use `--wait` to observe upgrade (default: true)

## Outputs

- Directory `./<talos_cluster_name>/` containing:
  - `talosconfig`
  - `kubeconfig` (mode `0600`)
- `talosctl` endpoint set to the all CPs or VIP
- `kubeconfig` is written directly to `./<talos_cluster_name>/kubeconfig` (not merged into `~/.kube/config`); when VIP is enabled, its `server` points at the VIP

## Tags

You can target specific parts of the role using tags:

- `talos`: General tag applied to Talos-related operations
- `upgrade`: Runs the Talos OS upgrade flow (`tasks/upgrade.yml`). Usually combined with `talos`.
- `destroy`: Paired with `never`; use to explicitly stop/delete VMs:

```bash
# stop worker and control plane VMs (requires --tags destroy)
ansible-playbook -i inventory.ini site.yml --tags destroy
```

Notes:
- Destructive actions in `tasks/proxmox.yml` are tagged `[destroy, never]`, so they run only when explicitly requested via `--tags destroy`.
- Upgrade tasks are included when `talos_upgrade_enabled: true` or when run with `--tags upgrade`.

## Notes and Limitations

- VM names are fixed format: `talos-cp-*` and `talos-wrk-*`
- Single NIC per VM; adjust role if multiple NICs are required
- Does not delete or resize existing VMs; changes to counts/sizes may require manual reconciliation