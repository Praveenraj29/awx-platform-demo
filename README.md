# AWX Platform Demo

Self-service infrastructure provisioning platform — ServiceNow catalog → AWX → Ansible → VMware.

## Project 1 — LVM Filesystem Provisioning (current)

Self-service storage provisioning via Ansible role. User requests a mount point and size — automation handles PV, VG, LV, filesystem, mount, and fstab.

## Architecture

ServiceNow Catalog → AWX Job Template → Ansible Role → Target Linux VM

## Stack

- Ansible Core 2.20
- AWX (k3s-hosted)
- Rocky Linux 10.2 target
- Collections: community.general, ansible.posix, community.vmware

## Sprints

| Sprint | Status |
|---|---|
| 1 — LVM provisioning role | Done |
| 2 — AWX Job Template + Survey | In progress |
| 3 — ServiceNow to AWX variables | Pending |
| 4 — CMDB sync + dynamic VM dropdown | Pending |
| 5 — Approval gate | Pending |
| 6 — Closure loop + email | Pending |

## Roadmap

- Project 2: HashiCorp Vault SSH CA — ephemeral credentials
- Project 3: Self-maintaining AWX platform — GitOps EE builds and job template management
- Project 4: Backstage + Keycloak — persona-aware developer portal
- OpenShift Virtualization as parallel hypervisor target

## Author

Praveenraj — DevOps/Platform Engineering
