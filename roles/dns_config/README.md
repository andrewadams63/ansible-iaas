# Role: dns_config

Configures DNS resolvers and search domains. On Linux, supports both `systemd-resolved` (default, recommended for modern distros) and direct `resolv.conf` management. On Windows, uses the `ansible.windows.win_dns_client` module.

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |
| Windows Server | 2019, 2022 |

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml).

| Variable | Default | Description |
|----------|---------|-------------|
| `dns_servers` | `["1.1.1.1", "8.8.8.8"]` | List of DNS resolver IPs — **override per environment** |
| `dns_search_domains` | `[]` | List of DNS search domains |
| `dns_resolver_method` | `resolved` | Linux resolver: `resolved` or `resolvconf` |
| `dns_win_adapter_name` | `*` | Windows adapter name (`*` = all adapters) |

### Environment-specific DNS

Override `dns_servers` and `dns_search_domains` in each environment's `group_vars/all/vars.yml`:

```yaml
# inventories/production/group_vars/all/vars.yml
dns_servers:
  - "10.2.0.2"     # Primary internal DNS
  - "10.2.0.3"     # Secondary internal DNS
dns_search_domains:
  - "internal.acme.com"
```

## Resolver methods

### `resolved` (default)

Uses `systemd-resolved`. Writes `/etc/systemd/resolved.conf` from template, ensures the service is running, and symlinks `/etc/resolv.conf` to the stub resolver at `/run/systemd/resolve/stub-resolv.conf`.

Features enabled in the template:
- `DNSSEC=allow-downgrade` — validates where supported, falls back gracefully
- `DNSOverTLS=opportunistic` — encrypts where supported
- `Cache=yes` — local DNS caching

### `resolvconf`

Writes `/etc/resolv.conf` directly. Use this for hosts that do not run `systemd-resolved` (e.g. minimal container base images, legacy systems).

```yaml
# Override for a specific group
dns_resolver_method: resolvconf
```

### Windows

Uses `ansible.windows.win_dns_client` to set DNS server addresses on all adapters. No reboot is required.

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `dns` | All tasks |

## Example usage

```bash
# Apply DNS config to all hosts
ansible-playbook playbooks/site.yml --tags dns

# Apply to production, check first
ansible-playbook playbooks/site.yml \
  --tags dns \
  -i inventories/production/hosts.yml \
  --check --diff

# Use resolv.conf method on a specific group
# In group_vars/mgmt/vars.yml:
#   dns_resolver_method: resolvconf
ansible-playbook playbooks/site.yml --tags dns -l mgmt
```

## Verifying DNS after a run

```bash
# On a Linux host
ansible linux -m command -a "resolvectl status" -l dev-web-01
ansible linux -m command -a "dig internal.acme.com" -l dev-web-01

# On a Windows host
ansible windows -m ansible.windows.win_command -a "nslookup internal.acme.com"
```
