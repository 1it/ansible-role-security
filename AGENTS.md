# AGENTS.md — `ansible-role-security`

## Role Conventions

- **Ansible target:** `>= 2.12` (tested through `ansible-core` 2.22).
- **FQCNs only:** Always use `ansible.builtin.*`, `ansible.posix.*`, or `community.general.*`. Stick to `ansible.builtin.systemd` (not `systemd_service`) and `ansible.builtin.dnf` on EL 8+.
- **Facts access:** Always use `ansible_facts['...']`; never use legacy top-level `ansible_*` variables.
- **Native Jinja2 types:** Never combine `{% set %}` with `{{ ... }}` inside `set_fact` for lists/dicts. Use single `{{ ... }}` expressions with task-level `vars:`.
- **Permissions:** Explicitly declare `owner`, `group`, and quoted `mode` (e.g., `"0644"`, `"0600"`, `"0755"`) on all `template`, `copy`, `file`, and `lineinfile` tasks.
- **Task naming & imports:** Prefix every task with `<Section> | <Description>`. Import task files statically in `tasks/main.yml` via `ansible.builtin.import_tasks` with granular `tags:`. Always keep `preflight.yml` tagged `always`.
- **Drop-in hygiene:** Prefer `*.d/` drop-ins (`/etc/sysctl.d/`, `/etc/security/limits.d/`, `/etc/ssh/sshd_config.d/`, `/etc/security/pwquality.conf.d/`, `/etc/apt/apt.conf.d/`, `/etc/systemd/system/*.d/`) instead of clobbering distro-owned configs.
- **Safe defaults:** Keep destructive/intrusive features opt-in (`false`): account removal, `/dev/shm` remounting, PAM restrictions, automatic updates, SFTP rsyslog, and `knockd`.
- **Sync metadata:** Whenever adding/changing variables, keep `defaults/main.yml`, `meta/argument_specs.yml`, and `README.md` in sync.

## Platform Guardrails

- **Debian 12/13 & Ubuntu 24.04/26.04:**
  - OpenSSH service is `ssh`; `/etc/ssh/sshd_config.d` mode is `0755`; `/run/sshd` must exist before running `sshd -t`.
  - OpenSSH <=9.5 (Debian 12, EL 8) rejects duplicate `Subsystem sftp` lines across drop-ins and `/etc/ssh/sshd_config`, so `Subsystem sftp` is managed in-place in `/etc/ssh/sshd_config` rather than duplicated in `00-security.conf`.
  - `/etc/sysctl.conf` is deprecated/absent on Debian 13+; always write to `/etc/sysctl.d/99-security.conf`.
  - `libpam-cracklib` is removed; use `libpam-pwquality` + `/etc/security/pwquality.conf.d/99-security.conf`.
  - `su` wheel group defaults to `sudo`.
  - Automatic updates use `unattended-upgrades` drop-ins (`20auto-upgrades`, `52unattended-upgrades-security`) without modifying `50unattended-upgrades` or APT sources.
- **EL 8/9/10 (Rocky, AlmaLinux, RHEL, CentOS Stream):**
  - OpenSSH service is `sshd`; `/etc/ssh/sshd_config.d` mode is `0700` and `/etc/ssh/sshd_config` mode is `0600`.
  - EL 8 uses `ChallengeResponseAuthentication`; EL 9+ uses `KbdInteractiveAuthentication` (handled in `templates/sshd_security.conf.j2`).
  - Leave `security_ssh_ciphers` / `kex_algorithms` / `macs` empty by default to preserve system-wide `crypto-policies`.
  - EL <=9 uses `/sbin/nologin` by default in `/etc/passwd`; EL 10 uses `/usr/sbin/nologin`.
  - `su` wheel group defaults to `wheel`.
  - Automatic updates use `dnf-automatic` (`/etc/dnf/automatic.conf` + `dnf-automatic.timer`).
- **Containers / Non-systemd hosts:**
  - `tasks/preflight.yml` checks `/run/systemd/system` and sets `_security_systemd_running`. Systemd tasks/handlers skip cleanly when systemd is not PID 1.
  - Never run unprivileged Docker tests with `--privileged` for sysctl; skip the `sysctl` tag (or set `security_sysctl_reload: false`) in unprivileged containers.

## Validation & Testing Commands

### 1. Syntax & Argument Spec Check
```bash
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT
ln -s "$PWD" "$TMPDIR/security"
cat > "$TMPDIR/playbook.yml" <<'EOF'
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: security
EOF
ANSIBLE_ROLES_PATH="$TMPDIR" ansible-playbook -i "localhost," --syntax-check "$TMPDIR/playbook.yml"
```

### 2. Template Rendering Check (`jinja2.StrictUndefined`)
Render all templates under `templates/` across Debian 12/13, Ubuntu 24.04/26.04, and EL 8/9/10 (`amd64` + `arm64`) with default and feature-enabled variable matrices using Python `jinja2.Environment(undefined=jinja2.StrictUndefined)`.

### 3. Docker Convergence & Idempotency Matrix
Test all official images (`debian:12`, `debian:13`, `ubuntu:24.04`, `ubuntu:26.04`, `rockylinux/rockylinux:9`, `rockylinux/rockylinux:10`) with container prefix `ansible-security-test-*` and cleanup via `trap ... EXIT`:
- Install `python3` in each test container, run the role twice via `community.docker.docker` connection plugin with `--skip-tags sysctl` (or `security_sysctl_reload=false` to also verify `/etc/sysctl.d/99-security.conf` generation without modifying host kernel params).
- Verify run 1 converges cleanly (`failed=0`) and run 2 achieves strict idempotency (`changed=0`, `failed=0`).
- Also run systemd-booted container tests (Debian 12/13, Ubuntu 24.04, Rocky 9) to verify `sshd` reload handler, `systemd` units, `knockd` drop-in, and automatic update timers.
