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

# Apt upgrade VM host (reboots if required, verifies libvirtd)
ansible-playbook playbooks/apt-upgrade.yml

# Apt upgrade VMs (serial:1, reboots if required, verifies wings)
ansible-playbook playbooks/apt-upgrade-vms.yml

# Backup Pterodactyl database and .env config
ansible-playbook playbooks/pterodactyl-backup.yml

# Update Pterodactyl Panel and Wings to latest
ansible-playbook playbooks/pterodactyl-update.yml

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
├── Cockpit web UI (:9090)
└── Monitoring exporters (scraped by Prometheus in k8s)
    ├── node_exporter (:9100)
    ├── zfs_exporter (:9134)
    └── smartctl_exporter (:9633)

VMs (on VLAN 50, Ubuntu 24.04 cloud-init)
├── pterodactyl (192.168.50.30) - Panel + Wings + MariaDB
└── node-2 (192.168.50.31) - Wings only
```

External access: Traefik reverse proxy (in separate sea-k8s-flux repo) → pterodactyl.reenchree.dev

## Key Design Decisions

- **inventory.yml is committed to git** — secrets are injected at runtime via Semaphore environment variables (see "Inventory & Secrets" below).
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

## Inventory & Secrets

The inventory is **`inventory.yml` committed to git** — Semaphore pulls it from the repo on each run (inventory ID 2, type `file`). To update inventory variables, edit `inventory.yml`, commit, and push.

**IMPORTANT: No secrets in inventory.yml.** Secrets are injected at runtime via the Semaphore "Default" environment (ID 4) as extra vars, which override inventory values. The following secrets live in the environment, NOT in git:
- `nut_client_password`
- `pterodactyl_panel_db_password`
- `pterodactyl_panel_app_key`
- `pterodactyl_panel_hashids_salt`
- `mysql_root_password`

To update a secret, use the Semaphore API: `PUT /api/project/2/environment/4` with the updated `json` field.

## Semaphore Integration

This repo is managed as a project in Semaphore at https://semaphore.reenchree.dev (project: "Sea Pegasus VM Host"). When adding a new playbook, also create a matching Semaphore task template so it can be run from the UI.

To add a task template, authenticate and POST to the API:

```bash
# Authenticate (get credentials from k8s secret)
USERNAME=$(kubectl get secret semaphore-secrets -n semaphore -o jsonpath='{.data.username}' | base64 -d)
PASSWORD=$(kubectl get secret semaphore-secrets -n semaphore -o jsonpath='{.data.password}' | base64 -d)
curl -s -X POST https://semaphore.reenchree.dev/api/auth/login \
  -H 'Content-Type: application/json' \
  -d "{\"auth\": \"$USERNAME\", \"password\": \"$PASSWORD\"}" \
  -c /tmp/semaphore-cookies.txt

# Get the project ID, inventory ID, repo ID, and environment ID
curl -s -b /tmp/semaphore-cookies.txt https://semaphore.reenchree.dev/api/projects
# Find "Sea Pegasus VM Host" and note its id, then query its resources:
# curl -s -b /tmp/semaphore-cookies.txt https://semaphore.reenchree.dev/api/project/$PROJECT_ID/inventory
# curl -s -b /tmp/semaphore-cookies.txt https://semaphore.reenchree.dev/api/project/$PROJECT_ID/repositories
# curl -s -b /tmp/semaphore-cookies.txt https://semaphore.reenchree.dev/api/project/$PROJECT_ID/environment

# Create the task template
curl -s -b /tmp/semaphore-cookies.txt -X POST \
  https://semaphore.reenchree.dev/api/project/$PROJECT_ID/templates \
  -H 'Content-Type: application/json' \
  -d '{"project_id": '$PROJECT_ID', "inventory_id": '$INVENTORY_ID',
       "repository_id": '$REPO_ID', "environment_id": '$ENV_ID',
       "name": "<Template Name>", "playbook": "playbooks/<name>.yml",
       "description": "<description>",
       "allow_override_args_in_task": true, "app": "ansible"}'

# Cleanup
rm /tmp/semaphore-cookies.txt
```

## External Dependencies (requirements.yml)

- `community.general` — general Ansible modules
- `community.libvirt` — libvirt/virsh modules
- `maxhoesel.pterodactyl` — Pterodactyl Panel and Wings installation
- `geerlingguy.nut_client` — NUT UPS client (installed from GitHub master branch)
- `reenchree.common` — shared collection with node_exporter, zfs_exporter, smartctl_exporter roles
