# AGENTS.md — Ansible homelab config

## Keep it simple

> "Whenever you can, do things simply.
>
> Use advanced features only when necessary, and select the feature that best matches your use case. For example, you will probably not need vars, vars_files, vars_prompt and --extra-vars all at once, while also using an external inventory file.
>
> If something feels complicated, it probably is. Take the time to look for a simpler solution."

Prefer fewer moving parts: inline vars over `vars_files`, `include_role` over indirection layers, one playbook over orchestrator playbooks.

When suggesting solutions, problem solving, or implementing changes — always strive for the simplest possible solution that works. Avoid overengineering, overcomplicating, building unnecessarily difficult solutions.

## Structure

Standard Ansible layout with a single inventory and roles under `roles/`.

```
ansible.cfg             # global config
hosts.ini               # single inventory ([local], [nodes])
group_vars/
  all/vars.yml          # non-secret SSH defaults
  all/cis.yml           # Ubuntu 24 CIS role overrides
  local/vars.yml        # ansible_connection: local
playbooks/              # all playbooks here
roles/
  dev_box/               # dev-box packages, Tailscale, and Docker setup
  docker_rootless/      # Docker Engine rootless mode
templates/              # chrony.conf.j2
```

## Key config (`ansible.cfg`)

| Setting | Value |
|---|---|
| inventory | `hosts.ini` |
| interpreter_python | `auto_silent` |
| forks | 25 |

SSH uses port `22`; remote usernames and key paths are set by inventory.

## Cloud-init provisioning

- `playbooks/server.yml` runs locally as root from cloud-init and must not prompt, pause, or ask for credentials.
- Cloud-init installs the collections in `collections/requirements.yml` before invoking the playbook.
- The Tailscale authentication key comes from authenticated OCI instance metadata; do not print it in task output.
- SSH uses public-key authentication for `ubuntu`, `rizky`, and `agent-walkllc`; password authentication is disabled.

## Running playbooks

```sh
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook -i hosts.ini playbooks/server.yml --syntax-check
ansible-playbook -i hosts.ini playbooks/ping.yml
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
