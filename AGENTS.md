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

## GitHub Workflow

- Always `git pull --rebase` before committing and pushing to maintain linear history.
- Never push commits without pulling first — keeps branches up to date.

## Plan approval & references

- Never implement major plans without approval from the repo admin/code owner.
- Major changes require the plan to be submitted for PR review before any implementation.
- All plans and reports must include a References section citing sources for every design decision (libraries, services, runtime behavior, security controls).
- Acceptable sources: official product documentation, personal blogs from engineers/devs/sysadmins, and product engineering blogs. Academic papers are never acceptable.

## Keep it simple

> "Whenever you can, do things simply.
>
> Use advanced features only when necessary, and select the feature that best matches your use case. For example, you will probably not need vars, vars_files, vars_prompt and --extra-vars all at once, while also using an external inventory file.
>
> If something feels complicated, it probably is. Take the time to look for a simpler solution."

Prefer fewer moving parts: inline vars over `vars_files`, `include_role` over indirection layers, one playbook over orchestrator playbooks.

## Ansible MCP Server

The Ansible Development Tools MCP server is connected. Use these tools for quality checks:

| Tool | Use |
|------|-----|
| `ansible_ansible_lint` | Lint playbooks/roles. Run on every change. |
| `ansible_ansible_content_best_practices` | Check alignment with Ansible conventions. Query by topic. |
| `ansible_ade_environment_info` | Verify Python, Ansible, ADT, collections status. |
| `ansible_ansible_navigator` | Run playbooks with smart environment detection. |
| `ansible_zen_of_ansible` | Design philosophy reference. |

### Lint workflow

```sh
ansible-lint .    # CLI fallback
```

Or via MCP: invoke `ansible_ansible_lint` with file path. Fail clean — 0 warnings before commit.

## Prerequisites

- Ansible installed (collection installs via pip or system package)

## Git Rules

- **Never force push.** `git push --force` and `--force-with-lease` are forbidden on any branch. Add new commits only.
- **Always rebase, never merge.** Keep a linear history. Use `git pull --rebase` to incorporate upstream changes.
- **Rebase before every push.** Rebasing onto the target branch before pushing avoids merge conflicts.

## Commit signing

All commits must be signed with SSH key `~/.ssh/agent-gh-signing`. Git is configured globally (`gpg.format = ssh`, `user.signingkey = ~/.ssh/agent-gh-signing.pub`, `commit.gpgsign = true`). Verify with `git log --show-signature -1` before pushing.
