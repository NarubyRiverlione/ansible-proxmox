# Proxmox Management

## Development Setup

To create and activate a project virtual environment and install dependencies, run:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

This directory contains Ansible playbooks and inventory for managing Proxmox on your homelab cluster, including CTs and VMs. It defines a two-step authentication flow:

1. **Bootstrapping**: connect as `root` to create the user defined by `guest_user`
2. **Day-to-day operations**: connect as the CT user (from `guest_user`) for container and update tasks

---

## Inventory and Variables

- **Inventory file**: `inventory.yml`
  - Hosts are listed under the `guests` group. Example with optional constructed group:
    ```yaml
    all:
      children:
        guests:
          hosts:
            ct-01.naruby.local:
              type: ct
              ct_user: naruby
              vmid: 106
            ct-02.naruby.local:
              type: ct
              ct_user: naruby
              vmid: 109
        constructed:
          groups:
            type_ct: "type is defined and type == 'ct'"
    ```
    - `type_ct` is an optional constructed group that dynamically matches hosts where `type == 'ct'`. If you do not use the constructed plugin, you can still filter by type using the loop-level variable in the power task: `-e loop_type=ct`.
- **Group vars**: place common vars under `group_vars/all/`. Plays set Proxmox API vars at the play level.

---

## Playbooks & Execution

The CT lifecycle is orchestrated via the umbrella playbook:

```bash
ansible-playbook main.yml
```

`main.yml` uses modular tasks and tags. Key plays/tasks and how to filter:

- Unified power management (CTs/VMs): tag `power` (uses `tasks/power.yml`)
  - Defaults: `desired_state=started`, `target_group=guests`
  - Normalized states: you can pass `start|stop` or `started|stopped` (both are accepted).
  - Loop-level filtering: you can limit by host variable `type` using `-e loop_type=ct` (or `vm`).
  - Examples:
    - Start all guests:
      `ansible-playbook -i inventory.yml main.yml -t power -e target_group=guests -e desired_state=start`
    - Stop only guests with `type: ct`:
      `ansible-playbook -i inventory.yml main.yml -t power -e target_group=guests -e desired_state=stop -e loop_type=ct`
    - Stop only guests whose hostname starts with ct- (wildcard limit on hostnames):
      `ansible-playbook -i inventory.yml main.yml -t power -e target_group=guests -e desired_state=stop --limit 'ct-*'`
    - If you keep a constructed/static group `type_ct`, you can also:
      `ansible-playbook -i inventory.yml main.yml -t power -e target_group=guests -e desired_state=stop --limit type_ct`
- Create PXE-boot VMs: tag `vm-create-pxe` (uses `tasks/create_vm_pxe.yml`)
  - Creates VMs configured for PXE (no ISO). Defaults: memory 2048MB, cores 2, disk 20GB on proxmoxpool01, node homelab-01, bridge vmbr0. Optional UEFI via `use_uefi: true`.
- Guests (CT/VM) OS-level tasks:
  - Create user on guests: tag `guests-create-user`
  - Update & upgrade guests: tags `guests-update, guests-upgrade`
  - Enable automatic security updates on guests: tag `guests-enable-auto-updates`
- VM-only OS-level tasks (after OS/SSH available):
  - Create user on VMs: tag `vm-create-user`
  - Update & upgrade VMs: tags `vm-update, vm-upgrade`
  - Enable automatic security updates on VMs: tag `vm-enable-auto-updates`

---

### Key Point: `gather_facts: false`

Most plays set `gather_facts: false` to skip the default “setup” step and speed up execution when host facts are not required.

---

## Task Files Layout

Tasks are modularized under `tasks/`:

- `tasks/power.yml` — unified power management for CTs and VMs
- `tasks/create_vm_pxe.yml` — create PXE-boot VMs (no ISO)
- `tasks/create_user.yml` — create a user on guests/VMs
- `tasks/install_updates.yml` — apt update/upgrade on guests/VMs
- `tasks/enable_auto_updates.yml` — enable unattended-upgrades

Use `main.yml` at the repository root and select desired operations with tags.
