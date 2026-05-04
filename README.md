# my-infra-orchestrator

Ansible-based orchestrator for managing home infrastructure without Docker — Raspberry Pi nodes, a MikroTik router, and anything else that speaks SSH or the RouterOS API.

---

## Repository layout

```
infra-orchestrator/
├── ansible.cfg              # Global Ansible settings
├── inventory.ini            # Device IPs grouped by type
├── requirements.yml         # Community roles & collections
├── main.yml                 # Master playbook
├── group_vars/
│   ├── all.yml              # Public shared variables
│   └── vault.yml            # AES-256 encrypted secrets  ← MUST be encrypted
├── host_vars/
│   ├── pi-voice.yml         # Voice API Pi config
│   └── mikrotik.yml         # MikroTik router config
└── templates/
    └── voice-api.service.j2 # systemd unit for Voice API
```

---

## Managed devices

| Host | IP | Role |
|---|---|---|
| `pi-runner` | 192.168.1.50 | GitHub Actions self-hosted runner (Pi 5) |
| `pi-voice` | 192.168.1.51 | Voice / audio API |
| `mikrotik` | 192.168.88.1 | MikroTik home router |

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

### 1 — Install required roles & collections

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.routeros
```

### 2 — Create your vault password file (local only, never committed)

```bash
echo "a-very-strong-passphrase" > .vault_pass
chmod 600 .vault_pass
```

`.vault_pass` is already listed in `.gitignore`.

### 3 — Populate and encrypt secrets

Edit `group_vars/vault.yml` and replace the placeholder values:

```yaml
personal_access_token: "ghp_your_real_token"
ansible_ssh_pass:       "your_pi_ssh_password"
router_password:        "your_mikrotik_password"
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

Open `group_vars/all.yml` and set `github_account`, `github_repo`, `system_timezone`, etc.

Host-specific tuning lives in `host_vars/pi-voice.yml` and `host_vars/mikrotik.yml`.

---

## Running playbooks

### Full provisioning (all hosts)

```bash
ansible-playbook main.yml
```

> If you have not set up `.vault_pass`, append `--ask-vault-pass`.

### Target a single group or host

```bash
# Only networking devices (MikroTik)
ansible-playbook main.yml --limit networking

# Only a specific Pi
ansible-playbook main.yml --limit pi-voice
```

### Dry-run (check mode)

```bash
ansible-playbook main.yml --check --diff
```

---

## What each play does

| Play | Hosts | Description |
|---|---|---|
| Common Setup | `raspberries` | `apt upgrade`, installs common packages, enables unattended-upgrades |
| GitHub Actions Runner | `pi-runner` | Installs & registers a self-hosted runner via `monolithprojects.github_actions_runner` |
| Voice API | `pi-voice` | Installs audio deps, deploys `/etc/systemd/system/voice-api.service` via template |
| Secure MikroTik | `networking` | Blocks Telnet (TCP 23), allows established connections, sets router identity |

---

## Security notes

* **`group_vars/vault.yml` must be AES-256 encrypted** before any commit — see step 3 above.  
* **`.vault_pass` must never be committed** — it is git-ignored.  
* No plain-text secrets appear anywhere else in the repository.

---

## Adding new devices

1. Add the host to `inventory.ini` under the appropriate group (or a new one).  
2. Create `host_vars/<hostname>.yml` for device-specific variables.  
3. Add a new play to `main.yml` targeting that host/group.
