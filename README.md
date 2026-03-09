# ansible-iaas

Infrastructure-as-Code Ansible project for provisioning and ongoing configuration management of Linux and Windows servers. Designed for reliable, idempotent, team-friendly operation — today from the CLI, tomorrow from Ansible Automation Platform.

---

## Table of Contents

- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Inventory](#inventory)
- [Secrets Management](#secrets-management)
  - [Backend: ansible-vault (local/dev)](#backend-ansible-vault-localdev)
  - [Backend: HashiCorp Vault (staging/production)](#backend-hashicorp-vault-stagingproduction)
  - [Switching backends](#switching-backends)
- [Roles](#roles)
- [Playbooks](#playbooks)
- [GitHub Actions](#github-actions)
  - [GitHub Environments setup](#github-environments-setup)
  - [Required secrets per environment](#required-secrets-per-environment)
- [Terraform Integration](#terraform-integration)
- [Team Conventions](#team-conventions)
- [Adding a New Role](#adding-a-new-role)
- [Adding a New Secret](#adding-a-new-secret)

---

## Architecture

```
ansible-iaas/
├── .github/workflows/
│   ├── lint.yml             # PR gate: ansible-lint + syntax-check
│   ├── provision.yml        # Post-Terraform first-boot (workflow_call or manual)
│   └── configure.yml        # Ongoing config management (push + daily schedule)
├── inventories/
│   ├── development/         # Least restrictive; secrets_backend: ansible_vault
│   ├── staging/             # secrets_backend: hashicorp_vault
│   └── production/          # Strictest; secrets_backend: hashicorp_vault
├── playbooks/
│   ├── pre_tasks/
│   │   └── fetch_secrets.yml  # Secrets broker — delegates to ansible-vault or HCP Vault
│   ├── site.yml             # Master playbook
│   ├── provision.yml        # First-boot provisioning
│   ├── linux.yml / windows.yml
│   ├── hardening.yml
│   ├── monitoring.yml
│   └── security_agents.yml
├── roles/
│   ├── common/              # Packages, NTP, sysctl, motd, rsyslog
│   ├── linux_hardening/     # CIS SSH, firewall, auditd, password policy
│   ├── datadog/             # Datadog agent (Linux + Windows)
│   ├── tenable_agent/       # Nessus agent (Linux + Windows)
│   ├── dns_config/          # systemd-resolved / resolv.conf / Windows DNS
│   └── ms_defender/         # Microsoft Defender for Endpoint (Linux + Windows)
├── collections/
│   └── requirements.yml     # Pinned Galaxy collections
├── scripts/
│   └── tf_inventory.py      # Terraform outputs → Ansible YAML inventory
├── .ansible-lint            # ansible-lint profile and skip rules
└── ansible.cfg              # Project defaults (dev inventory, pipelining, fact cache)
```

---

## Quick Start

### Prerequisites

```bash
# Python 3.10+
pip install ansible

# Install pinned Galaxy collections
ansible-galaxy collection install -r collections/requirements.yml -p collections/
```

### Local development (ansible-vault backend)

```bash
# 1. Create your vault password file — never commit this
echo "YOUR_VAULT_PASSWORD" > .vault_pass
chmod 600 .vault_pass

# 2. Encrypt the development vault file
ansible-vault encrypt inventories/development/group_vars/all/vault.yml

# 3. Edit it and populate real values
ansible-vault edit inventories/development/group_vars/all/vault.yml

# 4. Dry run — no changes made
ansible-playbook playbooks/site.yml --check --diff

# 5. Real run against development (safe default)
ansible-playbook playbooks/site.yml
```

### Target a single host or group

```bash
ansible-playbook playbooks/site.yml -l dev-web-01
ansible-playbook playbooks/site.yml -l webservers
```

### Target a specific environment

```bash
# Always pass -i explicitly for staging/production — never rely on the default
ansible-playbook playbooks/site.yml -i inventories/production/hosts.yml
```

### Run only specific capabilities (tags)

```bash
# Single capability
ansible-playbook playbooks/site.yml --tags datadog
ansible-playbook playbooks/site.yml --tags ssh
ansible-playbook playbooks/site.yml --tags tenable

# Multiple
ansible-playbook playbooks/site.yml --tags "common,dns"

# Skip a capability
ansible-playbook playbooks/site.yml --skip-tags mde
```

---

## Inventory

See [inventories/README.md](inventories/README.md) for full details on structure, host variables, and vault file management.

Each environment directory contains:

| File | Purpose |
|------|---------|
| `hosts.yml` | Static host definitions and group membership |
| `group_vars/all/vars.yml` | Plain variables — committed, safe to read |
| `group_vars/all/vault.yml` | Encrypted secrets — **must be ansible-vault encrypted** |
| `group_vars/linux/vars.yml` | Linux-specific overrides |
| `group_vars/windows/vars.yml` | Windows-specific overrides |
| `host_vars/<hostname>.yml` | Per-host overrides |

The default inventory (`ansible.cfg`) points to **development**. An unqualified `ansible-playbook` run never accidentally targets production.

---

## Secrets Management

All playbooks call `playbooks/pre_tasks/fetch_secrets.yml` as a `pre_tasks` include (tagged `always`). This file is the single secrets broker — it reads `secrets_backend` and routes accordingly. **Roles never handle credential lookup directly.**

```
playbook runs
  └─ pre_tasks/fetch_secrets.yml
       ├─ secrets_backend == ansible_vault  → reads from encrypted vault.yml
       └─ secrets_backend == hashicorp_vault → fetches from HCP Vault at runtime
                                               and overrides vars in-memory (no_log)
```

### Backend: ansible-vault (local/dev)

Development defaults to `secrets_backend: ansible_vault`. Secrets are stored in the encrypted `vault.yml` file in `group_vars/all/`.

**Variable naming convention:**

```yaml
# vault.yml (encrypted) — values only
vault_datadog_api_key: "dd-api-abc123"

# vars.yml (plain, committed) — references only, no values
datadog_api_key: "{{ vault_datadog_api_key }}"
```

This keeps variable *names* visible in git history while keeping *values* encrypted.

**Common vault commands:**

```bash
# Encrypt a vault file for the first time
ansible-vault encrypt inventories/development/group_vars/all/vault.yml

# Edit an encrypted file in your $EDITOR
ansible-vault edit inventories/development/group_vars/all/vault.yml

# View without editing
ansible-vault view inventories/development/group_vars/all/vault.yml

# Re-key (rotate the vault password)
ansible-vault rekey inventories/development/group_vars/all/vault.yml

# Decrypt temporarily (be careful — don't commit the result)
ansible-vault decrypt inventories/development/group_vars/all/vault.yml
```

**`.vault_pass` file:**
- Created locally by each operator, never committed (`.gitignore` enforces this)
- `ansible.cfg` sets `vault_password_file = .vault_pass`
- In GitHub Actions, the vault password is written from a GitHub Secret and deleted after the run

### Backend: HashiCorp Vault (staging/production)

Staging and production default to `secrets_backend: hashicorp_vault`. At the start of every play, `fetch_secrets.yml` connects to your HCP Vault server, retrieves secrets, and sets them as in-memory facts with `no_log: true`. The encrypted `vault.yml` files still exist in these environments but can contain placeholder values — they are not read when `hashicorp_vault` is the active backend.

**KV v2 path structure:**

```
<hcv_secrets_mount>/data/<hcv_secrets_path_prefix>/<secret_name>
```

Example for production:
```
secret/data/ansible/production/datadog       → { api_key: "..." }
secret/data/ansible/production/tenable       → { linking_key: "..." }
secret/data/ansible/production/windows_ansible → { username: "...", password: "..." }
```

**Bootstrap Vault with your secrets (run once):**

```bash
# Authenticate to your Vault server first
export VAULT_ADDR="https://vault.internal.acme.com"
vault login   # or set VAULT_TOKEN

# Enable KV v2 if not already enabled
vault secrets enable -path=secret kv-v2

# Write secrets per environment
vault kv put secret/ansible/development/datadog       api_key="YOUR_DD_API_KEY"
vault kv put secret/ansible/development/tenable       linking_key="YOUR_TENABLE_KEY"
vault kv put secret/ansible/development/windows_ansible username="ansible" password="YOUR_WIN_PASS"

vault kv put secret/ansible/staging/datadog           api_key="YOUR_DD_API_KEY"
vault kv put secret/ansible/staging/tenable           linking_key="YOUR_TENABLE_KEY"
vault kv put secret/ansible/staging/windows_ansible   username="ansible" password="YOUR_WIN_PASS"

vault kv put secret/ansible/production/datadog        api_key="YOUR_DD_API_KEY"
vault kv put secret/ansible/production/tenable        linking_key="YOUR_TENABLE_KEY"
vault kv put secret/ansible/production/windows_ansible username="ansible" password="YOUR_WIN_PASS"
```

**Authentication methods:**

| Method | When to use | Required env vars |
|--------|------------|-------------------|
| `approle` | CI/CD (GitHub Actions) — recommended | `VAULT_ROLE_ID`, `VAULT_SECRET_ID` |
| `token` | Local testing with a dev Vault server | `VAULT_TOKEN` |
| `aws_iam` | EC2/ECS with an IAM role attached | None (uses instance metadata) |
| `ldap` | On-prem Vault with AD integration | `VAULT_USERNAME`, `VAULT_PASSWORD` |

**Create an AppRole for CI (run once):**

```bash
# Enable AppRole auth if not already enabled
vault auth enable approle

# Create a policy granting read on ansible secrets
vault policy write ansible-reader - <<EOF
path "secret/data/ansible/*" {
  capabilities = ["read", "list"]
}
EOF

# Create the AppRole
vault write auth/approle/role/ansible-cicd \
  token_policies="ansible-reader" \
  token_ttl=1h \
  token_max_ttl=2h

# Retrieve the role_id (non-sensitive, can be stored as a variable)
vault read auth/approle/role/ansible-cicd/role-id

# Generate a secret_id (sensitive, treat like a password)
vault write -f auth/approle/role/ansible-cicd/secret-id
```

**Local testing with a Vault dev server:**

```bash
# Start a local Vault dev server (data is in-memory, resets on restart)
vault server -dev &
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="root"

# Write test secrets
vault kv put secret/ansible/development/datadog api_key="test-dd-key"
vault kv put secret/ansible/development/tenable linking_key="test-tenable-key"

# Run with HashiCorp Vault even locally
ansible-playbook playbooks/site.yml \
  --extra-vars "secrets_backend=hashicorp_vault hcv_addr=http://127.0.0.1:8200"
```

### Switching backends

Override `secrets_backend` at runtime without changing any files:

```bash
# Force HCP Vault even in development
ansible-playbook playbooks/site.yml \
  --extra-vars "secrets_backend=hashicorp_vault"

# Force ansible-vault in staging (useful for debugging without Vault access)
ansible-playbook playbooks/site.yml \
  -i inventories/staging/hosts.yml \
  --extra-vars "secrets_backend=ansible_vault"
```

The `secrets_backend` variable is set per environment in `group_vars/all/vars.yml`:

| Environment | Default backend | Rationale |
|-------------|-----------------|-----------|
| `development` | `ansible_vault` | Works offline, no Vault server required |
| `staging` | `hashicorp_vault` | Mirrors production secret handling |
| `production` | `hashicorp_vault` | No plaintext secrets on disk or in CI env |

---

## Roles

Each role has its own `README.md` with full variable documentation. See:

| Role | README | Tags |
|------|--------|------|
| `common` | [roles/common/README.md](roles/common/README.md) | `common`, `ntp`, `packages`, `sysctl` |
| `linux_hardening` | [roles/linux_hardening/README.md](roles/linux_hardening/README.md) | `hardening`, `ssh`, `firewall`, `auditd`, `password`, `services` |
| `datadog` | [roles/datadog/README.md](roles/datadog/README.md) | `datadog`, `monitoring` |
| `tenable_agent` | [roles/tenable_agent/README.md](roles/tenable_agent/README.md) | `tenable`, `security` |
| `dns_config` | [roles/dns_config/README.md](roles/dns_config/README.md) | `dns` |
| `ms_defender` | [roles/ms_defender/README.md](roles/ms_defender/README.md) | `mde`, `defender`, `security` |

**Key principle:** Role defaults (`roles/<role>/defaults/main.yml`) are documentation — every tuneable variable is defined there with a comment. Override in `group_vars`, never in the role itself.

---

## Playbooks

See [playbooks/README.md](playbooks/README.md) for full usage examples.

| Playbook | Hosts | Use case |
|----------|-------|---------|
| `site.yml` | all | Full desired-state convergence |
| `provision.yml` | all | First-boot after Terraform creates machines |
| `linux.yml` | linux | All Linux roles in dependency order |
| `windows.yml` | windows | All Windows roles |
| `hardening.yml` | linux | Hardening-only runs or audits |
| `monitoring.yml` | all | Datadog agent update/reinstall |
| `security_agents.yml` | all | Tenable + MDE update/reinstall |

---

## GitHub Actions

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `lint.yml` | Pull request to `main` | ansible-lint + syntax-check on every playbook |
| `provision.yml` | `workflow_call` from TF repo, or manual | Post-Terraform first-boot; supports check mode |
| `configure.yml` | Push to `main`, daily 06:00 UTC, manual | Config management pipeline: dev → staging → prod |

### GitHub Environments setup

In **Settings → Environments**, create three environments:

| Environment | Protection rules | Used by |
|------------|-----------------|---------|
| `development` | None (auto-approve) | Every push to `main` |
| `staging` | None (auto-approve) | Daily schedule, manual |
| `production` | **Required reviewer approval** | Daily schedule, manual |

The `production` environment gate means no production Ansible run can start without a human approving it in the GitHub UI, even on a schedule.

### Required secrets per environment

Set these in **Settings → Environments → \<env\> → Secrets**:

| Secret | Required when | Description |
|--------|--------------|-------------|
| `ANSIBLE_SSH_PRIVATE_KEY` | Always | SSH private key for the `ansible` service account |
| `ANSIBLE_VAULT_PASSWORD` | `secrets_backend: ansible_vault` | Decrypts the `vault.yml` files |
| `VAULT_ADDR` | `secrets_backend: hashicorp_vault` | e.g. `https://vault.internal.acme.com` |
| `VAULT_ROLE_ID` | `secrets_backend: hashicorp_vault` (AppRole) | AppRole role_id |
| `VAULT_SECRET_ID` | `secrets_backend: hashicorp_vault` (AppRole) | AppRole secret_id — treat as a password |
| `VAULT_TOKEN` | `secrets_backend: hashicorp_vault` (token auth) | Token auth fallback — dev Vault servers only |

**Recommended configuration:**
- `development`: set `ANSIBLE_SSH_PRIVATE_KEY` + `ANSIBLE_VAULT_PASSWORD`
- `staging`: set `ANSIBLE_SSH_PRIVATE_KEY` + `VAULT_ADDR` + `VAULT_ROLE_ID` + `VAULT_SECRET_ID`
- `production`: same as staging

The workflows are written so that unused secrets (e.g. `VAULT_TOKEN` when using AppRole) are simply empty and do not cause errors.

### Calling `provision.yml` from your Terraform workflow

In your Terraform GitHub Actions workflow, after `terraform apply`:

```yaml
- name: Call Ansible provision
  uses: ./.github/workflows/provision.yml   # or org/ansible-iaas/.github/workflows/provision.yml@main
  with:
    environment: production
    terraform_outputs: ${{ steps.tf.outputs.stdout }}
  secrets: inherit
```

---

## Terraform Integration

### Full workflow

```
GitHub Actions (Terraform repo)
  └─ terraform apply
       └─ terraform output -json → JSON passed as workflow input
            └─ ansible-iaas/.github/workflows/provision.yml
                 └─ scripts/tf_inventory.py converts JSON → YAML inventory
                      └─ ansible-playbook provision.yml
                           ├─ waits for SSH connectivity
                           └─ runs site.yml against new hosts
```

### Required Terraform output

Add to your Terraform root module. The `ansible_hosts` output shape is the contract with `tf_inventory.py`:

```hcl
output "ansible_hosts" {
  description = "Host map consumed by scripts/tf_inventory.py"
  value = {
    linux = {
      webservers = {
        for k, v in aws_instance.web : k => {
          ip         = v.private_ip
          extra_vars = {}
        }
      }
      dbservers = {
        for k, v in aws_instance.db : k => {
          ip         = v.private_ip
          extra_vars = { datadog_tags = ["role:db"] }
        }
      }
    }
    windows = {
      win_servers = {
        for k, v in aws_instance.win : k => {
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
# Generate inventory from live Terraform state
python3 scripts/tf_inventory.py --tf-dir ../infra > inventories/production/tf_hosts.yml

# Or from a pre-exported JSON file
terraform -chdir=../infra output -json > /tmp/tf_out.json
python3 scripts/tf_inventory.py /tmp/tf_out.json > inventories/production/tf_hosts.yml

# Run against the generated inventory alongside the static one
ansible-playbook playbooks/provision.yml \
  -i inventories/production/hosts.yml \
  -i inventories/production/tf_hosts.yml
```

---

## Team Conventions

1. **Never commit unencrypted vault files.** Every `vault.yml` must be ansible-vault encrypted before `git push`. The CI lint job will catch unencrypted vault files.

2. **Default inventory is development.** The `ansible.cfg` default protects production. Always pass `-i inventories/<env>/hosts.yml` for staging and production.

3. **Check mode before any production change.** Run with `--check --diff` first to preview every change. The GitHub workflow exposes this as a toggle on manual dispatch.

4. **Tag everything.** Every task block and role application in every playbook has tags. This allows targeted runs without separate playbooks for every scenario.

5. **Role defaults are documentation.** Every tuneable lives in `roles/<role>/defaults/main.yml` with a comment. Override in `group_vars`, never edit role defaults for environment-specific values.

6. **`no_log: true` on sensitive tasks.** Any task that references a secret (API key, linking key, password) must carry `no_log: true`. This prevents values appearing in runner logs, Ansible Tower job output, and CI run summaries.

7. **`failed_when: false` on optional services.** Services that may not be installed (e.g. `avahi-daemon`) use `failed_when: false` so the play continues on minimal installs.

8. **`secrets_backend` is the single lever.** Never hard-code credentials. If you need to test with a different secret source, use `--extra-vars secrets_backend=...`. Do not duplicate secret resolution logic outside `fetch_secrets.yml`.

9. **Pin collection versions.** Update `collections/requirements.yml` deliberately and test before merging. Unpinned collections can break runs on the next `galaxy install`.

10. **One PR per logical change.** The lint PR gate runs on every PR. Keep changes focused so failures are easy to trace.

---

## Adding a New Role

```bash
# 1. Create the directory skeleton
mkdir -p roles/my_role/{tasks,defaults,handlers,templates,files,meta}

# 2. Write meta/main.yml — platforms, description, dependencies
# 3. Write defaults/main.yml — ALL tuneable variables with comments and sensible defaults
# 4. Write tasks/main.yml — import sub-task files (ssh.yml, linux.yml, etc.), apply tags
# 5. Write handlers/main.yml if any service restarts are needed
# 6. Add the role to playbooks/linux.yml or windows.yml with appropriate tags
# 7. Add override vars to inventories/*/group_vars/all/vars.yml
# 8. Write roles/my_role/README.md (see existing role READMEs for the template)
# 9. Lint before pushing
ansible-lint roles/my_role/
```

---

## Adding a New Secret

When you introduce a new secret to a role:

**Step 1 — Add to `fetch_secrets.yml`**

```yaml
- name: "[secrets] Fetch my_service secrets"
  ansible.builtin.set_fact:
    my_service_api_key: >-
      {{ lookup('community.hashi_vault.hashi_vault',
           hcv_secrets_mount + '/data/' + hcv_secrets_path_prefix + '/my_service',
           **_hcv_kwargs).data.data.api_key }}
  no_log: true
```

**Step 2 — Add the `vault_` placeholder to every `vault.yml`**

```bash
for env in development staging production; do
  ansible-vault edit inventories/$env/group_vars/all/vault.yml
  # Add: vault_my_service_api_key: "REPLACE_ME"
done
```

**Step 3 — Reference it in `vars.yml`**

```yaml
my_service_api_key: "{{ vault_my_service_api_key }}"
```

**Step 4 — Bootstrap Vault**

```bash
vault kv put secret/ansible/development/my_service  api_key="..."
vault kv put secret/ansible/staging/my_service      api_key="..."
vault kv put secret/ansible/production/my_service   api_key="..."
```
