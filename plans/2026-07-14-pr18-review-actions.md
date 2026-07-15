# PR #18 Review Actions — Full Scope

**Date:** 2026-07-14
**PR:** [feat: cloud-init ansible bootstrap with CIS hardening](https://github.com/rzkw/ansible/pull/18)
**Branch:** `cloudinit-ansible-cis`

## Overview

Address 3 review comments + 3 additional instructions from rzkw on PR #18.
Full scope: review comments, templates removal, config imports, stale code cleanup, inventory fix.

---

## Task 1 — Review Comments: Config imports + SSH key

### `roles/dev-box/tasks/main.yml`

Add at end (after existing tasks), all under `become_user: ubuntu`:

```yaml
# ── Config Imports ──────────────────────────────────
- name: Create OCI CLI config directory
  ansible.builtin.file:
    path: /home/ubuntu/.oci
    state: directory
    owner: ubuntu
    group: ubuntu
    mode: "0700"

- name: Import OCI CLI config
  ansible.builtin.copy:
    src: ~/.oci/config
    dest: /home/ubuntu/.oci/config
    owner: ubuntu
    group: ubuntu
    mode: "0600"

- name: Import OCI CLI RC
  ansible.builtin.copy:
    src: ~/.oci/oci_cli_rc
    dest: /home/ubuntu/.oci/oci_cli_rc
    owner: ubuntu
    group: ubuntu
    mode: "0600"

- name: Create opencode config directory
  ansible.builtin.file:
    path: /home/ubuntu/.config/opencode
    state: directory
    owner: ubuntu
    group: ubuntu
    mode: "0755"

- name: Import opencode config
  ansible.builtin.copy:
    src: ~/.config/opencode/opencode.jsonc
    dest: /home/ubuntu/.config/opencode/opencode.jsonc
    owner: ubuntu
    group: ubuntu
    mode: "0644"

- name: Import opencode skills
  ansible.builtin.copy:
    src: ~/.config/opencode/skills/
    dest: /home/ubuntu/.config/opencode/skills/
    owner: ubuntu
    group: ubuntu
    mode: "0644"

- name: Import global AGENTS.md
  ansible.builtin.copy:
    src: ~/.config/opencode/AGENTS.md
    dest: /home/ubuntu/.config/opencode/AGENTS.md
    owner: ubuntu
    group: ubuntu
    mode: "0644"

- name: Import git config
  ansible.builtin.copy:
    src: ~/.gitconfig
    dest: /home/ubuntu/.gitconfig
    owner: ubuntu
    group: ubuntu
    mode: "0644"
```

### `roles/base/tasks/main.yml`

Replace the `key:` value in the authorized_key task:

```yaml
- name: Add sec-ssh-key for rizky
  ansible.posix.authorized_key:
    user: rizky
    key: "ssh-ed25519 AAAAGnNrLXNzaC1lZDI1NTE5QG9wZW5zc2guY29tAAAAIOsFnvsYS5kBFy1MpFHOQpL0IQSao8KRkmynissR79PvAAAAEnNzaDpkZXYtYm94LWJhY2t1cA== dev-box-backup"
```

---

## Task 2 — Integrate root playbook.yml into server.yml

Add to `playbooks/server.yml` post_tasks (after UFW tasks):

```yaml
    - name: Set hostname
      ansible.builtin.hostname:
        name: dev-box

    - name: Advertise as Tailscale exit node
      ansible.builtin.command: tailscale set --advertise-exit-node
      changed_when: false

    - name: Deploy rsync-backup script
      ansible.builtin.template:
        src: rsync-backup.sh.j2
        dest: /usr/local/bin/rsync-backup.sh
        mode: "0755"

    - name: Deploy rsync-backup systemd service
      ansible.builtin.template:
        src: rsync-backup.service.j2
        dest: /etc/systemd/system/rsync-backup.service
        mode: "0644"

    - name: Deploy rsync-backup systemd timer
      ansible.builtin.template:
        src: rsync-backup.timer.j2
        dest: /etc/systemd/system/rsync-backup.timer
        mode: "0644"

    - name: Enable and start rsync-backup timer
      ansible.builtin.systemd:
        name: rsync-backup.timer
        enabled: true
        state: started
        daemon_reload: true
```

---

## Task 3 — Remove stale files

| File | Reason |
|---|---|
| `playbooks/playbook.yml` | Replaced by server.yml |
| `playbooks/docker_rootless.yml` | Replaced by `include_role` in server.yml |
| `playbook.yml` (root) | Integrated into server.yml post_tasks |
| `templates/jail.local.j2` | Never deployed by any playbook |
| `templates/chrony.conf.j2` | Never deployed by any playbook |
| `templates/99-motd-warning.j2` | Never deployed by any playbook |
| `hosts.ini` | Fully commented out; real inventory is `inventory.yml` |
| `files/99-disable-ipv6.conf` | Never deployed; CIS handles IPv6 |
| `playbooks/ping.yml` | Utility playbook, not referenced |
| `group_vars/local/vars.yml` | Orphaned — no `local` group in inventory |

Keep: `templates/rsync-backup.{sh,service,timer}.j2`, `roles/docker_rootless/`.

---

## Task 4 — Inventory + group_vars cleanup

### Fix `ansible.cfg`

```ini
[defaults]
inventory = inventory.yml
interpreter_python = auto_silent
forks = 25
```

### Add vault vars

Create `group_vars/all/vault.yml` (already gitignored):

```yaml
oci_devbox_tailscale_ip: "REPLACE_WITH_ACTUAL_IP"
ssh_private_key_path: "~/.ssh/id_ed25519"
```

### Consolidate group_vars

Merge `group_vars/all.yml` + `group_vars/all/vars.yml` into `group_vars/all/vars.yml`:

```yaml
# ── Non-sensitive ──────────────────────────────────
ansible_user: "{{ vault_ansible_user }}"
ansible_port: 5678
ansible_connection: ssh

# ── Sensitive ──────────────────────────────────────
ansible_password: "{{ vault_ansible_password }}"
tailscale_auth_key: "{{ vault_tailscale_auth_key }}"

# ── Rsync backup ───────────────────────────────────
rsync_backup_sources: "/etc /home /root /var /usr/local/bin /usr/local/sbin /srv /opt"
rsync_backup_schedule: "*-*-* 02:00:00"
```

Delete `group_vars/all.yml`.

---

## Task 5 — Global AGENTS.md on dev box

Consolidate `/Users/rizky/AGENTS.md` + `/Users/rizky/AGENTS-2.md` into single file.
Place at `/home/ubuntu/.config/opencode/AGENTS.md` via the copy task in Task1.

**Merged content:**

```markdown
# Global Instructions

## Git Commit Signing

All commits pushed to any repository MUST be signed using SSH key `~/.ssh/agent-gh-signing`.

Git is configured globally at `~/.gitconfig`:

- `gpg.format = ssh`
- `user.signingkey = ~/.ssh/agent-gh-signing.pub`
- `commit.gpgsign = true`

Before pushing, verify the signature is present with `git log --show-signature -1`.

## Agent Configuration

### Workflow
- Always follow GitHub Flow: branch -> implement -> push -> PR -> delete branch after merge

### GitHub
- Username: `agent-walkllc`
- Email: `agent@walk-llc.com`
- gh CLI: authenticated (OAuth token, scopes: gist, read:org, repo)
- MCP PAT: `GitHub_MCP_PAT` in Bitwarden (30-day expiry, scopes: repo, project, read:user, read:org, gist)

### Bitwarden
- Email: `agent@walk-llc.com`
- CLI: installed at `/home/agent-walkllc/.local/bin/bw`
- Session: `BW_SESSION` env var (ephemeral)

### SSH
- Auth key: `~/.ssh/id_ed25519`
- Signing key: `~/.ssh/agent-gh-signing` (ed25519, registered as commit signing key)
- Config: `~/.ssh/config` with `Host github.com` entry

### MCP Configuration
- Global: `~/.config/opencode/opencode.jsonc` -> `https://api.github.com/mcp`
- Project: `.mcp.json` in workspace
- Auth: `GH_TOKEN` from Bitwarden via direnv

### Git Protocol
- HTTPS via gh CLI OAuth token (git_protocol = https)

### Commit Signing
- ALL commits MUST be signed using SSH key `~/.ssh/agent-gh-signing`
- Signing email: `288607573+agent-walkllc@users.noreply.github.com`
- Use explicit `-S` flag on every `git commit` (e.g., `git commit -S -m "..."`)
- Ensure `gpg.format = ssh` is set: `git config gpg.format ssh`
- Before committing, verify signing is configured: `git config user.signingkey`, `git config gpg.format`, and `git config user.email` must return expected values
- If `gpg.format` is not `ssh`, `user.signingkey` is not `~/.ssh/agent-gh-signing.pub`, or `user.email` is not `288607573+agent-walkllc@users.noreply.github.com`, set them before committing

### Bash Token Optimization

Always minimize token usage when calling the bash tool:

- **Batch independent calls**: Combine related commands with `&&` into single bash calls instead of separate tool invocations
- **Use built-in tools**: Prefer `read`, `grep`, `glob` tools over bash equivalents for file operations
- **Shorter alternatives**: Use `rg` over `grep -r`, `fd` over `find -name`
- **Suppress verbose output**: Add `-q`/`--quiet` flags, use `2>/dev/null`, or `| tail -1` to limit output
- **One-liners**: Chain operations with `&&` or `;` when they're sequential but independent

## Boundaries

- Always: Follow naming conventions
- Never: Commit secrets or API keys
```

---

## Task 6 — README rewrite

Rewrite `README.md` to reflect new structure:

- Single `server.yml` playbook
- CIS hardening + base + dev-box roles
- Cloud-init boot flow
- Pre-requisites
- Usage

---

## Execution Order

1. Tasks 1-2 (review comments + integration) — core PR changes
2. Task 3 (remove stale files) — cleanup
3. Task 4 (inventory + group_vars) — cleanup
4. Task5 (AGENTS.md consolidation) — config import
5. Task 6 (README rewrite) — documentation
6. Run `ansible-lint .` to verify
7. Commit and push to PR branch
