# Inventories

This directory contains one sub-directory per environment. Each follows the same structure so that behaviour is predictable and auditable across environments.

## Structure

```
inventories/
├── development/
│   ├── hosts.yml                       # Static host definitions
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── vars.yml                # Plain variables — committed, visible
│   │   │   └── vault.yml              # Encrypted secrets — must be vault-encrypted
│   │   ├── linux/
│   │   │   └── vars.yml                # Variables for all Linux hosts
│   │   └── windows/
│   │       └── vars.yml                # Variables for all Windows hosts
│   └── host_vars/
│       └── <hostname>.yml              # Per-host overrides (create as needed)
├── staging/                            # Same structure
└── production/                         # Same structure
```

## Environment comparison

| Setting | development | staging | production |
|---------|------------|---------|------------|
| `secrets_backend` | `ansible_vault` | `hashicorp_vault` | `hashicorp_vault` |
| `ssh_max_auth_tries` | 6 | 4 | 3 |
| `auditd_num_logs` | 5 | 5 | 10 |
| WinRM cert validation | `ignore` | `ignore` | `validate` |
| WinRM port | 5985 (HTTP) | 5985 | 5986 (HTTPS) |

## Variable precedence (Ansible standard)

From lowest to highest priority:

1. Role defaults (`roles/<role>/defaults/main.yml`)
2. Inventory `group_vars/all/` (`vars.yml`)
3. Inventory `group_vars/<group>/` (`vars.yml`)
4. Inventory `host_vars/<hostname>/`
5. `--extra-vars` on the command line (highest)

**Never put environment-specific values in role defaults.** Role defaults document what variables exist and their safe fallback values. All environment customisation belongs in `group_vars`.

---

## Managing hosts

### Adding a host

Edit `hosts.yml` in the appropriate environment:

```yaml
# inventories/production/hosts.yml
all:
  children:
    linux:
      children:
        webservers:
          hosts:
            prod-web-01:
              ansible_host: 10.2.1.10
            prod-web-02:
              ansible_host: 10.2.1.11
```

### Adding a host-specific override

Create `host_vars/<hostname>.yml`:

```yaml
# inventories/production/host_vars/prod-web-01.yml
datadog_tags:
  - "env:production"
  - "team:platform"
  - "role:web"
  - "region:us-east-1a"
```

### Testing connectivity

```bash
# Ping all hosts in development
ansible all -m ping

# Ping a specific group in production
ansible webservers -m ping -i inventories/production/hosts.yml

# Show gathered facts for a host
ansible dev-web-01 -m setup | less
```

---

## Secrets management

### Convention

All secrets follow this two-file pattern:

**`vault.yml`** (encrypted) — values only, `vault_` prefix:
```yaml
vault_datadog_api_key: "dd-api-abc123"
vault_tenable_linking_key: "nessus-link-xyz"
```

**`vars.yml`** (plain, committed) — references only, no values:
```yaml
datadog_api_key: "{{ vault_datadog_api_key }}"
tenable_linking_key: "{{ vault_tenable_linking_key }}"
```

This makes variable *names* visible in git (useful for code review) while keeping *values* encrypted.

### ansible-vault backend (development default)

**Initial setup for a new operator:**

```bash
# 1. Get the vault password from your password manager / 1Password / team lead
echo "THE_VAULT_PASSWORD" > .vault_pass
chmod 600 .vault_pass

# 2. Verify you can read the vault file
ansible-vault view inventories/development/group_vars/all/vault.yml
```

**Day-to-day vault operations:**

```bash
# Edit secrets
ansible-vault edit inventories/development/group_vars/all/vault.yml

# View secrets (read-only)
ansible-vault view inventories/development/group_vars/all/vault.yml

# Encrypt a new vault file (after creating it in plain text)
ansible-vault encrypt inventories/development/group_vars/all/vault.yml

# Rotate the vault password (inform all operators)
ansible-vault rekey inventories/development/group_vars/all/vault.yml

# Check a vault file is actually encrypted (should show $ANSIBLE_VAULT;...)
head -1 inventories/development/group_vars/all/vault.yml
```

**Never commit an unencrypted vault file.** If you accidentally decrypt it, re-encrypt before staging:

```bash
ansible-vault encrypt inventories/development/group_vars/all/vault.yml
git diff --stat   # verify the file shows $ANSIBLE_VAULT header, not plain text
```

### HashiCorp Vault backend (staging/production default)

When `secrets_backend: hashicorp_vault`, the `vault.yml` files in those environments are not read at runtime. They can contain `REPLACE_ME` placeholders. The real secrets live in HCP Vault and are fetched by `playbooks/pre_tasks/fetch_secrets.yml`.

See the [root README secrets section](../README.md#secrets-management) for full HCP Vault bootstrap instructions and auth method setup.

**KV path convention for this project:**

```
secret/data/ansible/<env>/<service>
                              ├── development/datadog        → { api_key }
                              ├── development/tenable        → { linking_key }
                              ├── development/windows_ansible → { username, password }
                              ├── staging/...
                              └── production/...
```

**Verify HCP Vault connectivity before a run:**

```bash
export VAULT_ADDR="https://vault.internal.acme.com"
export VAULT_TOKEN="..."   # or set up AppRole

# Check the secret exists
vault kv get secret/ansible/production/datadog

# Test the exact lookup the role will make
python3 -c "
import hvac
c = hvac.Client(url='$VAULT_ADDR', token='$VAULT_TOKEN')
print(c.secrets.kv.v2.read_secret_version('ansible/production/datadog', mount_point='secret'))
"
```

---

## group_vars reference

### `all/vars.yml` — key variables

| Variable | Description | Override location |
|----------|-------------|-------------------|
| `secrets_backend` | `ansible_vault` or `hashicorp_vault` | Per-environment vars.yml |
| `hcv_addr` | HashiCorp Vault URL | Per-environment vars.yml |
| `hcv_auth_method` | Vault auth method | Per-environment vars.yml |
| `hcv_secrets_mount` | KV v2 mount point | Per-environment vars.yml |
| `hcv_secrets_path_prefix` | Path prefix for this env's secrets | Per-environment vars.yml |
| `env` | Environment name string (`development`, etc.) | Per-environment vars.yml |
| `org_name` | Organisation name used in MOTD | Per-environment vars.yml |
| `datadog_api_key` | References `vault_datadog_api_key` | Populated by fetch_secrets.yml |
| `datadog_tags` | List of `key:value` tags | Per-environment vars.yml |
| `tenable_linking_key` | References `vault_tenable_linking_key` | Populated by fetch_secrets.yml |
| `tenable_agent_group` | Agent group in Tenable console | Per-environment vars.yml |
| `dns_servers` | List of DNS resolver IPs | Per-environment vars.yml |
| `dns_search_domains` | List of DNS search domains | Per-environment vars.yml |
| `ntp_servers` | List of NTP pool servers | Per-environment vars.yml |
| `ssh_permit_root_login` | sshd_config value | Per-environment vars.yml |
| `ssh_password_authentication` | sshd_config value | Per-environment vars.yml |
| `ssh_max_auth_tries` | sshd_config value | Per-environment vars.yml |

### `linux/vars.yml`

| Variable | Description |
|----------|-------------|
| `ansible_user` | SSH user (`ansible`) |
| `ansible_become` | Enable privilege escalation |
| `auditd_num_logs` | Rotated audit log count |

### `windows/vars.yml`

| Variable | Description |
|----------|-------------|
| `ansible_user` | WinRM user (from vault) |
| `ansible_password` | WinRM password (from vault) |
| `ansible_connection` | `winrm` |
| `ansible_winrm_transport` | `kerberos` (prod) or `ntlm` (testing) |
| `ansible_winrm_server_cert_validation` | `validate` (prod) or `ignore` (dev) |
| `ansible_port` | `5986` (prod, HTTPS) or `5985` (dev, HTTP) |
