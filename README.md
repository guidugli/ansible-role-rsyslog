[![CI](https://github.com/guidugli/ansible-role-rsyslog/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-rsyslog/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/release/guidugli/ansible-role-rsyslog?sort=semver)](https://github.com/guidugli/ansible-role-rsyslog/releases)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.rsyslog-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/rsyslog/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

# Ansible Role: rsyslog

Installs and configures rsyslog on supported Linux hosts. The role manages the main rsyslog configuration, local logging rules, optional TCP log-host reception, queued forwarding to a remote rsyslog server, optional TLS forwarding, and an optional logrotate policy. Opt-in CIS client profiles cover the role-controlled portions of the Ubuntu Linux 24.04 LTS rsyslog Level 1 recommendations and Level 2 TLS forwarding recommendations.

## Requirements

- Ansible Core 2.14 or newer, as declared by the role metadata.
- A supported Debian, Ubuntu, or Fedora target with package access.
- External privilege escalation when packages, files under `/etc`, files under `/var`, or system services are managed.
- Collections listed in `requirements.yml` for Molecule/Podman testing.
- A central log server value when `rsyslog_cis_level1_enabled` or `rsyslog_cis_level2_enabled` is `true`.
- A readable CA certificate file when TLS forwarding is enabled. The default is `/etc/ssl/certs/ca-certificates.crt`.

Applying this role is an explicit decision to manage rsyslog. The role does not uninstall or disable `systemd-journald`.

## Variables

Default variables are defined in `defaults/main.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `rsyslog_cis_level1_enabled` | boolean | `false` | Enables the CIS Ubuntu Level 1 client profile. Requires forwarding, prevents receiver mode, and manages logrotate. |
| `rsyslog_cis_level2_enabled` | boolean | `false` | Enables the CIS Ubuntu Level 2 TLS forwarding profile. Implies TLS forwarding configuration. |
| `rsyslog_port` | integer | `514` | TCP port used for remote reception or forwarding. Use a TLS syslog port such as `6514` when required by your environment. |
| `rsyslog_enable_log_host` | boolean | `false` | Enables this host as a TCP log receiver. Must be `false` for CIS client profiles. |
| `rsyslog_log_server` | string | `""` | Remote server hostname or address. Required by CIS client profiles. |
| `rsyslog_retry_count` | integer | `-1` | Remote forwarding retry count; `-1` means unlimited retries. |
| `rsyslog_max_disk_space` | string | `"1g"` | Maximum disk space used by the forwarding queue. |
| `rsyslog_save_on_shutdown` | string | `"on"` | Saves queued messages during shutdown. Values: `on`, `off`. |
| `rsyslog_queue_filename` | string | `"fwdRule1"` | Spool-file prefix for the forwarding queue. Only filename-safe characters are accepted. |
| `rsyslog_tls_enabled` | boolean | `false` | Enables TLS forwarding settings outside the CIS Level 2 profile. |
| `rsyslog_tls_ca_file` | string | `/etc/ssl/certs/ca-certificates.crt` | CA certificate file used by `DefaultNetstreamDriverCAFile`. |
| `rsyslog_tls_stream_driver` | string | `gtls` | TLS stream driver used for encrypted forwarding. |
| `rsyslog_tls_stream_driver_mode` | string | `"1"` | TLS stream driver mode. |
| `rsyslog_tls_stream_driver_auth_mode` | string | `anon` | TLS stream driver authentication mode. |
| `rsyslog_manage_logrotate` | boolean | `false` | Manages the distribution-specific rsyslog logrotate policy; implied by CIS client profiles. |
| `rsyslog_logrotate_user` | string | `root` | User used in the logrotate `su` directive. |
| `rsyslog_logrotate_group` | string | `auto` | Group used in the logrotate `su` directive. `auto` resolves to `adm` on Debian-family systems and `root` on Red Hat-family systems. |
| `rsyslog_logrotate_frequency` | string | `weekly` | Rotation frequency: `daily`, `weekly`, or `monthly`. |
| `rsyslog_logrotate_rotate` | integer | `13` | Number of rotated files retained. Adapt to organizational policy. |
| `rsyslog_logrotate_maxage` | integer | `90` | Maximum age in days for rotated logs. Adapt to organizational policy. |
| `rsyslog_logrotate_compress` | boolean | `true` | Compresses rotated logs. |
| `rsyslog_logrotate_delaycompress` | boolean | `true` | Delays compression until the next rotation cycle. |
| `rsyslog_logrotate_paths` | list | See `defaults/main.yml` | Log files covered by the managed logrotate policy. |

Internal variables are defined in `vars/main.yml` and normally should not be overridden.

| Variable | Description |
| --- | --- |
| `rsyslog_conf_path` | Main rsyslog configuration path. |
| `rsyslogd_path` | Rsyslog drop-in configuration directory. |
| `rsyslog_package` | Package list selected for the current distribution. |
| `rsyslog_tls_package` | TLS support package list selected for the current distribution. |
| `rsyslog_tls_effective_enabled` | Effective TLS state derived from `rsyslog_tls_enabled` and `rsyslog_cis_level2_enabled`. |
| `rsyslog_file` | Distribution-family-specific local logging-rules template. |
| `rsyslog_logrotate_path` | Distribution-family-specific rsyslog logrotate configuration path. |
| `rsyslog_logrotate_effective_group` | Resolved group for the logrotate `su` directive. |

The bundled local logging rules and log-retention defaults are implementation defaults, not a substitute for an organization-approved logging and retention policy.

## Example Playbook

The role never enables privilege escalation itself. The calling playbook must provide it when required.

```yaml
---
- name: Configure an Ubuntu rsyslog client with CIS Level 2 TLS forwarding
  hosts: ubuntu_clients
  become: true
  roles:
    - role: guidugli.rsyslog
      vars:
        rsyslog_cis_level1_enabled: true
        rsyslog_cis_level2_enabled: true
        rsyslog_log_server: logs.example.com
        rsyslog_port: 6514
        rsyslog_tls_ca_file: /etc/ssl/certs/ca-certificates.crt
```

A designated central log host is a separate operating mode and intentionally does not satisfy the CIS client recommendation that clients must not receive remote logs:

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

The repository uses shared Molecule convergence and verification logic with default and systemd-capable scenarios. The shared convergence enables the CIS Level 1 client profile on all platforms and exercises TLS forwarding on Debian-family platforms. Verification checks packages, file ownership, file modes, `$FileCreateMode`, forwarding semantics, TLS forwarding semantics, client listener absence, logrotate policy, rsyslog syntax, logrotate syntax, and service state under systemd.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
molecule test -s default
molecule test -s systemd
```

## Execution Notes

- **Privilege model:** the role contains no `become`, `become_user`, or `become_method`. Package installation and management of `/etc`, `/var`, and system services normally require `become: true` in the caller.
- **Container behavior:** package and configuration behavior remains testable in non-systemd containers. Service operations and handlers are guarded by `ansible_facts['service_mgr'] == 'systemd'`.
- **Systemd behavior:** systemd targets are explicitly unmasked, enabled, and started. The systemd Molecule scenario verifies running state through service facts and enabled state with `systemctl is-enabled`.
- **Idempotency:** role-owned forwarding and listener configuration is removed when the corresponding feature is disabled. Forwarding and logrotate configuration are rendered from templates for deterministic output.
- **CIS scope:** enabling `rsyslog_cis_level1_enabled` implements role-controlled portions of Ubuntu Level 1 rsyslog configuration. Enabling `rsyslog_cis_level2_enabled` adds role-controlled TLS forwarding controls. The organization must still approve logging destinations, certificate trust, and retention values and validate that logs arrive at the central service.
- **Transport:** forwarding uses TCP without TLS unless `rsyslog_tls_enabled` or `rsyslog_cis_level2_enabled` is true. TLS forwarding depends on certificate trust and correct remote receiver configuration.
- **Package-managed logrotate file:** this role manages the distribution-specific rsyslog logrotate path. Review package upgrade behavior in environments where vendor files are expected to remain untouched.
