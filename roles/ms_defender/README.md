# Role: ms_defender

Installs and configures Microsoft Defender for Endpoint (MDE). On Linux, installs the `mdatp` package via the Microsoft package repository and onboards using the script from the MDE portal. On Windows, configures built-in Windows Defender via PowerShell.

## Platforms

| OS | Versions |
|----|---------|
| RHEL / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |
| Windows Server | 2019, 2022 |

## Prerequisites

### Linux: onboarding package

Download the **MDE Linux server onboarding package** from the Microsoft 365 Defender portal:

1. Go to **Settings → Endpoints → Device management → Onboarding**
2. Select **Linux Server** and download the onboarding package
3. Extract the `.py` script and place it at:

```
roles/ms_defender/files/MicrosoftDefenderATPOnboardingLinuxServer.py
```

This file is referenced by `mde_onboarding_script_linux`. It contains your tenant's onboarding blob and is specific to your organisation. **Do not commit it to a public repository.** For private repos it is acceptable to commit; for extra security, store it in an artifact repository or S3 and download it in a pre-task.

### Windows: onboarding package (optional)

Windows onboarding via this role uses the built-in Defender service — no separate onboarding package is required for basic configuration. Full MDE onboarding for Windows typically uses Intune, MECM, or a group policy script.

## Role Variables

All defaults are in [defaults/main.yml](defaults/main.yml).

| Variable | Default | Description |
|----------|---------|-------------|
| `mde_onboarding_script_linux` | `MicrosoftDefenderATPOnboardingLinuxServer.py` | Filename in `roles/ms_defender/files/` |
| `mde_channel` | `prod` | Release channel: `prod`, `insiders-slow`, `insiders-fast` |
| `mde_realtime_protection` | `true` | Enable real-time protection |
| `mde_cloud_protection` | `true` | Enable cloud-delivered protection |
| `mde_exclusions_paths` | `[]` | File paths to exclude from scanning |
| `mde_exclusions_extensions` | `[]` | File extensions to exclude |
| `mde_exclusions_processes` | `[]` | Process names to exclude |

### Managed configuration

The `mdatp_managed.json.j2` template writes a managed policy to `/etc/opt/microsoft/mdatp/managed/mdatp_managed.json`. Settings in this file cannot be overridden by local users, making it suitable for enforcing a security baseline.

### Exclusions

Add exclusions for applications that generate false positives:

```yaml
# group_vars/dbservers/vars.yml
mde_exclusions_paths:
  - "/var/lib/postgresql"
  - "/var/lib/mysql"
mde_exclusions_processes:
  - "postgres"
  - "mysqld"
```

Exclusions are applied via the managed config on Linux and via `Add-MpPreference` on Windows.

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `mde` | All tasks |
| `defender` | All tasks (alias) |
| `security` | All tasks (alias, used in site.yml) |

## Template

| Template | Destination | Description |
|----------|-------------|-------------|
| `mdatp_managed.json.j2` | `/etc/opt/microsoft/mdatp/managed/mdatp_managed.json` | Managed policy: RTP, cloud protection, exclusions |

## Checking MDE status after installation

```bash
# Linux
ansible linux -m command -a "mdatp health" -l prod-web-01
ansible linux -m command -a "mdatp threat list" -l prod-web-01

# Windows
ansible windows -m ansible.windows.win_powershell \
  -a "script='Get-MpComputerStatus | Select-Object AMRunningMode,RealTimeProtectionEnabled'"
```

## Example usage

```bash
# Install/update MDE on all hosts
ansible-playbook playbooks/security_agents.yml --tags mde

# Linux only, check first
ansible-playbook playbooks/security_agents.yml \
  --tags mde \
  -l linux \
  -i inventories/production/hosts.yml \
  --check --diff

# Update exclusions only (re-applies managed config)
ansible-playbook playbooks/security_agents.yml \
  --tags mde \
  -l dbservers
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `mdatp health` shows `healthy: false` | Verify the onboarding script completed: check for `/etc/opt/microsoft/mdatp/mdatp_onboard.json` |
| High CPU from `mdatp_essentials` | Add exclusions for known-safe high-write paths |
| Package install fails on EL | Ensure the Microsoft GPG key was imported correctly |
| Onboarding script fails | Check that the `.py` file matches your current tenant (re-download from portal) |
