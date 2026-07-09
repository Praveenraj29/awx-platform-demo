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
| 5 — Disk attach via pyvmomi | ✅ Done | Auto-scale provisioning: checks VG free space, attaches new disk via ESXi API only when needed, extends VG, creates LV/filesystem/mount |
| 6 — AWX multi-step Workflow | ✅ Done | Native AWX Workflow Visualizer: Check Capacity → branches to Attach Disk (on failure) or straight to Provision Filesystem (on success) |
| 7 — Credentials hardening | ⬜ Pending | Move ESXi/ServiceNow secrets from plaintext job template vars into AWX Credentials |
| 8 — Personas + Approval gate | ⬜ Pending | Developer requests → VM owner approves → AWX runs |
| 9 — Closure loop | ⬜ Pending | AWX webhook → RITM auto-closed + email notification |
| 10 — Docs + architecture diagram + video | ⬜ Pending | README, diagram, YouTube demo recording |

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

## Sprint 5 details (in progress)

Extends provisioning so a request can exceed currently free VG space —
the workflow now checks free space first, and only attaches a new virtual
disk via the ESXi API when genuinely needed.

- **Standalone disk-attach playbook**: `playbooks/attach_disk.yml`
  — proven working via `community.vmware.vmware_guest_disk`
- **Known gotcha (standalone ESXi only, not vCenter-managed)**: new disks
  must be placed on the **same datastore** as the VM's existing disks.
  Cross-datastore placement fails with `File ... was not found` — appears
  to be a module limitation specific to non-vCenter-managed hosts.
- **Integrated playbook**: `playbooks/provision_with_autoscale.yml`
  — checks `vgs` free space, decides whether a new disk is needed,
  dynamically looks up the next free controller unit number and existing
  datastore (not hardcoded), attaches if needed, extends the VG, then
  creates the LV/filesystem/mount as before
- **Status**: disk-attach mechanics are tested and working end-to-end
  (verified via `lsblk` on the target VM). The VG-extend / auto-scale
  branch is written but not yet exercised by a real test request —
  next session should deliberately request more space than currently
  free to validate this path.

## Sprint 5 details

Extends provisioning so a request can exceed currently free VG space.
The workflow checks free space first, and only attaches a new virtual
disk via the ESXi API when genuinely needed.

- **Discovery playbook**: `playbooks/discover_disk_attach.yml` —
  probes existing disk/controller layout and real datastore free space
  before writing any attach logic
- **Standalone disk-attach**: `playbooks/attach_disk.yml` — proven via
  `community.vmware.vmware_guest_disk`
- **Integrated auto-scale playbook**: `playbooks/provision_with_autoscale.yml`
  — checks `vgs` free space, decides whether a new disk is needed,
  dynamically looks up the next free controller unit number and
  existing datastore, attaches if needed, extends the VG, then creates
  the LV/filesystem/mount
- **Verified end-to-end**: requested 20GB against a VG with only
  14.99GB free — correctly attached a 7GB disk, extended the VG,
  created a 20GB LV spanning both the original and new disk. Math
  checked out exactly (14.99 + 7 - 20 = 1.99GB free, matched VG's
  reported free space precisely)

### Known gotchas (standalone ESXi only, not vCenter-managed)

- New disks must be placed on the **same datastore** as the VM's
  existing disks — cross-datastore placement fails with
  `File ... was not found`, a module limitation specific to
  non-vCenter-managed hosts
- `delegate_to: localhost` tasks need `become: false` explicitly —
  otherwise they inherit the play's `become: yes` and fail trying to
  `sudo` on the AWX execution container, which does not have sudo installed
- Avoid self-referencing `default()` in `vars:` — causes infinite
  recursion when no higher-precedence source (like extra vars) defines
  the variable first
- New-disk identification uses a before/after snapshot diff of `sd*`
  device names rather than heuristics (partition/removable/lvm-master
  checks) — the heuristic approach misidentified an existing LVM
  device (`dm-3`) as the new disk in testing

## Sprint 6 details

Built as a native AWX Workflow using the Workflow Visualizer, chaining
three Job Templates with success/failure branching:
- **Check Capacity**: runs `vgs`, deliberately fails if free space is
  insufficient — this failure is the branch trigger (AWX Workflow
  Visualizer branches on job success/failure, not custom variable values)
- **Attach Disk**: generalized version of Sprint 5's disk-attach logic
  (`playbooks/attach_disk_dynamic.yml`) — no hardcoded VM name
- **Provision Filesystem**: existing job template, reused unchanged
- **Convergence limitation**: AWX's Workflow Visualizer doesn't cleanly
  support linking two different parent nodes to one existing shared child
  through the UI — ended up with two separate (functionally identical)
  Provision Filesystem nodes rather than a true visual diamond. Both
  paths tested and verified working independently.

### Two real bugs found and fixed during testing

Both were latent issues from Sprint 5 that only surfaced once Sprint 6
exercised code paths more thoroughly:

1. **LV size unit suffix missing** (`roles/filesystem_provision/tasks/03_provision.yml`)
   — `community.general.lvol`'s `size` parameter was passed as
   `{{ lv_size }}` with no unit, which defaults to megabytes/extents
   rather than gigabytes. Silently created 4-8MB volumes instead of the
   requested GB size across every prior test. Fixed by appending `G`.

2. **Destructive `lvg` default** (`playbooks/attach_disk_dynamic.yml`,
   `playbooks/provision_with_autoscale.yml`) — `community.general.lvg`
   defaults `remove_extra_pvs: true`, meaning specifying only the new
   disk as `pvs` caused the module to attempt removing every *other*
   PV from the VG (including the ones backing root/home/swap). Caught
   by a safe failure (PVs were in use), not by design — could have
   destroyed existing data on a VG where the other PVs weren't actively
   mounted. Fixed by explicitly setting `remove_extra_pvs: false`.

### Verified end-to-end (both branches)

- **Success path**: 5GB request against 14.99GB free → skipped Attach
  Disk, went straight to Provision Filesystem, correct 5GB LV created
- **Failure path**: 20GB request against 14.99GB free → correctly
  routed through Attach Disk (7GB disk attached, VG extended), then
  Provision Filesystem created a correct 20GB LV spanning both disks.
  Math checked out exactly (14.99 + 7 - 20 = 1.99GB free, matched)
