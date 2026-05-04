# my-infra-orchestrator

Ansible-based orchestrator for managing home infrastructure without Docker — Raspberry Pi nodes and anything else that speaks SSH. Designed to expand to additional device types (routers, microcontrollers, etc.) as needs grow.

---

## Repository layout

```
infra-orchestrator/
├── ansible.cfg          # Global Ansible settings
├── inventory.ini        # Host groups (no IPs — resolved from vault)
├── requirements.yml     # Community roles
├── main.yml             # Master playbook
├── group_vars/
│   ├── all.yml          # Public shared variables
│   └── vault.yml        # AES-256 encrypted secrets  ← MUST be encrypted
└── host_vars/
    └── pi-runner.yml    # Resolves ansible_host from vault
```

---

## Managed devices

| Host | Role |
|---|---|
| `pi-runner` | GitHub Actions self-hosted runner |

> The IP address is stored in `vault.yml` as `pi_runner_ip` and never committed in plain text.

---

## Prerequisites

| Tool | Version |
|---|---|
| Python | ≥ 3.9 |
| Ansible | ≥ 2.14 |
| ansible-galaxy | bundled with Ansible |

```bash
pip install ansible
```

---

## First-time setup

### 1 — Install required roles

```bash
ansible-galaxy install -r requirements.yml
```

### 2 — Create your vault password file (local only, never committed)

```bash
echo "a-very-strong-passphrase" > .vault_pass
chmod 600 .vault_pass
```

`.vault_pass` is already listed in `.gitignore`.

### 3 — Populate and encrypt secrets

Edit `group_vars/vault.yml` and replace **all** placeholder values:

```yaml
personal_access_token: "ghp_your_real_token"
ansible_ssh_pass:       "your_pi_ssh_password"
ssh_port:               "22"
github_account:         "your-github-username"
github_repo:            "your-app-repo"
pi_runner_ip:           "192.168.x.x"
```

Then encrypt the file so it is safe to commit:

```bash
ansible-vault encrypt group_vars/vault.yml
```

To edit it later:

```bash
ansible-vault edit group_vars/vault.yml
```

### 4 — Adjust public variables

Open `group_vars/all.yml` and set `system_timezone` if needed. All sensitive values (`github_account`, `github_repo`, `ssh_port`, `pi_runner_ip`) are now in `vault.yml`.

---

## Running playbooks

### Full provisioning

```bash
ansible-playbook main.yml
```

> If you have not set up `.vault_pass`, append `--ask-vault-pass`.

### Target a specific host

```bash
ansible-playbook main.yml --limit pi-runner
```

### Dry-run (check mode)

```bash
ansible-playbook main.yml --check --diff
```

---

## What each play does

| Play | Hosts | Description |
|---|---|---|
| Common Setup | `raspberries` | `apt dist-upgrade`, installs common packages, enables unattended-upgrades |
| GitHub Actions Runner | `pi-runner` | Installs & registers a self-hosted runner via `monolithprojects.github_actions_runner` |

---

## Security notes

* **`group_vars/vault.yml` must be AES-256 encrypted** before any commit — see step 3 above.  
* **`.vault_pass` must never be committed** — it is git-ignored.  
* No plain-text secrets appear anywhere else in the repository.

---

## Adding new devices

1. Add the host to `inventory.ini` under the appropriate group (or a new one).  
2. Create `host_vars/<hostname>.yml` for any device-specific variables.  
3. Add a new play to `main.yml` targeting that host/group.

