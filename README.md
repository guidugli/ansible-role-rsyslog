[![CI](https://github.com/guidugli/ansible-role-rsyslog/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-rsyslog/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/tag/guidugli/ansible-role-rsyslog?sort=semver)](https://github.com/guidugli/ansible-role-rsyslog/tags)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.rsyslog-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/rsyslog/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

# Ansible Role: rsyslog

Installs and configures rsyslog on supported Linux hosts. The role manages the main rsyslog configuration, local
logging rules, optional TCP log-host reception, queued forwarding to a remote rsyslog server, and an optional
logrotate policy. An opt-in client profile implements the role-controlled portions of the CIS Ubuntu Linux 24.04
LTS Level 1 rsyslog recommendations.

## Requirements

- Ansible Core 2.14 or newer, as declared by the role metadata.
- A supported Debian, Ubuntu, or Fedora target with package access.
- External privilege escalation when packages and files under `/etc` are managed.
- Collections listed in `requirements.yml` for Molecule/Podman testing.
- A central log server value when `rsyslog_cis_level1_enabled` is `true`.

Applying this role is an explicit decision to manage rsyslog. The role does not uninstall or disable
`systemd-journald`.

## Variables

Default variables are defined in `defaults/main.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `rsyslog_cis_level1_enabled` | boolean | `false` | Enables the CIS Ubuntu Level 1 client profile. Requires forwarding, prevents receiver mode, and manages logrotate. |
| `rsyslog_port` | integer | `514` | TCP port used for remote reception or forwarding. |
| `rsyslog_enable_log_host` | boolean | `false` | Enables this host as a TCP log receiver. Must be `false` for the CIS client profile. |
| `rsyslog_log_server` | string | `""` | Remote server hostname or address. Required by the CIS client profile. |
| `rsyslog_retry_count` | integer | `-1` | Remote forwarding retry count; `-1` means unlimited retries. |
| `rsyslog_max_disk_space` | string | `"1g"` | Maximum disk space used by the forwarding queue. |
| `rsyslog_save_on_shutdown` | string | `"on"` | Saves queued messages during shutdown. Values: `on`, `off`. |
| `rsyslog_queue_filename` | string | `"fwdRule1"` | Spool-file prefix for the forwarding queue. |
| `rsyslog_manage_logrotate` | boolean | `false` | Manages the distribution-specific rsyslog logrotate policy; implied by the CIS client profile. |
| `rsyslog_logrotate_frequency` | string | `weekly` | Rotation frequency: `daily`, `weekly`, or `monthly`. |
| `rsyslog_logrotate_rotate` | integer | `13` | Number of rotated files retained. Adapt to organizational policy. |
| `rsyslog_logrotate_maxage` | integer | `90` | Maximum age in days for rotated logs. Adapt to organizational policy. |
| `rsyslog_logrotate_compress` | boolean | `true` | Compresses rotated logs. |
| `rsyslog_logrotate_delaycompress` | boolean | `true` | Delays compression until the next rotation cycle. |
| `rsyslog_logrotate_paths` | list | See `defaults/main.yml` | Log files covered by the managed logrotate policy. |

Internal variables are defined in `vars/main.yml` and normally should not be overridden:

| Variable | Description |
| --- | --- |
| `rsyslog_conf_path` | Main rsyslog configuration path. |
| `rsyslogd_path` | Rsyslog drop-in configuration directory. |
| `rsyslog_package` | Package list selected for the current distribution. |
| `rsyslog_file` | Distribution-family-specific local logging-rules template. |

The bundled local logging rules and log-retention defaults are implementation defaults, not a substitute for an
organization-approved logging and retention policy.

## Example Playbook

The role never enables privilege escalation itself. The calling playbook must provide it when required.

```yaml
---
- name: Configure an Ubuntu rsyslog client with the CIS Level 1 profile
  hosts: ubuntu_clients
  become: true
  roles:
    - role: guidugli.rsyslog
      vars:
        rsyslog_cis_level1_enabled: true
        rsyslog_log_server: logs.example.com
        rsyslog_port: 514
        rsyslog_retry_count: -1
        rsyslog_logrotate_frequency: weekly
        rsyslog_logrotate_rotate: 13
        rsyslog_logrotate_maxage: 90
```

A designated central log host is a separate operating mode and intentionally does not satisfy the CIS client
recommendation that clients must not receive remote logs:

```yaml
---
- name: Configure a designated rsyslog receiver
  hosts: log_hosts
  become: true
  roles:
    - role: guidugli.rsyslog
      vars:
        rsyslog_enable_log_host: true
        rsyslog_port: 514
```

## Molecule Testing

The repository uses shared Molecule convergence and verification logic with default and systemd-capable scenarios.
The shared convergence enables the CIS Level 1 client profile. Verification checks packages, effective file modes,
`$FileCreateMode`, forwarding semantics, client listener absence, logrotate policy, rsyslog syntax, logrotate syntax,
and service state under systemd.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
molecule test -s default
molecule test -s systemd
```

## Execution Notes

- **Privilege model:** the role contains no `become`, `become_user`, or `become_method`. Package installation and
  management of `/etc`, `/var`, and system services normally require `become: true` in the caller.
- **Container behavior:** package and configuration behavior remains testable in non-systemd containers. Service
  operations and handlers are guarded by `ansible_facts['service_mgr'] == 'systemd'`.
- **Systemd behavior:** systemd targets are explicitly unmasked, enabled, and started. The systemd Molecule scenario
  verifies that `rsyslog.service` is enabled and running.
- **Idempotency:** role-owned forwarding and listener configuration is removed when the corresponding feature is
  disabled. Templates and package/service modules are deterministic.
- **CIS scope:** enabling `rsyslog_cis_level1_enabled` implements role-controlled portions of Ubuntu Level 1 rsyslog
  configuration. The organization must still approve logging destinations and retention values and validate that
  logs arrive at the central service. The role does not claim Level 2 TLS coverage.
- **Transport:** forwarding uses TCP without TLS. Use a separately designed TLS implementation when confidentiality
  and peer authentication are required.
