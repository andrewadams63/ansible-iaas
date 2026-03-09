# Playbooks

This directory contains all runnable Ansible playbooks. Every playbook includes `pre_tasks/fetch_secrets.yml` (tagged `always`) before any roles execute, so the correct secret backend is always active.

## Playbook catalogue

| Playbook | Hosts | Purpose |
|----------|-------|---------|
| [`site.yml`](#siteyml) | all | Full desired-state convergence — the everyday config management playbook |
| [`provision.yml`](#provisionyml) | all | First-boot after Terraform creates machines |
| [`linux.yml`](#linuxyml) | linux | All Linux roles in dependency order (imported by site.yml) |
| [`windows.yml`](#windowsyml) | windows | All Windows roles (imported by site.yml) |
| [`hardening.yml`](#hardeningyml) | linux | Linux hardening only |
| [`monitoring.yml`](#monitoringyml) | all | Datadog agent install/update only |
| [`security_agents.yml`](#security_agentsyml) | all | Tenable + MDE install/update only |

## Pre-tasks: secrets fetch

Every playbook starts with:

```yaml
pre_tasks:
  - name: Fetch secrets (ansible_vault or hashicorp_vault)
    ansible.builtin.include_tasks: pre_tasks/fetch_secrets.yml
    tags: [always]
```

This runs before any role — even when you use `--tags` to target a subset of tasks. It reads `secrets_backend` and either leaves ansible-vault variables in place (no-op) or fetches from HashiCorp Vault and overrides the variables in-memory.

See [pre_tasks/fetch_secrets.yml](pre_tasks/fetch_secrets.yml) for full inline documentation.

---

## site.yml

The master playbook. Imports `linux.yml` and `windows.yml`. Use this for routine config management runs — it applies the full desired state and is fully idempotent.

```bash
# Full run, development (default inventory)
ansible-playbook playbooks/site.yml

# Dry run — preview all changes
ansible-playbook playbooks/site.yml --check --diff

# Production — always check first
ansible-playbook playbooks/site.yml \
  -i inventories/production/hosts.yml \
  --check --diff

# Then apply
ansible-playbook playbooks/site.yml \
  -i inventories/production/hosts.yml

# Limit to a single host
ansible-playbook playbooks/site.yml -l dev-web-01

# Limit to a group
ansible-playbook playbooks/site.yml -l webservers

# Run a specific capability across all hosts
ansible-playbook playbooks/site.yml --tags datadog
ansible-playbook playbooks/site.yml --tags dns
ansible-playbook playbooks/site.yml --tags "common,ntp"

# Skip a capability
ansible-playbook playbooks/site.yml --skip-tags mde

# Verbose output for debugging
ansible-playbook playbooks/site.yml -l dev-web-01 -vv
```

---

## provision.yml

First-boot playbook. Used immediately after Terraform creates machines. It:

1. Waits for SSH connectivity (up to 5 minutes) before proceeding
2. Gathers facts once connectivity is confirmed
3. Imports `linux.yml` and `windows.yml`

Designed to be called from GitHub Actions via `workflow_call` from your Terraform repository, or run manually.

```bash
# Manual run against newly created hosts using a Terraform-derived inventory
terraform -chdir=../infra output -json > /tmp/tf_out.json
python3 scripts/tf_inventory.py /tmp/tf_out.json > inventories/production/tf_hosts.yml

ansible-playbook playbooks/provision.yml \
  -i inventories/production/hosts.yml \
  -i inventories/production/tf_hosts.yml

# Check mode before committing
ansible-playbook playbooks/provision.yml \
  -i inventories/production/hosts.yml \
  -i inventories/production/tf_hosts.yml \
  --check --diff

# Limit to specific new hosts only
ansible-playbook playbooks/provision.yml \
  -i inventories/production/hosts.yml \
  -i inventories/production/tf_hosts.yml \
  -l prod-web-03,prod-web-04
```

---

## linux.yml

Applies all Linux roles in the correct order:

```
pre_tasks/fetch_secrets.yml → common → dns_config → linux_hardening → datadog → tenable_agent → ms_defender
```

This playbook is imported by `site.yml`. You can also run it directly to apply all Linux roles without touching Windows hosts.

```bash
ansible-playbook playbooks/linux.yml

# Check only
ansible-playbook playbooks/linux.yml --check --diff

# Single role via tag
ansible-playbook playbooks/linux.yml --tags hardening

# Single host, verbose
ansible-playbook playbooks/linux.yml -l dev-web-01 -vv
```

---

## windows.yml

Applies all Windows roles:

```
pre_tasks/fetch_secrets.yml → dns_config → datadog → tenable_agent → ms_defender
```

```bash
ansible-playbook playbooks/windows.yml \
  -i inventories/production/hosts.yml

# Windows DNS config only
ansible-playbook playbooks/windows.yml \
  -i inventories/production/hosts.yml \
  --tags dns
```

---

## hardening.yml

Linux hardening only. Uses `--tags` to narrow scope without running other roles.

```bash
# Full hardening run with diff
ansible-playbook playbooks/hardening.yml --check --diff

# SSH config only (safe to run frequently)
ansible-playbook playbooks/hardening.yml --tags ssh

# Firewall rules only — open port 443 on webservers after updating group_vars
ansible-playbook playbooks/hardening.yml --tags firewall -l webservers

# Audit rules only
ansible-playbook playbooks/hardening.yml --tags auditd

# Password policy only
ansible-playbook playbooks/hardening.yml --tags password

# Service disable list only
ansible-playbook playbooks/hardening.yml --tags services

# Production hardening — always check first
ansible-playbook playbooks/hardening.yml \
  -i inventories/production/hosts.yml \
  --check --diff
```

---

## monitoring.yml

Installs or updates the Datadog agent on all hosts. Use this when:
- Rolling out a new Datadog agent version
- Changing agent configuration (tags, checks, APM)
- Adding Datadog to a new host group

```bash
# Update Datadog on all hosts
ansible-playbook playbooks/monitoring.yml

# Production, check first
ansible-playbook playbooks/monitoring.yml \
  -i inventories/production/hosts.yml \
  --check --diff

# Apply to a single group
ansible-playbook playbooks/monitoring.yml -l dbservers

# Verify agent is running after the play
ansible all -m command -a "systemctl status datadog-agent"
```

---

## security_agents.yml

Installs or updates Tenable (Nessus) agent and Microsoft Defender for Endpoint. Use `--tags` to target one agent at a time.

```bash
# Both agents on all hosts
ansible-playbook playbooks/security_agents.yml

# Tenable only
ansible-playbook playbooks/security_agents.yml --tags tenable

# MDE only
ansible-playbook playbooks/security_agents.yml --tags mde

# Production, single agent, check first
ansible-playbook playbooks/security_agents.yml \
  -i inventories/production/hosts.yml \
  --tags tenable \
  --check --diff

# Verify Tenable link status after run
ansible linux -m command -a "/opt/nessus_agent/sbin/nessuscli agent status"
```

---

## Common patterns

### Check mode on a production host

Always run `--check --diff` before applying changes to production. This shows exactly what would change without making any modifications.

```bash
ansible-playbook playbooks/site.yml \
  -i inventories/production/hosts.yml \
  -l prod-web-01 \
  --check --diff
```

### Targeted remediation

If a drift detection run shows that a specific control is out of compliance, use tags to remediate just that control:

```bash
# Re-apply SSH config only
ansible-playbook playbooks/hardening.yml \
  -i inventories/production/hosts.yml \
  --tags ssh \
  -l prod-web-01

# Re-sync Datadog config only
ansible-playbook playbooks/monitoring.yml \
  -i inventories/production/hosts.yml \
  -l prod-web-01
```

### Force a specific secrets backend

```bash
# Override the default backend for a run
ansible-playbook playbooks/site.yml \
  --extra-vars "secrets_backend=hashicorp_vault hcv_addr=https://vault.internal.acme.com"

# Test with local Vault dev server
export VAULT_TOKEN=root VAULT_ADDR=http://127.0.0.1:8200
ansible-playbook playbooks/site.yml \
  --extra-vars "secrets_backend=hashicorp_vault hcv_addr=http://127.0.0.1:8200"
```

### Syntax check all playbooks

```bash
for pb in playbooks/*.yml; do
  echo "Checking $pb..."
  ansible-playbook "$pb" --syntax-check -i inventories/development/hosts.yml
done
```

### List all tasks without running them

```bash
ansible-playbook playbooks/site.yml --list-tasks
ansible-playbook playbooks/site.yml --list-tasks --tags hardening
```

### List all hosts that would be affected

```bash
ansible-playbook playbooks/site.yml --list-hosts
ansible-playbook playbooks/site.yml -i inventories/production/hosts.yml --list-hosts
```
