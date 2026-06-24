# Ansible homelab — repo cleanup log

## Scope

Full cleanup of `/Users/rizky/Documents/GitHub/ansible` — a personal Ansible homelab repo managing 3 Ubuntu servers + 1 local node.

## Tasks completed

### Layout standardisation
- Consolidated 3 separate inventory files into single `hosts.ini` with `[local]` and `[nodes]` groups
- Removed hardcoded IP addresses (VMs not yet provisioned)
- Moved all playbooks to `playbooks/`
- Moved `group_vars` from `playbooks/group_vars` to root `group_vars/all/` and `group_vars/local/`
- Moved `index.html`, `nginx.conf` → deleted (were learning examples)
- Removed `playbooks/hosts.ini`, `playbooks/local_vars/`

### FQCN consistency pass
Converted all short-name module references to FQCN across all `.yml` files:
- `apt:` → `ansible.builtin.apt:`
- `group:` → `ansible.builtin.group:`
- `command:` → `ansible.builtin.command:`
- `key=value` service syntax → proper YAML
- Fixed invalid `ansible.builtin.become:` / `ansible.builtin.become_user:` — these are task keywords, not module calls
- Removed nonsensical `service:` task that listed git/ansible/ca-certificates as services

### File removals
- `nodejs-app/` (playbook + Express app + inline role + RHEL/dnf reference)
- `install-packages/` (empty file)
- `playbooks/nginx-playbook.yml`
- `nginx.conf`, `index.html`
- `test.txt` (empty)
- `docker-engine-rootless/` (replaced by standard role)

### docker_rootless role (new)
Migrated from `docker-engine-rootless/` self-contained directory to standard `roles/docker_rootless/`:

**Bugs fixed:**
- `Suites: jammy` hardcoded → `{{ ansible_distribution_release }}` (dynamic)
- Missing `Architectures` in docker.sources → uses `dpkg --print-architecture`
- Removed `shell: /bin/bash` from `group:` task (groups don't have shells — ignored but misleading)
- Added uninstall of conflicting old packages (`docker.io`, `docker-compose`, `podman-docker`, etc.)
- Added `docker-ce-rootless-extras` to install list (required for `dockerd-rootless-setuptool.sh`)
- `DOCKER_HOST` path used username `/run/user/{{ service_user }}/` → registered user UID via `service_user_info.uid`
- Hardcoded `/home/{{ service_user }}` paths → `{{ service_user_info.home }}`
- Handler had leading space `" {{ item }}"` → `"{{ item }}"`
- Trailing blank lines cleaned up
- SPDX comments fixed (`#SPDX` → `# SPDX`)

### PII removal
- `ansible_user: rizky` → `ansible_user: "{{ vault_ansible_user }}"` (templated from vault)
- `author: Rizky` → `author: homelab` in both role meta files
- AGENTS.md reference to username removed

### ansible-lint
- Installed `ansible-lint` via `pip3`
- Individual role/playbook lint passes: 0 failures
- Full-repo sweep (`ansible-lint .`): **0 failures, 0 warnings** (production profile)

### AGENTS.md
Rewritten completely — documents layout, commands, conventions, role descriptions, verification step.

## Final stats
- **36 files changed**, +214 / −384 lines
- **11 directories** at root (clean standard Ansible layout)
- **3 playbooks** in `playbooks/`
- **1 role** in `roles/` (docker_rootless)
- **ansible-lint passes with 0 failures**
