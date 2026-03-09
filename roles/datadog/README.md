# Role: datadog

Installs and configures the Datadog Agent v7 on Linux (via apt/yum repository) and Windows (via MSI installer).

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |
| Windows Server | 2019, 2022 |

## Secrets

This role requires `datadog_api_key`. It is populated automatically by `playbooks/pre_tasks/fetch_secrets.yml` — do not reference the vault variable directly in role tasks.

| Secret variable | Vault path (HCP) | vault.yml key (ansible-vault) |
|----------------|-----------------|-------------------------------|
| `datadog_api_key` | `<prefix>/datadog → api_key` | `vault_datadog_api_key` |

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml).

| Variable | Default | Description |
|----------|---------|-------------|
| `datadog_api_key` | *(from secrets)* | Datadog API key — populated by fetch_secrets.yml |
| `datadog_site` | `datadoghq.com` | Datadog intake site (`datadoghq.eu` for EU customers) |
| `datadog_agent_major_version` | `7` | Major version of the Datadog agent to install |
| `datadog_tags` | `[]` | List of `key:value` tags applied to all metrics |
| `datadog_checks` | `{}` | Integration check configs (see Datadog docs) |
| `datadog_process_agent_enabled` | `true` | Enable process-level monitoring |
| `datadog_apm_enabled` | `false` | Enable APM tracing |
| `datadog_logs_enabled` | `true` | Enable log collection |
| `datadog_logs_config_container_collect_all` | `false` | Auto-collect all container logs |

### Tags applied to every host

The `datadog_tags` list is set per environment in `group_vars/all/vars.yml`:

```yaml
# development
datadog_tags:
  - "env:development"
  - "team:platform"
  - "ansible_managed:true"   # added by template

# production
datadog_tags:
  - "env:production"
  - "team:platform"
```

Additional tags can be added per-group or per-host:

```yaml
# group_vars/webservers/vars.yml
datadog_tags:
  - "env:production"
  - "team:platform"
  - "role:web"
```

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `datadog` | All tasks |
| `monitoring` | All tasks (alias, used in site.yml) |

## Template

| Template | Destination | Description |
|----------|-------------|-------------|
| `datadog.yaml.j2` | `/etc/datadog-agent/datadog.yaml` | Main agent config: API key, site, tags, feature toggles |

## Enabling integrations (checks)

To enable a Datadog integration, add its config to `datadog_checks` in group_vars:

```yaml
# inventories/production/group_vars/dbservers/vars.yml
datadog_checks:
  postgres:
    init_config: {}
    instances:
      - host: localhost
        port: 5432
        username: datadog
        password: "{{ vault_dd_postgres_password }}"
```

Note: the check config is not yet wired to the role — add a task to `tasks/linux.yml` to write the check file to `/etc/datadog-agent/conf.d/`.

## Example usage

```bash
# Install/update Datadog on all hosts
ansible-playbook playbooks/monitoring.yml

# Datadog only, production webservers
ansible-playbook playbooks/monitoring.yml \
  -i inventories/production/hosts.yml \
  -l webservers \
  --check --diff

# Apply just the Datadog tag within a full site run
ansible-playbook playbooks/site.yml --tags datadog
```
