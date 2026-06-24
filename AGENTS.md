# AGENTS.md — Ansible homelab config

## Structure

Standard Ansible layout with a single inventory and roles under `roles/`.

```
ansible.cfg             # global config
hosts.ini               # single inventory ([local], [nodes])
group_vars/
  all/vars.yml          # SSH port 5678, vault password template
  all/vault.yml         # encrypted vault (gitignored)
  local/vars.yml        # ansible_connection: local
playbooks/              # all playbooks here
roles/
  docker_rootless/      # Docker Engine rootless mode
templates/              # chrony.conf.j2
```

## Key config (`ansible.cfg`)

| Setting | Value |
|---|---|
| inventory | `hosts.ini` |
| interpreter_python | `auto_silent` |
| forks | 25 |

Custom SSH port `5678` is set in `group_vars/all/vars.yml`. User name is templated from vault.

## Secrets

- `group_vars/all/vault.yml` is gitignored. Vault variables are loaded automatically via group_vars.
- Run with `--ask-vault-pass` or configure `ANSIBLE_VAULT_PASSWORD_FILE`.
- `ansible_password` is never stored in plaintext — always templated from vault.

## Running playbooks

```sh
# all playbooks live under playbooks/
ansible-playbook playbooks/ping.yml
ansible-playbook playbooks/playbook.yml                   # install packages
ansible-playbook playbooks/docker_rootless.yml           # rootless Docker Engine
```

## Style rules — enforced

- Use FQCN for every module call (`ansible.builtin.apt:`, `ansible.builtin.file:`, etc.). No bare `apt:` or `copy:`.
- Task-level keywords (`become`, `become_user`) are never prefixed.
- Target OS is Debian/Ubuntu only (`apt`).
- Playbooks typically use `become: true`.

## Roles

- `docker_rootless` — installs Docker Engine in rootless mode. Follows official docs for `apt` repo setup, package install, and `dockerd-rootless-setuptool.sh`.
- All roles follow standard conventions (`tasks/`, `defaults/`, `handlers/`, `vars/`, `meta/`).

## Verification

```sh
pip3 install ansible-lint
ansible-lint .
```

Must pass with 0 failures before committing.

## Prerequisites

- Ansible installed (collection installs via pip or system package)
