# Test `server.yml` Before Deploy + Tailscale-Only SSH (No Bastion)

> **For agentic workers:** do NOT implement this plan until the PR containing it is approved by the repo admin/code owner. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Verify `playbooks/server.yml` converges before the OCI VM deploy, and drop the bastion/SSH-lockout dependency by switching to Tailscale-only SSH access.

**Architecture:** Run the real `server.yml` playbook inside a Molecule/Docker container (systemd-enabled Ubuntu) to prove convergence + idempotence, with OCI/vault-specific values stubbed via a gitignored extra-vars file and the docker-rootless role gated off. Remove the vault-dependent UFW home-IP rule; SSH then rides Tailscale SSH (`tailscale up --ssh`), which bypasses the host firewall, so no bastion and no public port 22 are needed.

**Root cause of last lockout:** `playbooks/server.yml:35` UFW rule used `home_ip`, resolved only from a vault absent on the VM. UFW enables with default-deny -> port 22 blocked everywhere -> locked out (bastion would have been the only way in).

**Tech Stack:** Ansible 2.16, `ansible-lint`, `molecule` (docker driver), rootless Docker, `community.general` collection.

## Global Constraints

- FQCN for every module call (`ansible.builtin.*`, `community.general.*`); task-level keywords (`become`, `become_user`) never prefixed.
- `ansible-lint .` must pass with 0 failures before commit.
- All commits signed with SSH key `~/.ssh/agent-gh-signing` (explicit `-S`).
- GitHub Flow: branch -> signed commit -> `git pull --rebase` -> push -> PR -> merge -> delete branch. Never force-push.
- Rootless docker context only (never `default`).
- Simplest possible solution — no new abstractions beyond what the container test requires.
- No CI workflow changes (pre-existing `.github/workflows/ansible-lint.yml` is out of scope).
- **Major changes require the plan to be submitted for PR review and approved by the repo admin/code owner before implementation.**

---

## Task 1: AGENTS.md — plan-approval + references policy

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Add a `## Plan approval & references` section** after the `## GitHub Workflow` section:

```markdown
## Plan approval & references

- Never implement major plans without approval from the repo admin/code owner.
- Major changes require the plan to be submitted for PR review before any implementation.
- All plans and reports must include a References section citing sources for every design decision (libraries, services, runtime behavior, security controls).
- Acceptable sources: official product documentation, personal blogs from engineers/devs/sysadmins, and product engineering blogs. Academic papers are never acceptable.
```

- [ ] **Step 2: Commit**

```bash
git add AGENTS.md plans/2026-08-04-test-server-playbook-tailscale-ssh.md
git commit -S -m "docs: add plan approval and references policy; add server playbook test plan"
```

## Task 2: Test environment (ADT + lint baseline)

**Files:**
- venv (Python 3.12) via `ansible_ade_setup_environment` (osType `linux`, osDistro `ubuntu`, python 3.12; collections `community.general`, `community.crypto`, `ansible.posix`); install `molecule`, `molecule-plugins[docker]`, `ansible-lint`. Fall back to `ansible_adt_check_env` if the MCP installer handles it.

- [ ] **Step 1: Start rootless Docker**

Run: `systemctl --user start docker` (verify socket at `~/.docker/run/docker.sock`; if the daemon was never configured, run `dockerd-rootless-setuptool.sh install` first).

- [ ] **Step 2: Baseline lint**

Run: `ansible-lint .` — record current failures.

## Task 3: Container test for `server.yml` (molecule)

**Files:**
- Create: `playbooks/local-test.yml` (gitignored stub vars)
- Modify: `.gitignore` (add `playbooks/local-test.yml`)
- Create/replace: `molecule/linux/molecule.yml`, `converge.yml`, `prepare.yml`, `verify.yml` (delete broken `create.yml`/`destroy.yml` stubs; use driver-bundled ones)

- [ ] **Step 1: Create the gitignored stub vars file**

`playbooks/local-test.yml`:
```yaml
---
oci_config: ""
oci_cli_rc: ""
opencode_config: ""
opencode_agents_md: ""
skip_docker_rootless: true
```

`.gitignore` append:
```
playbooks/local-test.yml
```

- [ ] **Step 2: Regenerate the molecule scenario**

Run `molecule init scenario linux --driver docker` in `molecule/`, then adapt the generated files to the target below (schema keys match the installed molecule version).

`molecule/linux/molecule.yml` target settings:
```yaml
driver: {name: docker}
platforms:
  - name: instance
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    command: /sbin/init
    cgroupns_mode: host
    privileged: true
    volumes: ["/sys/fs/cgroup:/sys/fs/cgroup:rw"]
# roles_path -> ../../roles ; collections_path -> collections
# extra-vars -> -e @../../playbooks/local-test.yml
#                -e server_playbook_hosts=instance -e server_playbook_connection=docker
scenario:
  test_sequence: [dependency, create, prepare, converge, idempotence, verify, destroy]
```

- [ ] **Step 3: Write converge, prepare, verify**

`molecule/linux/converge.yml`:
```yaml
---
- name: Converge
  import_playbook: ../../playbooks/server.yml
```

`molecule/linux/prepare.yml` (roles require these users):
```yaml
---
- name: Prepare
  hosts: all
  become: true
  tasks:
    - name: Create ubuntu user
      ansible.builtin.user:
        name: ubuntu
        state: present
    - name: Create rizky user
      ansible.builtin.user:
        name: rizky
        state: present
```

`molecule/linux/verify.yml`:
```yaml
---
- name: Verify
  hosts: all
  gather_facts: true
  tasks:
    - name: Check timezone
      ansible.builtin.command: timedatectl show -p Timezone --value
      changed_when: false
      register: tz
    - name: Assert timezone
      ansible.builtin.assert:
        that: tz.stdout == "Australia/Sydney"
    - name: Gather package facts
      ansible.builtin.package_facts:
    - name: Assert dev tools installed
      ansible.builtin.assert:
        that:
          - "'gh' in ansible_facts.packages"
          - "'tailscale' in ansible_facts.packages"
          - "'terraform' in ansible_facts.packages"
    - name: Check ufw status
      ansible.builtin.command: ufw status
      changed_when: false
      register: ufw_status
    - name: Assert tailscale ufw rule
      ansible.builtin.assert:
        that: "'tailscale0' in ufw_status.stdout"
```

## Task 4: `server.yml` — comment out CIS hardening

**Files:**
- Modify: `playbooks/server.yml:12-23` (CIS clone pre-task) and `:19-23` (CIS include_role)

- [ ] **Step 1: Comment out the CIS clone pre-task and include_role**

```yaml
  pre_tasks:
    - name: install ansible collections
      ansible.builtin.command:
        cmd: ansible-galaxy collection install -r {{ playbook_dir }}/../collections/requirements.yml
      args:
        creates: /root/.ansible/collections/ansible_collections/community/general
    # CIS hardening temporarily disabled — its sshd restrictions clash with
    # Tailscale SSH. Re-enable once the tailnet access path is confirmed.
    # - name: clone cis hardening role
    #   ansible.builtin.git:
    #     repo: https://github.com/ansible-lockdown/UBUNTU24-CIS.git
    #     dest: /tmp/UBUNTU24-CIS
    #     version: main
    #     depth: 1
  tasks:
    # - name: cis hardening
    #   ansible.builtin.include_role:
    #     name: /tmp/UBUNTU24-CIS
    #   vars:
    #     ubtu24cis_rule_4_2_3: false
```

## Task 5: `server.yml` — testable hosts/connection + Tailscale-only UFW

**Files:**
- Modify: `playbooks/server.yml:3-4` (hosts/connection) and `:34-39` (UFW post_tasks)
- Modify: `group_vars/all/vars.yml:11` (remove dead `home_ip`)

- [ ] **Step 1: Parametrize hosts/connection (defaults preserve ansible-pull behavior)**

```yaml
- hosts: "{{ server_playbook_hosts | default('localhost') }}"
  connection: "{{ server_playbook_connection | default('local') }}"
  become: true
```

- [ ] **Step 2: Replace the home-IP UFW rule with a Tailscale-interface rule**

```yaml
  post_tasks:
    - name: enable ufw
      community.general.ufw:
        state: enabled
    - name: allow ssh on tailscale interface
      community.general.ufw:
        rule: allow
        direction: in
        interface: tailscale0
```

- [ ] **Step 3: Remove the dead `home_ip` var**

`group_vars/all/vars.yml` — delete the `home_ip: "{{ vault_home_ip }}"` line (grep: only remaining reference is the removed UFW task).

## Task 6: `dev-box` — gate docker rootless for the sandbox

**Files:**
- Modify: `roles/dev-box/tasks/main.yml:141-145`

- [ ] **Step 1: Add a skip guard**

```yaml
- name: Docker rootless setup
  ansible.builtin.include_role:
    name: docker_rootless
  vars:
    service_user: ubuntu
  when: not skip_docker_rootless | default(false)
```

## Task 7: Run the molecule test

- [ ] **Step 1: Run the full molecule sequence**

Run: `molecule test` (in `molecule/`)
Expected: converge + idempotence + verify all pass. Fix any non-idempotent task surfaced by the idempotence step.

## Task 8: Static gates

- [ ] **Step 1: Lint**

Run: `ansible-lint .` — 0 failures.

- [ ] **Step 2: Syntax + task list**

Run:
```bash
ansible-playbook --syntax-check playbooks/server.yml playbooks/ping.yml playbooks/docker_rootless.yml
ansible-playbook --list-tasks playbooks/server.yml
```

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -S -m "test: run server playbook in molecule container with tailscale-only ssh"
```

## Task 9: ponytail review + commit + PR

- [ ] **Step 1: Ponytail review**

Review the diff for over-engineering (one line per finding; `net: -N lines`). Expected net-negative from the `home_ip` deletion. Remove anything flagged.
- [ ] **Step 2: Rebase + push + open PR**

```bash
git pull --rebase
git push -u origin feat/plan-server-playbook-test
gh pr create --title "Test server playbook + tailscale-only SSH" --body "See plans/2026-08-04-test-server-playbook-tailscale-ssh.md"
```

## Post-deploy SSH (no bastion) — verification notes

- After provisioning, connect directly to the tailnet IP: `ssh agent-walkllc@<100.x.y.z>` (or `tailscale ssh root@oci-devbox`). No bastion, no public port 22.
- `inventory.yml` requires `-e oci_devbox_tailscale_ip=<100.x.y.z> -e ssh_private_key_path=~/.ssh/...` at real deploy time (currently undefined vars).
- Confirm on the VM: `tailscale status` shows the node; `tailscale up --ssh` ran during provision.
- Known risk to verify during execution: molecule v8 ansible-native schema keys (use `molecule init` output as the base), and whether `ufw allow in on tailscale0` is accepted for a not-yet-existing interface in the container.

## Out of scope

- Root `playbook.yml` (hostname/exit-node/rsync), OCI/terraform provisioning, and `.github/workflows/ansible-lint.yml` (broken pre-existing).

## References

- Tailscale, "Access Oracle Cloud VMs privately using Tailscale" — https://tailscale.com/docs/install/cloud/oracle-cloud
- Tailscale, "Use ufw to lock down an Ubuntu server" — https://tailscale.com/docs/how-to/secure-ubuntu-server-with-ufw
- Tailscale, "Tailscale SSH" — https://tailscale.com/docs/features/tailscale-ssh
- Tailscale issue #18696 — tailscaled runtime behavior; SSH bypasses kernel firewall — https://github.com/tailscale/tailscale/issues/18696
- supun.io, "Restrict external SSH access using Tailscale and UFW" — https://supun.io/tailscale-ssh-restrict
- Ansible, `community.general.ufw` module (interface/direction params) — https://docs.ansible.com/projects/ansible/latest/collections/community/general/ufw_module.html
- Molecule docs, "Test complete playbooks" (converge via `import_playbook`) — https://docs.ansible.com/projects/molecule/configuration/
- Ansible DevTools MCP `ansible_ansible_content_best_practices` (topics: playbooks, roles, testing) and `ansible_zen_of_ansible` — confirmed structure/best practices
- Repo-internal design doc (home_ip intended inline) — `plans/2026-07-13-cloudinit-ansible-cis-design.md`
