# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation for a Debian 13 KVM/libvirt VM host ("pegasus") running Pterodactyl game servers. Manages the full stack: ZFS storage, VLAN networking, VM provisioning via cloud-init, and Pterodactyl Panel + Wings installation.

## Common Commands

```bash
# Install Galaxy dependencies (required before first run)
ansible-galaxy install -r requirements.yml

# Test connectivity
ansible vm_hosts -m ping

# Full host provisioning (skips network role by default via 'never' tag)
ansible-playbook playbooks/site.yml

# Check system health
ansible-playbook playbooks/check-system.yml

# Network changes (DANGEROUS - requires physical/console access)
ansible-playbook playbooks/network-only.yml

# Deploy Pterodactyl VM (Panel + Wings)
ansible-playbook playbooks/pterodactyl.yml

# Deploy Wings-only node
ansible-playbook playbooks/node-2.yml

# Tear down Pterodactyl VM
ansible-playbook playbooks/pterodactyl-teardown.yml

# NUT UPS client setup
ansible-playbook playbooks/nut-client.yml

# Dry run any playbook
ansible-playbook playbooks/<name>.yml --check

# Run specific tags
ansible-playbook playbooks/site.yml --tags zfs,libvirt
```

## Architecture

```
pegasus (192.168.0.253) - Debian 13 VM Host
├── ZFS RAIDZ1 pool "vmpool" (/dev/sda-d) → /vmpool
├── libvirt/KVM hypervisor (storage pool at /vmpool)
├── systemd-networkd networking
│   ├── br0 (untagged bridge, host mgmt, DHCP 192.168.0.x)
│   └── br-vlan50 (VLAN 50 bridge, VMs, static 192.168.50.x)
├── NUT client → graceful VM shutdown on power failure
└── Cockpit web UI (:9090)

VMs (on VLAN 50, Ubuntu 24.04 cloud-init)
├── pterodactyl (192.168.50.30) - Panel + Wings + MariaDB
└── node-2 (192.168.50.31) - Wings only
```

External access: Traefik reverse proxy (in separate sea-k8s-flux repo) → pterodactyl.reenchree.dev

## Key Design Decisions

- **inventory.yml is gitignored** — contains secrets. Use `inventory-sample.yml` as template.
- **Network role is tagged `never`** in site.yml to prevent accidental network disruption. Use `network-only.yml` to apply network changes explicitly.
- **`ansible.cfg` sets `become = True`** globally — all tasks run with sudo by default.
- **VM provisioning uses cloud-init** — the `create-vm` role downloads Ubuntu cloud images, generates cloud-init ISO, and runs `virt-install`. VMs are fully configured on first boot.
- **Pipelining is enabled** in ansible.cfg for faster execution.

## Role Dependency Chain

**Host setup** (site.yml): `prereq` → `zfs` → `network` (skipped) → `libvirt` → `cockpit`

**VM deployment** (pterodactyl.yml):
1. On pegasus: `create-vm` (provisions VM)
2. On VM: MariaDB setup → `maxhoesel.pterodactyl` panel/wings roles → `pterodactyl-apache-proxy`

**Power protection** (nut-client.yml): `geerlingguy.nut_client` → `nut-shutdown` (custom shutdown script)

## Inventory Structure

No `group_vars/` or `host_vars/` directories — all variables live in `inventory.yml` under two groups:
- **vm_hosts**: Physical host config (ZFS, network, NUT, libvirt)
- **vms**: VM definitions with per-host vars (vm_name, vcpus, ram, disk, IP, Pterodactyl config)

## External Dependencies (requirements.yml)

- `community.general` — general Ansible modules
- `community.libvirt` — libvirt/virsh modules
- `maxhoesel.pterodactyl` — Pterodactyl Panel and Wings installation
- `geerlingguy.nut_client` — NUT UPS client (installed from GitHub master branch)
