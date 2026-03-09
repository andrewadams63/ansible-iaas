# Role: linux_hardening

CIS Benchmark-aligned Linux hardening. Covers SSH configuration, host firewall, audit daemon, password quality policy, and service minimization. Depends on the `common` role (declared in `meta/main.yml`).

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |

## Task files

| File | What it does |
|------|-------------|
| `tasks/ssh.yml` | Writes `sshd_config` from template, validates with `sshd -t`, regenerates host keys if missing |
| `tasks/firewall.yml` | firewalld (EL) or ufw (Debian/Ubuntu) — default deny, allow listed TCP ports |
| `tasks/auditd.yml` | Installs auditd, writes `auditd.conf`, deploys CIS audit rules |
| `tasks/password_policy.yml` | Installs pwquality, writes `pwquality.conf` |
| `tasks/services.yml` | Stops and disables services listed in `services_to_disable` |

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml).

### SSH

| Variable | Default | Description |
|----------|---------|-------------|
| `ssh_port` | `22` | Port sshd listens on |
| `ssh_permit_root_login` | `no` | Disallow root SSH login |
| `ssh_password_authentication` | `no` | Keys only |
| `ssh_max_auth_tries` | `4` | Reduced in production via group_vars |
| `ssh_login_grace_time` | `60` | Seconds before unauthenticated connections are dropped |
| `ssh_client_alive_interval` | `300` | Idle timeout in seconds |
| `ssh_client_alive_count_max` | `2` | Max missed keepalives before disconnect |
| `ssh_allowed_users` | `ansible` | Space-separated `AllowUsers` list |
| `ssh_x11_forwarding` | `no` | X11 forwarding disabled |

Only modern ciphers are permitted (ChaCha20-Poly1305, AES-256-GCM, AES-128-GCM). The template is validated with `sshd -t` before it is written, preventing a bad config from locking you out.

### Firewall

| Variable | Default | Description |
|----------|---------|-------------|
| `firewall_allowed_tcp_ports` | `["22"]` | List of TCP ports to permit |

```yaml
# Example: allow SSH + HTTPS
firewall_allowed_tcp_ports:
  - "22"
  - "443"
```

### Auditd

| Variable | Default | Description |
|----------|---------|-------------|
| `auditd_max_log_file` | `8` | Maximum log file size (MB) |
| `auditd_num_logs` | `5` (dev) / `10` (prod) | Number of rotated log files to keep |
| `auditd_log_max_file_action` | `rotate` | Action when max log size reached |
| `auditd_space_left_action` | `email` | Action when disk space gets low |
| `auditd_admin_space_left_action` | `halt` | Action at critical disk space |
| `auditd_disk_full_action` | `halt` | Action when disk is full |

**Audit rules deployed** (`files/audit.rules`):
- Identity files: `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers`
- Login/logout tracking: `faillog`, `lastlog`, `tallylog`
- SSH config changes
- Privileged command execution: `sudo`, `su`, `newgrp`, `chsh`, `chage`
- Filesystem mounts and file deletion syscalls
- Kernel module load/unload

### Password policy (pwquality)

| Variable | Default | Description |
|----------|---------|-------------|
| `password_min_length` | `14` | Minimum password length |
| `password_min_class` | `4` | Minimum character classes (upper, lower, digit, special) |
| `password_max_repeat` | `3` | Max consecutive identical characters |
| `password_remember` | `5` | Number of previous passwords remembered |
| `password_max_days` | `90` | Maximum password age (days) |
| `password_min_days` | `1` | Minimum days before password can be changed |
| `password_warn_days` | `14` | Days before expiry to warn user |

### Service minimization

| Variable | Default | Description |
|----------|---------|-------------|
| `services_to_disable` | `[avahi-daemon, cups, rpcbind, nfs, bluetooth]` | Services stopped and disabled |

Services that are not installed are silently skipped (`failed_when: false`).

### umask

| Variable | Default | Description |
|----------|---------|-------------|
| `common_umask` | `027` | Default umask written to `/etc/profile.d/umask.sh` |

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `hardening` | All tasks in this role |
| `ssh` | sshd_config, banner, host keys |
| `firewall` | firewalld / ufw rules |
| `auditd` | auditd install, config, rules |
| `password` | pwquality install + config |
| `services` | Service disable list |
| `umask` | Default umask setting |

## Environment differences

The environments set different `ssh_max_auth_tries` values via `group_vars`:

| Environment | `ssh_max_auth_tries` | Rationale |
|-------------|----------------------|-----------|
| development | 6 | Easier debugging |
| staging | 4 | Production-like |
| production | 3 | CIS recommendation |

## Example usage

```bash
# Full hardening run with diff
ansible-playbook playbooks/hardening.yml --check --diff

# SSH config only
ansible-playbook playbooks/hardening.yml --tags ssh

# Audit rules only, single host
ansible-playbook playbooks/hardening.yml --tags auditd -l prod-web-01

# Open an additional port on webservers
# In inventories/production/group_vars/webservers/vars.yml:
# firewall_allowed_tcp_ports: ["22", "443"]
ansible-playbook playbooks/hardening.yml --tags firewall -l webservers
```

## Safety notes

- The SSH config template is validated with `sshd -t` before being applied. A syntax error in the template will fail the task rather than writing a broken config.
- Always run `--check --diff` before applying SSH changes to a host you need to maintain access to.
- The `auditd_admin_space_left_action: halt` and `auditd_disk_full_action: halt` defaults are CIS-compliant but will halt the system if the disk fills up. Adjust `auditd_num_logs` and `auditd_max_log_file` if disk space is constrained.
