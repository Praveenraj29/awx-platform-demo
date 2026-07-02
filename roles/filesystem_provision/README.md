# filesystem_provision

Ansible role that provisions LVM-based filesystems on Linux VMs. Handles the full stack: PV → VG → LV → filesystem → mount → fstab.

## What it does

- Discovers available unallocated disks automatically
- Creates a new VG or extends an existing one based on current state
- Creates a logical volume, formats it, mounts it, and adds a persistent fstab entry
- Fails cleanly with human-readable messages when resources are insufficient

## Requirements

ansible-galaxy collection install -r requirements.yml

## Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| mount_point | yes | ~ | Where to mount e.g. /data/logs |
| lv_size | yes | ~ | Size e.g. 5G |
| lv_name | yes | ~ | LV name e.g. lv_logs |
| fstype | no | ext4 | Filesystem type: ext4 or xfs |
| vg_name | no | ~ | VG name - auto-generated if not supplied |
| disk_device | no | ~ | Disk to use - auto-discovered if not supplied |

## Example

ansible-playbook provision.yml -e "mount_point=/data/logs lv_name=lv_logs lv_size=4G"

## Tested on

- Rocky Linux 10.2
- Ubuntu 26.04 (control node)

## Author

Praveenraj
