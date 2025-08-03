# Proxmox Management

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
  This ensures all plays targeting the `ct` group log in as the `ct_user` user by default.

---

## Playbooks

### 1. `ct_init.yml` (Bootstrap)

- **Purpose**: create the `ct_user` user on each CT host and enable sudo + SSH key access.
- **Authentication**: uses `remote_user: root` (overrides the inventory’s `ansible_user`).
- **Tasks**:
  1. Create `ct_user` and add to `sudo` group
  2. Configure passwordless sudo file in `/etc/sudoers.d/`
  3. Set up `ct_user .ssh` directory
  4. Copy `/root/.ssh/authorized_keys` to `ct_user /.ssh/authorized_keys`

### 2. `install_updates.yml` (CT Updates)

- **Purpose**: update and upgrade Debian/Ubuntu containers in the `ct` group.
- **Authentication**: runs as the CT user (inherits `ansible_user` from `group_vars/ct.yml`).

### 3. `ct.yml` (Manage CT Containers)

- **Purpose**: start or stop LXC containers in Proxmox using the `community.general.proxmox` module.
- **Authentication**: runs as the CT user (inherits from group vars).
- **Loop**: iterates `vmid` values defined per host in the CT group.

---

## Running the Playbooks

```bash
# 1. Bootstrap CT hosts (run once per host)
ansible-playbook ct_init.yml

# 2. Update containers
ansible-playbook install_updates.yml

# 3. Manage CT state (start/stop)
ansible-playbook ct.yml --tags start_ct
ansible-playbook ct.yml --tags stop_ct
```

---

### Key Point: `gather_facts: false`

Most playbooks set `gather_facts: false` to skip the default “setup” step and speed up execution when host facts are not required.
