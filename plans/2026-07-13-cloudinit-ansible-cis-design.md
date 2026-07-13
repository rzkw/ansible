# Migrate Cloud-Init from Bash to Ansible with CIS Hardening

**Date:** 2026-07-13
**Repos:** `rzkw/ansible`, `rzkw/oci-cloudinfra`

## Problem

`oci-cloudinfra/user-data.sh` is a 578-line bash script doing CIS L1 hardening (~490 lines) and dev-box package installation (~85 lines) during cloud-init boot. Unmaintainable, unverifiable, approaching 32KB OCI limit.

## Solution

Replace `user-data.sh` with a cloud-init YAML config using the `ansible` module (ansible-pull). A single playbook orchestrates hardening (`ansible-lockdown.UBUNTU24_CIS`), base system config, and dev-box packages via roles. Order: hardening → base → dev-box.

## Design Principle

[Keep it simple](https://docs.ansible.com/projects/ansible/latest/tips_tricks/ansible_tips_tricks.html#keep-it-simple). No wrapper roles, no unnecessary abstractions, no galaxies. One playbook, three `include_role` calls. Vars inline or in one file.

## File Changes

### rzkw/oci-cloudinfra

| File | Action |
|------|--------|
| `user-data.yaml` | Create — cloud-init with ansible module |
| `user-data.sh` | Delete — all logic moves to Ansible |
| `terraform/instances/variables.tf` | Edit — update `user_data_path` default |

### rzkw/ansible

| File | Action |
|------|--------|
| `playbooks/server.yml` | Create — single playbook |
| `group_vars/all/cis.yml` | Create — CIS role overrides |
| `collections/requirements.yml` | Create — galaxy deps |
| `roles/base/tasks/main.yml` | Edit — flesh out |
| `roles/dev-box/tasks/main.yml` | Edit — fixes, tailscale, docker, remove dupes |

## Execution Order at Boot

1. cloud-init: `package_update`, `package_upgrade`, install `git`
2. cloud-init `ansible` module: install `ansible-core` via pip
3. `ansible-pull`: clone `rzkw/ansible.git`, run `playbooks/server.yml`
4. `pre_tasks`: install collections (UBUNTU24_CIS, community.general, etc.)
5. `include_role: UBUNTU24_CIS` — CIS L1 hardening
6. `include_role: base` — apt upgrade, packages, timezone, unattended-upgrades, SSH key
7. `include_role: dev-box` — dev tools, tailscale, docker_rootless
8. `post_tasks`: enable UFW, restrict SSH to home IP

## Key Decisions

- **`include_role` over static `roles:`** — UBUNTU24_CIS is installed in `pre_tasks`. Static roles resolve at parse time (before pre_tasks run). Dynamic `include_role` resolves at execution time.
- **[ADDRESS]** — skip CIS default OpenSSH-allow-from-any. Add restricted rule in post_tasks.
- **No audit** — `run_audit: false`, `setup_audit: false`. Machine is fresh.
- **home_ip inline** — defined as `vars:` in server.yml. No vault, no extra file.

## Out of Scope

- Root `playbook.yml` (hostname, PufferPanel, rsync) — unchanged
- CI/CD changes
- Tailscale auth config (happens manually post-provision)
