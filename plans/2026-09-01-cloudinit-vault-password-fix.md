# Fix Cloud-Init Provisioning: Pass Vault Password via OCI Metadata

**Date:** 2026-09-01
**Repos:** `rzkw/ansible`, `rzkw/oci-cloudinfra`
**Status:** Pending approval

## Problem

`ansible-pull` fails during cloud-init because `playbooks/server.yml` uses vault-encrypted variables (`home_ip`, `ansible_password`, `oci_config`, etc.) but `ansible-pull` runs without vault access. Error at line 31 (`community.general.ufw`) — the `home_ip` variable is undefined.

```
The error appears to be in '/root/.ansible/pull/vm/playbooks/server.yml': line 31, column 7, but may
```

## Solution

Pass the vault password through OCI instance metadata so `ansible-pull` can decrypt vault variables. Also remove the home IP SSH restriction (no static home IP available).

## Design Decisions

- **Vault password in OCI metadata** — same pattern already used for `tailscale_auth_key`. Metadata is encrypted at rest in OCI.
- **`bootcmd` for vault file creation** — runs before `ansible` module in cloud-init final stage.
- **Remove UFW home IP rule** — no static home IP. UFW stays enabled, SSH open to all.
- **Hardcode user-data path** — only one `user-data.yaml` exists, no need for variable.

## File Changes

### `rzkw/oci-cloudinfra` — 3 files

| File | Action | Details |
|------|--------|---------|
| `user-data.yaml` | Edit | Add `bootcmd` to create vault password file from metadata, add `pull_args` |
| `terraform/instances/main.tf` | Edit | Simplify metadata (remove conditionals), add `vault_password` |
| `terraform/instances/variables.tf` | Edit | Remove `user_data_path`, add `vault_password` variable |

### `rzkw/ansible` — 2 files

| File | Action | Details |
|------|--------|---------|
| `playbooks/server.yml` | Edit | Remove `restrict ssh to home ip` task (lines 34-39) |
| `group_vars/all/vars.yml` | Edit | Remove `home_ip: "{{ vault_home_ip }}"` |

## Detailed Changes

### 1. `oci-cloudinfra/user-data.yaml`

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

### 2. `oci-cloudinfra/terraform/instances/main.tf`

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

**Why `base64encode`?** OCI API requires `user_data` to be base64-encoded in instance metadata.
**Why `${path.module}/../../user-data.yaml`?** `path.module` resolves to `terraform/instances/`. Two levels up is the project root where `user-data.yaml` lives.

### 3. `oci-cloudinfra/terraform/instances/variables.tf`

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

### 4. `ansible/playbooks/server.yml`

Remove lines 34-39:
```yaml
    - name: restrict ssh to home ip
      community.general.ufw:
        rule: allow
        port: "22"
        proto: tcp
        src: "{{ home_ip }}"
```

Keep `enable ufw` task (lines 31-33).

### 5. `ansible/group_vars/all/vars.yml`

Remove:
```yaml
home_ip: "{{ vault_home_ip }}"
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

## Out of Scope

- Root `playbook.yml` — unchanged
- CI/CD changes
- Removing `vault_home_ip` from vault file (no longer referenced, harmless)
- Other vault variables (`ansible_password`, `oci_config`, etc.) — still needed for roles
