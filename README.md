# ansible-iaas

Infrastructure-as-Code Ansible project for provisioning and ongoing configuration management of Linux and Windows servers. Designed for reliable, idempotent, team-friendly operation — today from the CLI, tomorrow from Ansible Automation Platform.

---

## Table of Contents

- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Inventory](#inventory)
- [Roles](#roles)
- [Playbooks](#playbooks)
- [Secrets Management (Vault)](#secrets-management-vault)
- [GitHub Actions](#github-actions)
- [Terraform Integration](#terraform-integration)
- [Team Conventions](#team-conventions)
- [Adding a New Role](#adding-a-new-role)

---

## Architecture

```
ansible-iaas/
├── .github/workflows/       # CI/CD: lint (PR), provision (post-TF), configure (schedule/push)
├── inventories/
│   ├── development/         # Dev hosts + group_vars (least restrictive)
│   ├── staging/
│   └── production/          # Prod hosts + group_vars (strictest)
├── roles/
│   ├── common/              # Applied to every host: packages, NTP, sysctl, motd
│   ├── linux_hardening/     # CIS-aligned SSH, firewall, auditd, password policy
│   ├── datadog/             # Datadog agent (Linux + Windows)
│   ├── tenable_agent/       # Nessus agent (Linux + Windows)
│   ├── dns_config/          # systemd-resolved / resolv.conf / Windows DNS
│   └── ms_defender/         # Microsoft Defender for Endpoint (Linux + Windows)
├── playbooks/
│   ├── site.yml             # Master — applies everything
│   ├── provision.yml        # First-boot — waits for connectivity then runs site
│   ├── linux.yml            # All Linux roles in order
│   ├── windows.yml          # All Windows roles
│   ├── hardening.yml        # Hardening only
│   ├── monitoring.yml       # Datadog only
│   └── security_agents.yml  # Tenable + MDE only
├── collections/
│   └── requirements.yml     # Pinned Ansible Galaxy collections
├── scripts/
│   └── tf_inventory.py      # Converts Terraform outputs → Ansible inventory
└── ansible.cfg              # Project-level Ansible config
```

---

## Quick Start

### Prerequisites

```bash
# Python 3.10+
pip install ansible

# Install Galaxy collections
ansible-galaxy collection install -r collections/requirements.yml -p collections/

# Create your vault password file (never commit this)
echo "YOUR_VAULT_PASSWORD" > .vault_pass
chmod 600 .vault_pass
```

### First run (check mode — no changes)

```bash
ansible-playbook playbooks/site.yml --check --diff
```

### Target a single host or group

```bash
ansible-playbook playbooks/site.yml -l dev-web-01
ansible-playbook playbooks/site.yml -l webservers
```

### Target a specific environment

```bash
ansible-playbook playbooks/site.yml -i inventories/production/hosts.yml
```

### Run only specific capabilities (tags)

```bash
# Datadog only
ansible-playbook playbooks/site.yml --tags datadog

# SSH hardening only
ansible-playbook playbooks/site.yml --tags ssh

# Multiple tags
ansible-playbook playbooks/site.yml --tags "common,dns"
```

---

## Inventory

Each environment (`development`, `staging`, `production`) has:

- `hosts.yml` — static host definitions and group membership
- `group_vars/all/vars.yml` — plain variables (committed)
- `group_vars/all/vault.yml` — encrypted secrets (**must be vault-encrypted**)
- `group_vars/linux/vars.yml` — Linux-specific variables
- `group_vars/windows/vars.yml` — Windows-specific variables
- `host_vars/` — per-host overrides

The default inventory (`ansible.cfg`) points to **development** so that an unqualified `ansible-playbook` run never accidentally targets production.

---

## Roles

| Role | Purpose | Tags |
|------|---------|------|
| `common` | Packages, NTP, sysctl, motd, rsyslog | `common`, `ntp`, `packages` |
| `linux_hardening` | SSH, firewall, auditd, password policy, service minimization | `hardening`, `ssh`, `firewall`, `auditd`, `password` |
| `datadog` | Datadog agent install + config | `datadog`, `monitoring` |
| `tenable_agent` | Nessus agent install + link | `tenable`, `security` |
| `dns_config` | DNS resolver configuration | `dns` |
| `ms_defender` | Microsoft Defender for Endpoint | `mde`, `defender`, `security` |

All role defaults live in `roles/<role>/defaults/main.yml`. Override them in `inventories/<env>/group_vars/` — never edit role defaults for environment-specific values.

---

## Playbooks

| Playbook | When to use |
|----------|-------------|
| `site.yml` | Full desired-state convergence (config management runs) |
| `provision.yml` | First-boot after Terraform creates a machine |
| `linux.yml` | All Linux roles (imported by site.yml) |
| `windows.yml` | All Windows roles (imported by site.yml) |
| `hardening.yml` | Hardening-only runs |
| `monitoring.yml` | Datadog update/reinstall |
| `security_agents.yml` | Tenable + MDE update/reinstall |

---

## Secrets Management (Vault)

### Convention

- All secrets are stored in `vault.yml` files (one per environment, in `group_vars/all/`)
- Vault variables are prefixed `vault_` (e.g. `vault_datadog_api_key`)
- They are referenced in plain `vars.yml` as `datadog_api_key: "{{ vault_datadog_api_key }}"` — this keeps variable names visible without exposing values

### Encrypting vault files

```bash
# Encrypt a new vault file
ansible-vault encrypt inventories/production/group_vars/all/vault.yml

# Edit an encrypted file
ansible-vault edit inventories/production/group_vars/all/vault.yml

# View without editing
ansible-vault view inventories/production/group_vars/all/vault.yml

# Re-key (change password)
ansible-vault rekey inventories/production/group_vars/all/vault.yml
```

### In GitHub Actions

Store the vault password as a GitHub Secret (`ANSIBLE_VAULT_PASSWORD`) per environment. The workflows write it to `.vault_pass` and delete it after the run. The file is also in `.gitignore` as a safety net.

---

## GitHub Actions

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `lint.yml` | Pull request | ansible-lint + syntax-check (PR gate) |
| `provision.yml` | `workflow_call` or manual | Post-Terraform first-boot provisioning |
| `configure.yml` | Push to main, daily schedule, manual | Ongoing config management — dev → staging → prod |

### GitHub Environments

Configure these in **Settings → Environments**:

- `development` — auto-approve, used on every push to `main`
- `staging` — auto-approve, used on schedule and manual runs
- `production` — **require manual approval** from a designated reviewer

Each environment needs:
- `ANSIBLE_VAULT_PASSWORD` secret
- `ANSIBLE_SSH_PRIVATE_KEY` secret

---

## Terraform Integration

### Workflow

```
GitHub Actions (TF repo)
  └─ terraform apply
       └─ calls ansible-iaas provision.yml workflow (workflow_call)
            └─ tf_inventory.py converts TF outputs → Ansible inventory
                 └─ ansible-playbook provision.yml -i tf_hosts.yml
```

### Required Terraform output

Add to your Terraform root module:

```hcl
output "ansible_hosts" {
  value = {
    linux = {
      webservers = {
        for k, v in aws_instance.web : k => {
          ip         = v.private_ip
          extra_vars = {}
        }
      }
    }
  }
}
```

### Manual usage

```bash
# From a Terraform working directory
python3 scripts/tf_inventory.py --tf-dir ../infra > inventories/production/tf_hosts.yml
ansible-playbook playbooks/provision.yml \
  -i inventories/production/hosts.yml \
  -i inventories/production/tf_hosts.yml
```

---

## Team Conventions

1. **Never commit unencrypted vault files.** The `vault.yml` files must be ansible-vault encrypted before `git push`.
2. **Default inventory is development.** Always pass `-i inventories/<env>/hosts.yml` for staging/production.
3. **Check mode before production.** Run with `--check --diff` first. The GitHub workflow exposes this as a toggle.
4. **Tag everything.** Every task and role application has tags so operators can scope runs precisely.
5. **Role defaults are documentation.** Every tuneable variable lives in `roles/<role>/defaults/main.yml` with a comment. Override in `group_vars`, never in the role itself.
6. **`no_log: true` on sensitive tasks.** Any task that uses a secret (linking key, API key) must have `no_log: true` to prevent the value appearing in runner logs.
7. **`failed_when: false` on optional services.** Services that may not be installed (e.g. avahi-daemon) use `failed_when: false` so the play doesn't break on minimal installs.

---

## Adding a New Role

```bash
# 1. Create the skeleton
mkdir -p roles/my_role/{tasks,defaults,handlers,templates,files,meta}

# 2. Write meta/main.yml (platforms, description)
# 3. Write defaults/main.yml (all tuneable variables with sensible defaults)
# 4. Write tasks/main.yml (import sub-task files, apply tags)
# 5. Add the role to playbooks/linux.yml or windows.yml
# 6. Add override vars to inventories/*/group_vars/all/vars.yml as needed
# 7. Run ansible-lint to catch issues before pushing
ansible-lint roles/my_role/
```
