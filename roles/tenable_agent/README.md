# Role: tenable_agent

Installs the Tenable Nessus Agent and links it to your Tenable manager (cloud.tenable.com or on-prem Security Center). Idempotent — checks whether the agent is already linked before running the link command.

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |
| Windows Server | 2019, 2022 |

## Secrets

| Secret variable | Vault path (HCP) | vault.yml key (ansible-vault) |
|----------------|-----------------|-------------------------------|
| `tenable_linking_key` | `<prefix>/tenable → linking_key` | `vault_tenable_linking_key` |

The linking key is used with `no_log: true` — it will not appear in any Ansible output or CI logs.

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml).

| Variable | Default | Description |
|----------|---------|-------------|
| `tenable_linking_key` | *(from secrets)* | Agent linking key from Tenable.io or Security Center |
| `tenable_manager_host` | `cloud.tenable.com` | Hostname of the Tenable manager |
| `tenable_manager_port` | `443` | Port for the Tenable manager |
| `tenable_agent_group` | `default` | Agent group name in the Tenable console |
| `tenable_agent_version` | `10.7.3` | Version of the Nessus agent to install |
| `tenable_agent_rpm_url` | *(computed)* | Download URL for the EL RPM |
| `tenable_agent_deb_url` | *(computed)* | Download URL for the Debian/Ubuntu DEB |
| `tenable_agent_win_url` | *(computed)* | Download URL for the Windows MSI |
| `tenable_download_dir` | `/tmp` | Temporary directory for installer downloads |

### Pinning the agent version

The URLs are constructed from `tenable_agent_version`. To upgrade:

```yaml
# group_vars/all/vars.yml
tenable_agent_version: "10.8.0"
```

Then run:

```bash
ansible-playbook playbooks/security_agents.yml --tags tenable
```

The role downloads and installs the package — if the version is already installed, the package manager will skip it (idempotent).

### Agent groups per environment

Separate your environments in the Tenable console using the `tenable_agent_group` variable:

```yaml
# inventories/development/group_vars/all/vars.yml
tenable_agent_group: "development"

# inventories/production/group_vars/all/vars.yml
tenable_agent_group: "production"
```

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `tenable` | All tasks |
| `security` | All tasks (alias, used in site.yml) |

## Idempotency

Before running the link command, the role checks:

```bash
/opt/nessus_agent/sbin/nessuscli agent status
```

If the output contains `"Linked to"`, the link task is skipped. The agent is only re-linked if it becomes unlinked (e.g. after a deactivation or re-image).

## Example usage

```bash
# Install/re-link on all hosts
ansible-playbook playbooks/security_agents.yml --tags tenable

# Upgrade agent version — check first
ansible-playbook playbooks/security_agents.yml \
  --tags tenable \
  -i inventories/production/hosts.yml \
  --check --diff

# Single host debug
ansible-playbook playbooks/security_agents.yml \
  --tags tenable \
  -l prod-web-01 \
  -vv
```

## Getting the linking key

1. Log in to **Tenable.io** or **Tenable Security Center**
2. Navigate to **Settings → Sensors → Nessus Agents**
3. Click **Link Agent** and copy the linking key
4. Store it in your vault:
   ```bash
   ansible-vault edit inventories/production/group_vars/all/vault.yml
   # Set: vault_tenable_linking_key: "YOUR_KEY_HERE"
   ```
   Or in HashiCorp Vault:
   ```bash
   vault kv put secret/ansible/production/tenable linking_key="YOUR_KEY_HERE"
   ```
