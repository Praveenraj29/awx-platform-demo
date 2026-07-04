# AWX Platform Demo

Self-service infrastructure provisioning platform — ServiceNow catalog → AWX → Ansible → VMware.

## Project 1 — LVM Filesystem Provisioning (current)

Self-service storage provisioning via Ansible role. User requests a mount point and size — automation handles PV, VG, LV, filesystem, mount, and fstab entry. Fully end-to-end tested.

## Architecture
## Stack

- Ansible Core 2.20
- AWX 24.6.1 (k3s-hosted, homelab ESXi)
- RHEL 9.5 target VM
- ServiceNow (PDI instance)
- Collections: community.general, ansible.posix, community.vmware
- Cloudflare Tunnel (Snow → AWX connectivity)

## Sprints

| Sprint | Status |
|---|---|
| 1 — LVM provisioning role | ✅ Done |
| 2 — AWX Job Template + Survey | ✅ Done |
| 3 — ServiceNow catalog → AWX variables end to end | ✅ Done |
| 4 — CMDB sync + dynamic VM dropdown | 🔄 In progress |
| 5 — Approval gate (VM owner approves) | ⬜ Pending |
| 6 — Closure loop + RITM auto-close + email | ⬜ Pending |

## What it does

1. Linux admin opens ServiceNow self-service catalog
2. Fills form: mount point, LV name, size, filesystem type
3. Business Rule fires → REST Message (OAuth2) → AWX API
4. AWX Ansible role runs against target VM
5. Filesystem provisioned, mounted, fstab entry added
6. RITM work notes updated with job status

## Key engineering decisions

- Role discovers available disks automatically (no manual disk specification)
- Uses VG free space before requesting new disk
- Fails cleanly with human-readable messages when resources unavailable
- Idempotent — safe to run multiple times
- Cross-platform: works on RHEL/Rocky/Ubuntu

## Roadmap

- Project 2: HashiCorp Vault SSH CA — ephemeral credentials, no static keys
- Project 3: Self-maintaining AWX platform — GitOps EE builds and job template management
- Project 4: Backstage + Keycloak — persona-aware developer portal
- OpenShift Virtualization as parallel hypervisor target alongside VMware

## Author

Praveenraj — DevOps/Platform Engineering
