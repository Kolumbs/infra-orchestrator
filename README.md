# infra-orchestrator

Ansible-based orchestrator for managing home infrastructure without Docker — Raspberry Pi nodes and anything else that speaks SSH. Designed to expand to additional device types (routers, microcontrollers, etc.) as needs grow.

---

## Repository layout

```
infra-orchestrator/
├── ansible.cfg              # Global Ansible settings
├── inventory.ini            # Host groups (no IPs — resolved from vault)
├── requirements.yml         # Community roles
├── main.yml                 # Master playbook
├── group_vars/
│   └── all/
│       ├── vault.yml        # AES-256 encrypted secrets  ← MUST be encrypted
│       └── vars.yml         # Silent bridge (safe to commit — no real data)
└── host_vars/
    └── pi-server-1.yml      # Resolves ansible_host from vault
```

This repository uses a **zero-knowledge configuration**: even with full read access to the repo, an observer learns nothing about usernames, IPs, ports, or credentials without the vault password.

---

## Managed devices

| Host | Group | Role |
|---|---|---|
| `pi-server-1` | `servers`, `web_server` | GitHub Actions self-hosted runner + Apache web server |

---

## Prerequisites

| Tool | Version |
|---|---|
| Ansible | ≥ 2.14 |
| ansible-galaxy | bundled with Ansible |
| sshpass | any |

```bash
sudo apt install ansible-core sshpass
```

---

## First-time server bootstrap (manual, on the Pi)

Before Ansible can connect, a dedicated `ansible` user must be created on each server. Do this once over direct access (keyboard/HDMI or with the default `pi` user).

### 1 — Create the automation user

```bash
sudo useradd --create-home --shell /bin/bash --password '!' <username>
```

- `--password '!'` locks password login — SSH key is the only way in
- `/bin/bash` is required so Ansible can execute commands remotely

### 2 — Add your SSH public key

On your control machine, generate a key if you don't have one yet:

```bash
ssh-keygen -t ed25519
```

Then copy it to the Pi in one command (run from your control machine):

```bash
cat ~/.ssh/id_ed25519.pub | ssh <pi-user>@<pi-ip> "sudo mkdir -p /home/<username>/.ssh && sudo tee /home/<username>/.ssh/authorized_keys && sudo chmod 700 /home/<username>/.ssh && sudo chmod 600 /home/<username>/.ssh/authorized_keys && sudo chown -R <username>:<username> /home/<username>/.ssh"
```

### 3 — Grant passwordless sudo

```bash
sudo visudo -f /etc/sudoers.d/<username>
```

Add this line:

```
<username> ALL=(ALL) NOPASSWD: ALL
```

### 4 — Verify access from your control machine

```bash
ssh -i ~/.ssh/id_ed25519 <username>@<pi-ip>
```

Should connect without a password prompt. Once confirmed, Ansible is ready to use.

---

## First-time setup

### 1 — Install required roles

```bash
ansible-galaxy install -r requirements.yml
```

### 2 — Populate and encrypt secrets

Variables are split across two files:

- **`group_vars/all/vault.yml`** — sensitive values (credentials, IPs, ports). Every variable here uses a `vault_` prefix. Fill in all placeholder values, then encrypt the file before committing:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

To edit it later:

```bash
ansible-vault edit group_vars/all/vault.yml
```

- **`group_vars/all/vars.yml`** — non-sensitive values (repo name, timezone, etc.) committed in plain text, plus a bridge mapping each `vault_*` variable to the name Ansible and roles expect. Safe to commit as-is.

---

## Running playbooks

### Full provisioning

```bash
ansible-playbook main.yml
```

> Ansible will prompt for your vault password on each run.

### Target a specific host

```bash
ansible-playbook main.yml --limit pi-server-1
```

### Dry-run (check mode)

```bash
ansible-playbook main.yml --check --diff
```

---

## What each play does

| Play | Hosts | Description |
|---|---|---|
| Common Setup | `servers` | Installs common packages |
| System Upgrade | `servers` | Runs explicit `apt` upgrade policy via `system_upgrade_policy` (`none`/`safe`/`full`/`dist`/`yes`) |
| Web Server | `web_server` | Installs Apache (`apache2`) with optional version pinning and ensures service is running |
| GitHub Actions Runner | `pi-server-1` | Installs & registers a self-hosted runner via `monolithprojects.github_actions_runner` |

---

## Security notes

* **`group_vars/all/vault.yml` must be AES-256 encrypted** before any commit — see step 2 above.
* Ansible prompts for the vault password on each run — no plaintext password file is used or needed.
* `group_vars/all/vars.yml` contains only non-sensitive values and `{{ vault_* }}` references — it is safe to commit as-is.
* No plain-text secrets, IPs, usernames, or ports appear anywhere in the repository.
* **`vault_github_runner_registration_token`** is only needed when provisioning or reinstalling a runner — use a short-lived PAT (e.g. 1-day expiry, `repo` scope) and revoke it once provisioning is done.

---

## Removing things from managed hosts

Ansible does not automatically remove anything when you delete a task or role from a playbook. If you remove a package, service, or user from the playbook, it will keep running on the host untouched.

To actually remove something, you must explicitly add a task with `state: absent` first, run the playbook, then remove the task:

```yaml
- name: Remove package
  ansible.builtin.apt:
    name: somepackage
    state: absent

- name: Remove user
  ansible.builtin.user:
    name: someuser
    state: absent
```

For roles (e.g. GitHub Actions runner), check the role's documentation for a removal state — most support `state: absent` or similar.

---

## Adding new devices

1. Add the host to `inventory.ini` under the appropriate group (or a new one).
2. Add any new secrets to `group_vars/all/vault.yml` (with `vault_` prefix) and a corresponding bridge entry in `group_vars/all/vars.yml`.
3. Create `host_vars/<hostname>.yml` for device-specific variables (e.g. `ansible_host`).
4. Add a new play to `main.yml` targeting that host/group.
