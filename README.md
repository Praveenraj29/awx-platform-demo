# AWX Platform Demo

Self-service infrastructure provisioning platform — ServiceNow catalog → AWX → Ansible → VMware.

## Project 1 — LVM Filesystem Provisioning (current)

Self-service storage provisioning via Ansible role. User requests a mount point and size — automation handles disk attachment, PV, VG, LV, filesystem, mount, and fstab entry. Fully end-to-end tested.

## Architecture

Two flows, one platform:
- **Scheduled**: AWX pulls VM data from ESXi → syncs into ServiceNow CMDB table (keeps catalog dropdown current)
- **On-demand**: ServiceNow catalog request → Business Rule resolves target VM's IP from CMDB → AWX dynamically targets and provisions

## Stack

- Ansible Core 2.20
- AWX 24.6.1 (k3s-hosted, homelab ESXi)
- RHEL 9.5 target VM
- ServiceNow PDI instance
- VMware ESXi 7.0 (standalone homelab)
- Collections: community.general, ansible.posix, community.vmware
- Cloudflare Tunnel (Snow → AWX connectivity)
- OAuth2 authentication (Snow → AWX)

## Sprints

| Sprint | Status | Goal |
|---|---|---|
| 1 — LVM provisioning role | ✅ Done | Idempotent Ansible role: PV→VG→LV→mkfs→mount→fstab |
| 2 — AWX Job Template + Survey | ✅ Done | Role wrapped in AWX, survey-driven variables |
| 3 — Snow catalog → AWX variables | ✅ Done | End-to-end: Snow form → Business Rule → OAuth → AWX → VM |
| 4 — CMDB sync + dynamic VM dropdown | ✅ Done | ESXi VMs → Snow custom table (OAuth) → catalog Reference field, filtered to powered-on VMs, dynamic AWX host targeting via add_host |
| 5 — Disk attach via pyvmomi | ⬜ Pending | vmware_guest_disk → attach disk to VM via ESXi API |
| 6 — AWX multi-step Workflow | ⬜ Pending | Discover → Attach Disk → Provision Filesystem chained |
| 7 — Personas + Approval gate | ⬜ Pending | Developer requests → VM owner approves → AWX runs |
| 8 — Closure loop | ⬜ Pending | AWX webhook → RITM auto-closed + email notification |
| 9 — Docs + architecture diagram + video | ⬜ Pending | README, diagram, YouTube demo recording |

## What it does today (Sprint 4 complete)

**Background — CMDB sync (every 15 min):**
1. AWX pulls live VM inventory from ESXi (`vmware_vm_info`)
2. Syncs into ServiceNow custom table `u_vmware_virtual_machine` via OAuth-authenticated REST calls
3. Catalog's VM picker always reflects current inventory, filtered to powered-on VMs

**On-demand — provisioning request:**
1. Linux admin opens ServiceNow self-service catalog
2. Fills form: target VM (picked from live CMDB data), mount point, LV name, size, filesystem type
3. Business Rule looks up the selected VM's IP from the CMDB table, fires REST Message (OAuth2) → AWX API
4. AWX dynamically registers the target IP as an in-memory host (`add_host`) — no static inventory maintenance required
5. Ansible role runs against the resolved target VM
6. Filesystem provisioned, mounted, fstab entry added permanently
7. RITM work notes updated with job status and resolved target IP

## Key engineering decisions

- Role discovers available disks automatically — no manual disk specification needed
- Uses VG free space before requesting a new disk (two-tier decision logic)
- Fails cleanly with human-readable messages when resources unavailable
- Idempotent — safe to run multiple times without side effects
- Cross-platform: works on RHEL, Rocky Linux, Ubuntu

## Project Roadmap

| Project | Focus |
|---|---|
| Project 1 (current) | ServiceNow + AWX + Ansible + VMware end-to-end |
| Project 2 | HashiCorp Vault — ephemeral SSH CA, zero static credentials |
| Project 3 | Self-maintaining AWX — GitOps EE builds, auto job template registration |
| Project 4 | Backstage + Keycloak — persona-aware developer portal + SSO |

## Author

Praveenraj — DevOps / Platform Engineering
