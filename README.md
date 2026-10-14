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

# sea-pegasus-ansible

Ansible for **pegasus** (`192.168.50.100`), a Debian 13 (trixie) KVM/libvirt
host that runs Pterodactyl game-server VMs. Manages ZFS storage, VLAN-50
`systemd-networkd` bridging, cloud-init VM provisioning, and the Pterodactyl
Panel + Wings stack inside guest VMs.

See [CLAUDE.md](CLAUDE.md) for the authoritative, in-depth reference.

## Status

Decommissioned 2026-10-14 (see banner). Powered off; Prometheus scrape
targets removed from sea-k8s-flux. Bringing it back requires physical
power-on / WoL — nothing here automates that.

## Host

- **pegasus**, Debian 13 trixie, SSH `pi@192.168.50.100`
- **Storage**: ZFS **RAIDZ1** pool `vmpool` on 4x SATA disks (`/dev/sd[a-d]`),
  mounted at `/vmpool` and used as the libvirt storage pool path.
- **Network**: `enp5s0` on a VLAN 50 access port, sole port of `br-vlan50`
  (host DHCP + VMs). See CLAUDE.md.

## Inventory & secrets

`inventory.yml` **is committed to git** (there is no sample file). It holds
**no secrets** — `nut_client_password`, `pterodactyl_panel_db_password`,
`pterodactyl_panel_app_key`, `pterodactyl_panel_hashids_salt`, and
`mysql_root_password` are injected at runtime via the Semaphore "Default"
environment (ID 4). Production runs go through Semaphore project
**"Sea Pegasus VM Host"** (ID 2).

## Usage

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

ansible vm_hosts -m ping                       # connectivity
ansible-playbook playbooks/site.yml            # full host provisioning
ansible-playbook playbooks/check-system.yml    # health check
```

The `network` role is `never`-tagged in `site.yml`; apply it explicitly with
`playbooks/network-only.yml` **only with console access** (the
`systemd-networkd` cutover can drop SSH). `playbooks/network-rollback.yml`
reverts to ifupdown in an emergency.

Other key playbooks: `pterodactyl.yml` / `node-2.yml` (create VMs),
`pterodactyl-backup.yml` (DB + `.env` backup), `pterodactyl-update.yml`
(Panel + Wings update), `apt-upgrade.yml` (host, notify-only),
`apt-upgrade-vms.yml` (VMs, serial reboot), `nut-client.yml` (UPS shutdown).

## Roles

Local: `prereq`, `zfs`, `network`, `libvirt`, `cockpit`, `create-vm`,
`nut-shutdown`, `pterodactyl-apache-proxy`. Shared:
`reenchree.common.{node_exporter,zfs_exporter,smartctl_exporter}`. External:
`maxhoesel.pterodactyl` (Panel + Wings), `geerlingguy.nut_client`.
