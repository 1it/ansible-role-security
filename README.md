# ansible-role-security

Standalone Ansible role for production Linux security hardening across modern Debian, Ubuntu, and Enterprise Linux systems.

## Requirements

- **Ansible:** `>= 2.12` (tested up to `ansible-core` 2.22)
- **Collections:**
  - `ansible.builtin`
  - `ansible.posix` (required for optional `/dev/shm` mount hardening)
- **Supported Platforms (`amd64` and `arm64`):**
  - Debian 12 (Bookworm), Debian 13 (Trixie)
  - Ubuntu 24.04 (Noble), Ubuntu 26.04 (Resolute)
  - EL 8, 9, 10 (RHEL, Rocky Linux, AlmaLinux, CentOS Stream)

## Role Sections & Tags

| Tag | Default | Description |
| --- | --- | --- |
| `system` | Enabled | Root/SSH directory permissions, `nologin` shell on existing system users, core dump & PAM resource limits, opt-in user/group removal, opt-in `/dev/shm` hardening |
| `sysctl` | Enabled | Kernel & network hardening drop-in (`/etc/sysctl.d/99-security.conf`) |
| `ssh` | Enabled | OpenSSH drop-ins (`/etc/ssh/sshd_config.d/00-security.conf`, optional `99-security-match.conf`), config validation, optional SFTP rsyslog/logrotate |
| `pam` | Opt-in (`false`) | `libpwquality` drop-in (`/etc/security/pwquality.conf.d/99-security.conf`) and optional `su` `pam_wheel.so` restriction |
| `auto_updates` / `updates` | Opt-in (`false`) | `unattended-upgrades` (Debian/Ubuntu) or `dnf-automatic` (EL 8/9/10) |
| `knockd` | Opt-in (`false`) | `knockd` port-knocking daemon with systemd restart drop-in |

## Key Default Variables

All variables are defined and documented in [`defaults/main.yml`](defaults/main.yml) and validated by [`meta/argument_specs.yml`](meta/argument_specs.yml).

- **System & Limits:**
  - `security_nologin_users` / `security_nologin_extra_users` — existing system accounts enforced to `nologin` (non-existent users are skipped).
  - `security_remove_unused_users: false`, `security_remove_unused_groups: false` — opt-in legacy account removal.
  - `security_harden_dev_shm: false` — opt-in `/dev/shm` `nosuid,nodev,noexec`.
  - `security_disable_core_dumps: true` — manages `/etc/security/limits.d/99-security-coredump.conf`.
  - `security_limits: []` / `security_extra_limits: []` — optional `/etc/security/limits.d/99-security-limits.conf`.
- **Sysctl:**
  - `security_sysctl_enabled: true`, `security_sysctl_reload: true`, `security_sysctl_file: "/etc/sysctl.d/99-security.conf"`.
  - `security_sysctl_params` / `security_sysctl_extra_params: {}` — merged kernel parameters (set a key to `null` in `security_sysctl_extra_params` to omit a default).
- **OpenSSH:**
  - `security_ssh_enabled: true`, `security_ssh_port: 22`, `security_ssh_permit_root_login: "prohibit-password"`, `security_ssh_password_authentication: false`, `security_ssh_x11_forwarding: false`, `security_ssh_use_dns: false`.
  - `security_ssh_extra_config: {}` — additive or overriding `sshd_config` directives.
  - `security_ssh_match_blocks: []` — written to `/etc/ssh/sshd_config.d/99-security-match.conf`.
  - `security_ssh_sftp_rsyslog_enabled: false` — opt-in rsyslog + logrotate rule for `internal-sftp`.
- **PAM (Opt-in):**
  - `security_pam_enabled: false`, `security_pam_pwquality_params`, `security_pam_pwquality_extra_params: {}`.
  - `security_pam_su_restrict_wheel: false`, `security_pam_su_group: ""` (defaults to `sudo` on Debian/Ubuntu, `wheel` on EL).
- **Automatic Security Updates (Opt-in):**
  - `security_auto_updates_enabled: false`, `security_auto_updates_apply: true`, `security_auto_updates_reboot: false`.
  - `security_auto_updates_blacklist: []` / `security_auto_updates_extra_blacklist: []`.
- **Knockd (Opt-in):**
  - `security_knockd_enabled: false`, `security_knockd_sequence: ""` (required when enabled).

## Example Playbook

```yaml
---
- name: Harden Linux servers
  hosts: all
  become: true
  roles:
    - role: security
      vars:
        security_ssh_port: 22
        security_ssh_permit_root_login: "prohibit-password"
        security_ssh_password_authentication: false
        security_ssh_extra_config:
          MaxAuthTries: 3
        security_sysctl_extra_params:
          net.ipv4.conf.all.rp_filter: 2
          net.ipv4.conf.default.rp_filter: 2
        security_limits:
          - { domain: "*", type: "soft", item: "nofile", value: 65536 }
          - { domain: "*", type: "hard", item: "nofile", value: 65536 }
        security_pam_enabled: true
        security_auto_updates_enabled: true
```

Run only specific sections with tags:

```bash
ansible-playbook -i inventory.ini site.yml --tags ssh,sysctl
```

## License

Apache-2.0
