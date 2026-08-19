# homelab-infra

Ansible playbooks for provisioning and configuring VMs across my 4-node Proxmox cluster, orchestrated through Semaphore running on a Raspberry Pi. This is the infrastructure-automation foundation for a broader homelab security portfolio — SIEM, honeypot, and Active Directory attack-and-defend labs all build on top of what's provisioned here.

## What's here

- `ansible.cfg` — base Ansible configuration
- `inventory/hosts.ini.example` — template inventory showing the expected structure (proxmox_nodes group, connection vars). Copy to `inventory/hosts.ini` and fill in real values — that file is gitignored and never committed.
- `playbooks/provision.yml` — provisions one or more VMs from a Proxmox template, automatically picking the least-loaded cluster node (by CPU/memory) when `target_node` is set to `auto`. Delegates the per-VM work to `provision-single.yml`, then installs the Tanium client via `tanium-client.yml`.
- `playbooks/provision-single.yml` — the per-VM logic used by `provision.yml`: scores each node's load, clones the template via the Proxmox REST API, waits for the new VM to report an IP through the QEMU guest agent, and adds it to a dynamic `new_vms` inventory group.
- `playbooks/provision-and-install.yml` — a self-contained alternative that provisions VMs using the `community.general.proxmox_kvm` module against a fixed node and installs Tanium inline. Overlaps significantly with `provision.yml` + `provision-single.yml`; **not yet consolidated — see To Do below.**
- `playbooks/tanium-client.yml` — installs and configures the Tanium endpoint client on target VMs (sets hostname, installs the client bundle, starts the service).

## Requirements

- Ansible with the `community.general` collection installed (`ansible-galaxy collection install community.general`)
- A Proxmox API token (see Setup below)
- SSH access to Proxmox nodes as configured in your inventory
- A Tanium client install bundle, supplied separately (proprietary — not included in this repo)

## Setup

1. Copy `inventory/hosts.ini.example` to `inventory/hosts.ini` and fill in your actual node IPs/hostnames.
2. This repo intentionally does **not** hardcode Proxmox host/node values inside the playbooks. When running through Semaphore, set an Environment (Extra Variables, JSON) on the relevant Task Templates:
```json
   {
     "proxmox_host": "<your proxmox API endpoint>",
     "target_node": "<a real node name>",
     "template_node": "<a real node name>",
     "proxmox_nodes": ["<node1>", "<node2>", "<node3>", "<node4>"]
   }
```
   Running outside Semaphore (plain `ansible-playbook`), pass the same values with `-e`.

## Security notes

- Real inventory (`inventory/hosts.ini`) and all real infrastructure values (node IPs, hostnames) are deliberately kept out of git — either gitignored locally or injected at runtime via Semaphore's Environment feature. Only placeholder/example values are committed.
- The Proxmox API token currently used (`root@pam!provisioner`) is tied to the root account. Moving to a dedicated, least-privilege PVE user/role is a planned improvement (see To Do).

## To do

- [ ] Decide between `provision-and-install.yml` and `provision.yml`/`provision-single.yml` — consolidate into one supported provisioning path.
- [ ] Move the Proxmox API token off `root@pam` onto a scoped, least-privilege user.
- [ ] Add Ansible roles for Wazuh agent + Sysmon deployment (feeds the `home-siem-lab` project).
- [ ] Add a role for automated GOAD/Ludus range deployment (feeds `ad-attack-and-defend`).