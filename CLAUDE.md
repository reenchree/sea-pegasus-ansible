> **DECOMMISSIONED 2026-10-14.** The only migrated workload (Minecraft Bedrock
> "ReenChree Bedrock Server") now runs on the Hercules `minecraft` VM —
> see sea-hercules-ansible/CLAUDE.md "Minecraft VM". Semaphore project 2 is
> deleted 2026-10-14 (Semaphore DB preserved in Hercules snapshot `tank/vms@pre-sem-project2-delete-2026-10-14`). The host was powered off on 2026-10-14 after recovery
> (`vmpool` exported cleanly; `node-2` and `pterodactyl` VMs left defined, shut
> off). If it is ever revived: it last ran at 192.168.50.100 via `enp5s0`
> (1GbE); the 2.5GbE port is actually `enp4s0`. The full Pterodactyl staging
> archive (both nodes' Wings config, panel MariaDB, Bedrock + Paper 1.21.10
> volumes, Pterodactyl backups) is retained at Hercules
> `/tank/vms/_migration/pegasus-2026-10-14` (root-only; Paper + Satisfactory
> data archived, not deployed). This repo is kept for reference only.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation for **pegasus** (192.168.50.100), a Debian 13 (trixie) KVM/libvirt VM host that runs Pterodactyl game servers. Manages the full stack: ZFS RAIDZ1 storage, VLAN networking via systemd-networkd, VM provisioning via cloud-init, and the Pterodactyl Panel + Wings stack inside guest VMs.

**Status note (2026-10-14, pre-decommission)**: **Historical, superseded by the DECOMMISSIONED banner:** Pegasus was **back on**, re-cabled: the only live link is `enp5s0` on an **access port, VLAN 50** (Family Room Lab 2.5g Switch port 1), DHCP **192.168.50.100** (no longer 192.168.0.253, no VLAN 1 mgmt). Networking was hand-fixed live, then mirrored into `roles/network` (rendered templates diff-clean vs the host). Root cause of past drop-offs: three managers fighting — NetworkManager (auto "Wired connection" profiles), ifupdown/dhcpcd on bridge port `enp4s0` (same MAC as `br0`), and networkd, which also dropped the lease on every carrier flap (one 2026-04-01 flap left the host IP-less for 64 min). Now networkd-only (NM masked, ifupdown lo-only). VMs (`pterodactyl`, `node-2`) were **shut off** and `/vmpool` data was recovered, then exported cleanly. Scrape targets were removed from sea-k8s-flux at decommission (G1).

## Hardware

- **Host**: pegasus, custom build (no vendor — generic AMD/Intel desktop, two onboard Realtek NICs: `enp4s0` = RTL8125B **2.5GbE** `a8:29:48:24:aa:5d`, `enp5s0` = RTL8168h **1GbE** `a8:a1:59:5f:b6:4a`, both `r8169`; plus an unused ConnectX-4 Lx `enp6s0f0np0`/`enp6s0f1np1`)
- **OS**: Debian 13 trixie
- **Storage**: 4x SATA disks (`/dev/sd[a-d]`) in a ZFS RAIDZ1 pool named `vmpool`, mounted at `/vmpool`. Also serves as libvirt's storage pool path. (Earlier README claims "mirror on 2x Samsung EVO 500GB"; the live config in `inventory.yml` is **RAIDZ1 on 4 disks** — README is stale, this CLAUDE.md and the inventory are authoritative.)
- **Network**: `enp5s0` (1GbE) on a VLAN 50 **access** port, sole port of `br-vlan50` (bridge MAC pinned to `a8:a1:59:5f:b6:4a`). The 2.5GbE `enp4s0` is uncabled — to use it, recable and change `primary_interface` (pinned bridge MAC keeps the lease). Never bridge both NICs to the same switch (STP is off → loop).
- **SSH user**: `pi` (not `localadmin` — pegasus pre-dates the hercules naming switch)

Live hardware specs (CPU/RAM exact model, drive sizes) are not in this repo — they live on the box. To re-confirm if ever revived:

```bash
ssh -i ~/.ssh/id_ansible pi@192.168.50.100 'lscpu; free -h; lsblk; sudo zpool status'
```

## Repository Structure

```
.
├── ansible.cfg                # become=True globally, pipelining, profile_tasks callback
├── requirements.yml           # External collections/roles (see "External Dependencies")
├── inventory.yml              # Committed to git, secrets injected via Semaphore env
├── playbooks/
│   ├── site.yml                     # Full host provisioning (network role tagged 'never')
│   ├── check-system.yml             # Health check: zpool, virsh pools, bridges, vlans, networkd
│   ├── network-only.yml             # Apply network role explicitly (DANGEROUS, needs console access)
│   ├── network-rollback.yml         # Emergency: revert to ifupdown if systemd-networkd cutover broke SSH
│   ├── node_exporter.yml            # Standalone redeploy of reenchree.common.node_exporter
│   ├── zfs_exporter.yml             # Standalone redeploy of reenchree.common.zfs_exporter
│   │                                # (no standalone smartctl_exporter.yml — only via site.yml)
│   ├── nut-client.yml               # NUT UPS client + custom vm-host-shutdown hook
│   ├── pterodactyl.yml              # Create pterodactyl VM + install MariaDB + Panel + Wings
│   ├── node-2.yml                   # Create node-2 VM + install Wings only
│   ├── pterodactyl-teardown.yml     # Destroy pterodactyl VM
│   ├── pterodactyl-backup.yml       # mysqldump panel DB + copy .env (30d retention)
│   ├── pterodactyl-update.yml       # Pull latest Panel release, migrate, fix perms
│   ├── apt-upgrade.yml              # Notify-only on vm_hosts (no auto-reboot)
│   └── apt-upgrade-vms.yml          # Serial:1 on vms, auto-reboot, verifies wings active
└── roles/                     # Local roles (see "Roles" table)
    ├── prereq/                # Enable contrib/non-free, install base packages
    ├── zfs/                   # Install ZFS via DKMS, create vmpool if missing
    ├── network/               # systemd-networkd only: uplink -> br-vlan50 (DHCP, pinned MAC)
    ├── libvirt/               # KVM/libvirt + storage pool at /vmpool
    ├── cockpit/               # Cockpit web UI on :9090
    ├── create-vm/             # Download Ubuntu cloud image, gen cloud-init ISO, virt-install
    ├── nut-shutdown/          # Custom /usr/local/bin/vm-host-shutdown wired into upsmon SHUTDOWNCMD
    ├── pterodactyl-apache-proxy/  # Apache reverse proxy on Panel VM, fronts Wings :8443
    └── geerlingguy.nut_client/    # Vendored copy (gitignored at roles/geerlingguy.*/)
```

### Roles

| Role | Source | Purpose |
|---|---|---|
| `prereq` | local | Enable contrib/non-free repos, install base packages, dist-upgrade |
| `zfs` | **local** (not the shared `reenchree.common.zfs`) | Install ZFS, DKMS module, create RAIDZ1 pool. Hercules uses the shared role; pegasus predates that and is intentionally kept on the local copy. |
| `network` | local | networkd as sole owner (masks NetworkManager, ifupdown lo-only), `primary_interface` → `br-vlan50` (DHCP, pinned `bridge_mac`, `IgnoreCarrierLoss`), wait-online timeout 60s, static `/etc/resolv.conf` (systemd-resolved not installed). Bridge MAC changes need the bridge deleted before `networkctl reload`. |
| `libvirt` | local | qemu-kvm + libvirt-daemon-system + define storage pool at `/vmpool` |
| `cockpit` | local | Cockpit + cockpit-machines on :9090 |
| `create-vm` | local | Download Ubuntu cloud image, gen cloud-init ISO, `virt-install` |
| `nut-shutdown` | local | `/usr/local/bin/vm-host-shutdown` (graceful `virsh shutdown` loop), patches `SHUTDOWNCMD` in `upsmon.conf` |
| `pterodactyl-apache-proxy` | local | Apache `mod_proxy` config on Panel VM so HTTPS Panel can call HTTP Wings without mixed-content errors; appends `127.0.0.1` to Wings `trusted_proxies` |
| `reenchree.common.node_exporter` | shared collection | node_exporter :9100 |
| `reenchree.common.zfs_exporter` | shared collection | zfs_exporter :9134 |
| `reenchree.common.smartctl_exporter` | shared collection | smartctl_exporter :9633 |
| `maxhoesel.pterodactyl.pterodactyl_panel` / `_wings` | external collection | Panel + Wings install inside the VMs |
| `geerlingguy.nut_client` | GitHub tag 2.0.0 (vendored) | NUT client daemon |

## Architecture

```
pegasus (192.168.50.100) — Debian 13 KVM/libvirt host (SSH: pi@)
├── ZFS RAIDZ1 pool "vmpool" (/dev/sda-d) → /vmpool
├── libvirt/KVM hypervisor (storage pool at /vmpool, VM disks under /vmpool/disks)
├── systemd-networkd (sole owner; NetworkManager masked)
│   └── br-vlan50    — enp5s0 (VLAN 50 access port) bridge; host DHCP 192.168.50.100
│                      (MAC pinned a8:a1:59:5f:b6:4a) + VM NICs, gw 192.168.50.1
├── NUT client → secondary to NUT server 192.168.0.251 (UPS `ups`, user `observer`)
│                  graceful VM shutdown via /usr/local/bin/vm-host-shutdown on power-fail
├── Cockpit web UI :9090
└── Prometheus exporters (scraped by kube-prom-stack in sea-k8s-flux when host is up)
    ├── node_exporter      :9100 (job `bare-metal-node`,  instance=pegasus)
    ├── zfs_exporter       :9134 (job `bare-metal-zfs`,   instance=pegasus)
    └── smartctl_exporter  :9633 (job `bare-metal-smart`, instance=pegasus, 300s scrape)

VMs (on br-vlan50, Ubuntu 24.04 cloud-init, SSH: pi@)
├── pterodactyl   192.168.50.30   4 vCPU / 8 GiB  / 250 GiB   Panel + Wings + MariaDB (Wings on :8888)
└── node-2        192.168.50.31   4 vCPU / 16 GiB / 250 GiB   Wings only (default :8080)
```

External access: Traefik in `sea-k8s-flux` fronts the Panel at `https://pterodactyl.reenchree.dev`. The Panel uses `selfsign` SSL mode internally; Traefik handles public TLS.

## Cross-repo links

- **`../reenchree-ansible-common`** — shared collection providing `node_exporter`, `zfs_exporter`, `smartctl_exporter` roles. `requirements.yml` pins it to tag `v1.8.0`. The collection also has `base`, `zfs`, `sanoid`, `nut_server` roles that pegasus deliberately does **not** use (pegasus keeps its own local `zfs` role and has no sanoid snapshots — VMs are stateless game-server hosts, no precious data to snapshot).
- **`../sea-k8s-flux`** — `infra/kube-prom-stack/helm-release.yaml` defines the bare-metal scrape jobs; pegasus targets were removed at decommission (G1). If you add a new exporter port on pegasus, add a matching scrape entry there.
- **`../sea-hercules-ansible`** — sibling repo for the other bare-metal host. Same shared-collection pattern but uses many more shared roles and runs much more on the host directly (NAS, Garage S3, FreePBX VM, Semaphore VM).

## Common Commands

For production runs, use Semaphore. Local invocations:

```bash
# Install Galaxy dependencies (required before first run)
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# Test connectivity
ansible vm_hosts -m ping

# Full host provisioning (skips network role by default via 'never' tag)
ansible-playbook playbooks/site.yml

# Check system health
ansible-playbook playbooks/check-system.yml

# Network changes (DANGEROUS — requires physical/console access)
ansible-playbook playbooks/network-only.yml
ansible-playbook playbooks/network-rollback.yml   # emergency revert to ifupdown

# Deploy VMs
ansible-playbook playbooks/pterodactyl.yml
ansible-playbook playbooks/node-2.yml
ansible-playbook playbooks/pterodactyl-teardown.yml

# Power protection
ansible-playbook playbooks/nut-client.yml

# Patching
ansible-playbook playbooks/apt-upgrade.yml         # vm_hosts, notify-only
ansible-playbook playbooks/apt-upgrade-vms.yml     # vms, serial:1, auto-reboot, verifies wings

# Pterodactyl lifecycle
ansible-playbook playbooks/pterodactyl-backup.yml
ansible-playbook playbooks/pterodactyl-update.yml

# Standalone exporter redeploys
ansible-playbook playbooks/node_exporter.yml
ansible-playbook playbooks/zfs_exporter.yml

# Targeted runs
ansible-playbook playbooks/site.yml --check
ansible-playbook playbooks/site.yml --tags zfs,libvirt
```

## Key Design Decisions

- **inventory.yml is committed to git** — secrets are injected at runtime via Semaphore environment variables (see "Inventory & Secrets" below). README.md still says inventory.yml is gitignored; that's stale — CLAUDE.md is authoritative.
- **Network role is tagged `never`** in `site.yml` to prevent accidental network disruption. Use `network-only.yml` to apply network changes explicitly, and have console access ready.
- **`ansible.cfg` sets `become = True`** globally — all tasks run with sudo by default.
- **VM provisioning uses cloud-init** — `create-vm` downloads Ubuntu cloud images, generates a cloud-init ISO, and runs `virt-install`. VMs are fully configured on first boot.
- **Pipelining is enabled** in ansible.cfg for faster execution.
- **No sanoid / no ZFS snapshots** — unlike hercules, the pegasus `vmpool` is intentionally unsnapshotted. Game-server VMs are treated as cattle; persistent state (Panel DB, Panel `.env`) is backed up via `pterodactyl-backup.yml`.

## Role Dependency Chain

**Host setup** (`site.yml`):
`prereq` → `reenchree.common.node_exporter` → `reenchree.common.smartctl_exporter` → `zfs` → `reenchree.common.zfs_exporter` → `network` (skipped — `never` tag) → `libvirt` → `cockpit`

**VM deployment** (`pterodactyl.yml`):
1. On pegasus: `create-vm` (provisions VM)
2. On the VM: MariaDB setup → `maxhoesel.pterodactyl.pterodactyl_panel` → `maxhoesel.pterodactyl.pterodactyl_wings` → `pterodactyl-apache-proxy`

**Node-2 deployment** (`node-2.yml`):
1. On pegasus: `create-vm`
2. On the VM: `maxhoesel.pterodactyl.pterodactyl_wings` only

**Power protection** (`nut-client.yml`): `nut-shutdown` → `geerlingguy.nut_client` (in that order — `nut-shutdown` lays down the shutdown script + `SHUTDOWNCMD` line, then `geerlingguy.nut_client` configures the rest).

## Inventory Structure

No `group_vars/` or `host_vars/` directories — everything lives in `inventory.yml` under two groups:

- **`vm_hosts`** — physical host config (ZFS pool, network interfaces, NUT client, libvirt storage pool)
- **`vms`** — VM definitions with per-host vars (vm_name, vcpus, ram, disk, IP, Pterodactyl config). Common SSH/Python settings live in `vms.vars`.

The `vms` group hardcodes `ansible_ssh_private_key_file: ~/.ssh/id_ansible`; vm_hosts inherits the workstation default.

## Inventory & Secrets

The inventory is **`inventory.yml` committed to git** — Semaphore pulls it from the repo on each run (project ID **2**, inventory ID 2, type `file`). To update inventory variables: edit `inventory.yml`, commit, push.

**IMPORTANT: No secrets in `inventory.yml`.** Secrets are injected at runtime via the Semaphore "Default" environment (ID **4**) as extra vars, which override inventory values. The following secrets live in the environment, NOT in git:

- `nut_client_password`
- `pterodactyl_panel_db_password`
- `pterodactyl_panel_app_key`
- `pterodactyl_panel_hashids_salt`
- `mysql_root_password`

To update a secret, use the Semaphore API: `PUT /api/project/2/environment/4` with the updated `json` field, or edit it in the UI under Sea Pegasus VM Host → Environment → Default.

## Semaphore Integration

This repo is managed as project **"Sea Pegasus VM Host"** (project ID 2) at https://semaphore.reenchree.dev. When adding a new playbook, also create a matching Semaphore task template so it can be run from the UI.

Auth: use the API token at `~/.config/semaphore/token` (mode 0600) as a `Bearer` header — same pattern as the hercules repo. See `../sea-hercules-ansible/CLAUDE.md` for the full token mint / common-operations recipe.

### Task Templates (project ID 2)

| Template | ID | Playbook |
|---|---|---|
| Check System Health | 1 | `playbooks/check-system.yml` |
| Deploy Node Exporter | 2 | `playbooks/node_exporter.yml` |
| Deploy Node-2 VM | 3 | `playbooks/node-2.yml` |
| Deploy Pterodactyl VM | 4 | `playbooks/pterodactyl.yml` |
| Deploy ZFS Exporter | 5 | `playbooks/zfs_exporter.yml` |
| Host Provisioning (Full) | 6 | `playbooks/site.yml` |
| NUT Client Setup | 7 | `playbooks/nut-client.yml` |
| Network Configuration | 8 | `playbooks/network-only.yml` |
| Network Rollback | 9 | `playbooks/network-rollback.yml` |
| Pterodactyl Backup (Daily) | 10 | `playbooks/pterodactyl-backup.yml` |
| Pterodactyl Update (Panel + Wings) | 11 | `playbooks/pterodactyl-update.yml` |
| Teardown Pterodactyl VM | 12 | `playbooks/pterodactyl-teardown.yml` |
| VM Apt Upgrade (Weekly) | 13 | `playbooks/apt-upgrade-vms.yml` |
| VM Host Apt Upgrade | 14 | `playbooks/apt-upgrade.yml` |

### Adding a new playbook

```bash
TOK=$(cat ~/.config/semaphore/token)
H="Authorization: Bearer $TOK"

# Discover the resource IDs once (Sea Pegasus VM Host = project 2)
curl -H "$H" https://semaphore.reenchree.dev/api/project/2/inventory
curl -H "$H" https://semaphore.reenchree.dev/api/project/2/repositories
curl -H "$H" https://semaphore.reenchree.dev/api/project/2/environment

# Create the template
curl -H "$H" -X POST https://semaphore.reenchree.dev/api/project/2/templates \
  -H 'Content-Type: application/json' \
  -d '{"project_id":2,"inventory_id":<ID>,"repository_id":<ID>,"environment_id":4,
       "name":"<Template Name>","playbook":"playbooks/<name>.yml",
       "description":"<description>","allow_override_args_in_task":true,"app":"ansible"}'
```

## External Dependencies (requirements.yml)

| Dep | Pin | Purpose |
|---|---|---|
| `community.general` | `>=7.0.0` | General modules (`apache2_module`, `modprobe`, …) |
| `community.libvirt` | `>=1.0.0` | `virt`, `virt_pool`, `virt_net` modules used by `create-vm` and `nut-shutdown` |
| `maxhoesel.pterodactyl` | `5.0.0` | Pterodactyl Panel + Wings roles, installed inside the VMs |
| `reenchree.common` | git tag **`v1.8.0`** | Shared monitoring exporter roles (see `../reenchree-ansible-common`). Pinned — bump deliberately. |
| `geerlingguy.nut_client` | git tag `2.0.0` | NUT client (vendored under `roles/geerlingguy.nut_client/`, gitignored at `roles/geerlingguy.*/`) |

## Development Principles

- **All changes go through roles** — never fix things by SSHing in and running one-off commands. The playbooks are the single source of truth for pegasus state.
- **README.md is stale** in several places (says inventory is gitignored, says ZFS is a 2-disk mirror, lists only the Step 1/2/3 deploy flow). This `CLAUDE.md` is authoritative.
- Before running anything that touches `network` or reboots the host, remember pegasus has been **powered off** since the 2026-10-14 decommission (see banner). Wake-on-LAN / physical-press is required to bring it back; nothing in this repo automates that.

## Decommission record

Decommissioned 2026-10-14 (see banner). Everything above this section is historical reference.

- **Semaphore project 2** ("Sea Pegasus VM Host") is deleted 2026-10-14 (Semaphore DB preserved in Hercules snapshot `tank/vms@pre-sem-project2-delete-2026-10-14`). A `tank/vms` snapshot on Hercules is taken first; record its name here when done.
- **`playbooks/network-rollback.yml` is stale and unsafe.** It reverts to ifupdown, which fights systemd-networkd on the current VLAN 50 access-port layout. Do not run it.
- **Former Semaphore templates (project 2, 14):**

| Template | ID | Playbook |
|---|---|---|
| Check System Health | 1 | `playbooks/check-system.yml` |
| Deploy Node Exporter | 2 | `playbooks/node_exporter.yml` |
| Deploy Node-2 VM | 3 | `playbooks/node-2.yml` |
| Deploy Pterodactyl VM | 4 | `playbooks/pterodactyl.yml` |
| Deploy ZFS Exporter | 5 | `playbooks/zfs_exporter.yml` |
| Host Provisioning (Full) | 6 | `playbooks/site.yml` |
| NUT Client Setup | 7 | `playbooks/nut-client.yml` |
| Network Configuration | 8 | `playbooks/network-only.yml` |
| Network Rollback | 9 | `playbooks/network-rollback.yml` |
| Pterodactyl Backup (Daily) | 10 | `playbooks/pterodactyl-backup.yml` |
| Pterodactyl Update (Panel + Wings) | 11 | `playbooks/pterodactyl-update.yml` |
| Teardown Pterodactyl VM | 12 | `playbooks/pterodactyl-teardown.yml` |
| VM Apt Upgrade (Weekly) | 13 | `playbooks/apt-upgrade-vms.yml` |
| VM Host Apt Upgrade | 14 | `playbooks/apt-upgrade.yml` |

- **Former schedules (3, all inactive):** "Weekly Sunday 4:00 AM" (`0 4 * * 0`, template 13); "Daily 3:00 AM" (`0 3 * * *`, template 10); "Weekly Sunday 4:30 AM" (`30 4 * * 0`, template 14).
