# Role: common

Baseline configuration applied to **every managed host** before any other role runs. Handles package installation, NTP, kernel parameters, MOTD, and log shipping bootstrap.

## Dependency

This role is declared as a dependency in `linux_hardening/meta/main.yml` so it runs automatically before hardening. It should also be listed first in any playbook that applies Linux roles directly.

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml). Override in `inventories/<env>/group_vars/`.

| Variable | Default | Description |
|----------|---------|-------------|
| `common_packages` | `[vim, curl, wget, unzip, ...]` | Packages installed on every host |
| `common_packages_extra` | `[]` | Append-only list for group/host additions |
| `ntp_servers` | `[time.cloudflare.com, time.google.com]` | NTP pool servers for chrony |
| `common_timezone` | `UTC` | System timezone (use `timedatectl list-timezones`) |
| `common_motd_path` | `/etc/motd` | Path to the MOTD file |
| `ansible_service_account` | `ansible` | The service account Ansible connects as |
| `common_sysctl_settings` | See defaults | Dict of `sysctl` key → value pairs |

### Sysctl defaults applied

| Key | Value | Rationale |
|-----|-------|-----------|
| `net.ipv4.tcp_syncookies` | `1` | Protects against SYN flood attacks |
| `net.ipv4.conf.all.rp_filter` | `1` | Blocks source-routed packets |
| `net.ipv4.conf.all.accept_redirects` | `0` | Ignores ICMP redirects |
| `net.ipv4.conf.all.send_redirects` | `0` | Does not send ICMP redirects |
| `net.ipv6.conf.all.accept_redirects` | `0` | Ignores IPv6 ICMP redirects |
| `kernel.dmesg_restrict` | `1` | Restricts dmesg to privileged users |

Override the entire dict or add entries:

```yaml
# group_vars/all/vars.yml
common_sysctl_settings:
  net.ipv4.tcp_syncookies: 1
  net.core.somaxconn: 65535   # Added for high-traffic web servers
```

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `common` | All tasks in this role |
| `timezone` | Set system timezone |
| `packages` | Package installation |
| `ntp` | Chrony config + service |
| `sysctl` | Kernel parameter tuning |
| `motd` | Message of the day |
| `logging` | rsyslog service |

## Templates

| Template | Destination | Description |
|----------|-------------|-------------|
| `chrony.conf.j2` | `/etc/chrony.conf` | NTP server list |
| `motd.j2` | `/etc/motd` | Login banner showing hostname and environment |

## Example: adding extra packages per group

```yaml
# inventories/production/group_vars/linux/vars.yml
common_packages_extra:
  - strace
  - tcpdump
  - sysstat
```

## Example: standalone usage

```bash
# Apply common role only
ansible-playbook playbooks/site.yml --tags common

# Apply to a single host
ansible-playbook playbooks/site.yml --tags common -l dev-web-01

# Check what NTP changes would be made
ansible-playbook playbooks/site.yml --tags ntp --check --diff
```
