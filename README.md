# blog-iscsi-multipath-manual

Companion code for the blog post
[Setting Up iSCSI Multipathing Manually on Linux](https://blog.bsdserver.nl/setting-up-iscsi-multipathing-manually-on-linux-2/).

The post walks through the steps by hand; this repository does the same steps with Ansible, and adds
the target the post assumes exists:

- **storage01**: a Linux LIO iSCSI target (`targetcli-fb`) with two portals, one on each storage
  network, exporting its second disk as LUN 0 (no CHAP, as in the post).
- **node01**: `open-iscsi` and `multipath-tools` (section 3), discovery and login on both portals with
  automatic login at boot (section 4), and `/etc/multipath.conf` with the `mpatha` alias, multibus and
  round-robin, blacklisting everything except the LUN's WWID (section 5).

Terraform creates the two Ubuntu 24.04 VMs on Proxmox VE, each with one NIC on each of two bridges.

## Requirements

- Terraform >= 1.9 and a Proxmox VE API token (the bpg/proxmox provider, pinned in `terraform/versions.tf`).
  Two bridges (one per storage network); the datastore for the cloud image needs the `import` content type.
- Ansible (ansible-core 2.17 or later; tested with 2.22). Only `ansible.builtin` modules are used.
- Ubuntu 24.04 on the target and the initiators.

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # endpoint, token, node, bridges, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible
ansible-playbook site.yml
ansible-playbook verify.yml    # the checks from section 6 of the post
```

No Proxmox? Copy `ansible/inventory.example.yml` to `ansible/inventory.yml` with your own hosts. The
target needs an unused disk and an address on each storage network.

Then try section 7 of the post: take one storage NIC of the initiator down while writing to
`/dev/mapper/mpatha` and watch `multipath -ll` mark the path failed and reinstate it afterwards.

## Variables

Set in `ansible/group_vars/all/vars.yml`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `iscsi_target_ips` | `10.0.10.11`, `10.0.20.11` | The two portals (IP1 and IP2 in the post) |
| `iscsi_target_name` | `iqn.2025-03.com.example:storage` | Target IQN |
| `iscsi_wwid` | computed from the LUN `wwn` | WWID in the `blacklist_exceptions` and `multipaths` sections |
| `iscsi_target_luns` | `/dev/sdb` as `lun0` | Block devices exported by the target, with a fixed `wwn` (UUID) |

The target gives the LUN a fixed serial, so the WWID is known before the initiator sees the disk:
`36001405` followed by the first 25 hex digits of the `wwn` without dashes. With your own target, set
`iscsi_wwid` to the value of `/lib/udev/scsi_id -g -u -d /dev/sdX` for one of the iSCSI paths.

The discovery task uses `iscsiadm -m discovery ... -o new`: a plain re-discovery rewrites existing node
records and resets `node.startup` back to `manual`.

## Tested

Lab-tested on Ubuntu 24.04 (kernel 6.8) with ansible-core 2.22 and two portals: the first run configures
both ends, the second run reports `changed=0`, `verify.yml` passes (one session per portal, `mpatha` with
two `active ready running` paths) and both node records have `node.startup = automatic`. Terraform is
checked with `fmt -check`, `validate` and `tflint`; `terraform apply` was not run as part of these tests.

## License

MIT
