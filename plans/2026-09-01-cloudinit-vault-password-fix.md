# Fix Cloud-Init Provisioning

**Date:** 2026-09-01
**Repos:** `rzkw/ansible`, `rzkw/oci-cloudinfra`
**Status:** Pending approval

## Problem

`ansible-pull` fails during cloud-init with two issues:

**1. Collection not installed:**
```
ERROR! couldn't resolve module/action 'community.general.ufw'. This often indicates a misspelling,
missing collection, or incorrect module path.
```

The `pre_tasks` collection install has a `creates` check that matches a stale directory, causing the install to be skipped. Full error output:
```
[WARNING]: Could not match supplied host pattern, ignoring: vm
localhost | CHANGED => {
    "after": "bf9e62e0416604d67b6e28dc3185d05dc2b48c6f",
    "before": null,
    "changed": true
}
[WARNING]: Could not match supplied host pattern, ignoring: vm
ERROR! couldn't resolve module/action 'community.general.ufw'.
The error appears to be in '/root/.ansible/pull/vm/playbooks/server.yml': line 31, column 7, but may
be elsewhere in the file depending on the exact syntax problem.
```

**2. Vault variables undefined:** `ansible-pull` runs without vault access, so `home_ip`, `ansible_password`, `oci_config`, etc. are undefined.

**3. `vm` host pattern warning:** `hosts.ini` is empty, `ansible-pull` clones to `/root/.ansible/pull/vm/` and may be checking that path as a host pattern.

## Solution

1. Simplify collection install (remove `creates` check)
2. Rewrite `hosts.ini` to localhost only
3. Pass vault password via OCI instance metadata
4. Remove home IP SSH restriction (no static home IP)

## File Changes

### `rzkw/ansible` — 3 files

| File | Action | Details |
|------|--------|---------|
| `playbooks/server.yml` | Edit | Simplify collection install, remove UFW home IP task |
| `hosts.ini` | Edit | Rewrite to localhost only |
| `group_vars/all/vars.yml` | Edit | Remove `home_ip` |

### `rzkw/oci-cloudinfra` — 3 files

| File | Action | Details |
|------|--------|---------|
| `user-data.yaml` | Edit | Add `bootcmd` + `pull_args` for vault password |
| `terraform/instances/main.tf` | Edit | Simplify metadata, add `vault_password` |
| `terraform/instances/variables.tf` | Edit | Remove `user_data_path`, add `vault_password` |

## Detailed Changes

### 1. `ansible/playbooks/server.yml`

**Collection install** — remove `creates` check, simplify path:
```yaml
- name: install ansible collections
  ansible.builtin.command:
    cmd: ansible-galaxy collection install -r collections/requirements.yml
```

**Remove UFW home IP task** — delete lines 34-39:
```yaml
    - name: restrict ssh to home ip
      community.general.ufw:
        rule: allow
        port: "22"
        proto: tcp
        src: "{{ home_ip }}"
```

Keep `enable ufw` task (lines 31-33).

### 2. `ansible/hosts.ini`

Before:
```ini
[local]
# fill in IP address after provisioning

[nodes]
# fill in IP addresses after provisioning
```

After:
```ini
[local]
localhost
```

### 3. `ansible/group_vars/all/vars.yml`

Remove:
```yaml
home_ip: "{{ vault_home_ip }}"
```

### 4. `oci-cloudinfra/user-data.yaml`

```yaml
#cloud-config
ssh_pwauth: false
package_update: true
package_upgrade: true
packages:
  - git
bootcmd:
  - "curl -s http://169.254.169.254/opc/v1/instance/metadata/vault_password > /root/.vault_pass"
ansible:
  package_name: ansible-core
  install_method: pip
  pull:
    - url: "https://github.com/rzkw/ansible.git"
      playbook_names: [playbooks/server.yml]
      pull_args: "--vault-password-file=/root/.vault_pass"
```

**Why `bootcmd`?** Runs in cloud-init `init` stage, before the `ansible` module runs in `final` stage. Ensures `/root/.vault_pass` exists before `ansible-pull` executes.

**Why `169.254.169.254`?** OCI instance metadata service endpoint. Available on every OCI instance at link-local address. Same pattern used in `roles/dev-box/tasks/main.yml:76` for `tailscale_auth_key`.

### 5. `oci-cloudinfra/terraform/instances/main.tf`

Before:
```hcl
metadata = {
  ssh_authorized_keys = var.ssh_public_keys != null ? var.ssh_public_keys : ""
  user_data           = var.user_data_path != null ? base64encode(file(var.user_data_path)) : null
  tailscale_auth_key  = var.tailscale_auth_key
}
```

After:
```hcl
metadata = {
  ssh_authorized_keys = var.ssh_public_keys
  user_data           = base64encode(file("${path.module}/../../user-data.yaml"))
  tailscale_auth_key  = var.tailscale_auth_key
  vault_password      = var.vault_password
}
```

### 6. `oci-cloudinfra/terraform/instances/variables.tf`

Remove:
```hcl
variable "user_data_path" {
  description = "Path to the cloud-init user_data script"
  type        = string
  default     = "user-data.yaml"
}
```

Add:
```hcl
variable "vault_password" {
  description = "Ansible vault password for cloud-init provisioning"
  type        = string
  sensitive   = true
}
```

## Execution Order at Boot

1. cloud-init: `package_update`, `package_upgrade`, install `git`
2. `bootcmd`: create `/root/.vault_pass` from OCI metadata
3. cloud-init `ansible` module: install `ansible-core` via pip
4. `ansible-pull` with `--vault-password-file=/root/.vault_pass`: clone repo, run `server.yml`
5. `pre_tasks`: install collections, clone CIS role
6. `include_role: UBUNTU24_CIS` — CIS L1 hardening
7. `include_role: base` — apt, timezone, unattended-upgrades, SSH key
8. `include_role: dev-box` — dev tools, tailscale, docker_rootless
9. `post_tasks`: enable UFW

## Before Applying

Pass vault password to Terraform:
```bash
terraform apply -var="vault_password=your-vault-password"
```

## Manual Testing (on instance)

```bash
echo "your-vault-password" > /root/.vault_pass
ansible-pull --url=https://github.com/rzkw/ansible.git playbooks/server.yml --vault-password-file=/root/.vault_pass
```

## Manual Step Required

Remove `vault_home_ip` from `group_vars/all/vault.yml` (gitignored, can't edit from here):
```bash
ansible-vault edit group_vars/all/vault.yml --vault-password-file <your-vault-pass-file>
```

## Out of Scope

- Root `playbook.yml` — unchanged
- CI/CD changes
- Other vault variables (`ansible_password`, `oci_config`, etc.) — still needed for roles
