# Proxmox Management

## Development Setup

To create and activate a project virtual environment and install dependencies, run:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

This directory contains Ansible playbooks and inventory for managing Proxmox on your homelab cluster. It defines a two-step authentication flow:

1. **Bootstrapping**: connect as `root` to create the user defined by `ct_user`
2. **Day-to-day operations**: connect as the CT user (from `ct_user`) for container and update tasks

---

## Inventory and Variables

- **Inventory file**: `inventory.yml`
  - CT hosts listed under the `ct` group.
  - Each CT host defines:
    ```yaml
    ct_user: naruby
    vmid: <container ID>
    ```
- **Group vars**: `group_vars/ct.yml`
  ```yaml
  ansible_user: "{{ ct_user }}"
  ```
  Ensures all plays targeting the `ct` group log in as the `ct_user` user by default.

---

## Playbooks & Execution

The CT lifecycle is orchestrated via the umbrella playbook:

```bash
ansible-playbook main.yml
```

`main.yml` runs the following task files in sequence:

- **Start CT containers**: `tasks/start_ct.yml`
- **Create CT user**: `tasks/create_user.yml`
- **Update & upgrade CTs**: `tasks/install_updates.yml`
- **Enable automatic security updates**: `tasks/enable_auto_updates.yml`
- **Stop CT containers**: `tasks/stop_ct.yml`

---

### Key Point: `gather_facts: false`

Most plays set `gather_facts: false` to skip the default “setup” step and speed up execution when host facts are not required.

---

## Task Files Layout

CT operations are now modularized under a `tasks/` directory:

- `tasks/start_ct.yml`
- `tasks/create_user.yml`
- `tasks/install_updates.yml`
- `tasks/enable_auto_updates.yml`
- `tasks/stop_ct.yml`

Use `main.yml` at the repository root to execute all CT workflows in sequence:

```bash
ansible-playbook main.yml
```
