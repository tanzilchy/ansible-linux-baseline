# linux_hardening

SSH daemon baseline and idle session timeouts.

## What it does

- Checks whether `sshd_config` includes `sshd_config.d/` and writes either a drop-in (`00-baseline.conf`) or the main file accordingly. This covers RHEL 8 (no Include line), RHEL 9 and Ubuntu (Include line) and newer SUSE (no `/etc/ssh/sshd_config` at all).
- Removes the installer's `01-permitrootlogin.conf` override.
- Validates the full sshd configuration with `sshd -t` before reloading, so a bad setting fails the run instead of locking everyone out.
- Sets a read-only `TMOUT` for every shell, and `StopIdleSessionSec` in logind on systemd 252 and newer.

## Why it works this way

Config layout is detected, not assumed. `StopIdleSessionSec` is only written where systemd understands it, because older versions ignore it silently, and a control that does nothing is worse than a missing one.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_hardening_permit_root_login` | str | `no` | `PermitRootLogin`. Keep `no` once bootstrap has created key-based accounts. |
| `linux_hardening_password_authentication` | str | `no` | `PasswordAuthentication`. |
| `linux_hardening_max_auth_tries` | int | `4` | `MaxAuthTries`. |
| `linux_hardening_login_grace_time` | int | `60` | `LoginGraceTime` in seconds. |
| `linux_hardening_idle_timeout` | int | `300` | Seconds before idle shells and logind sessions are closed. |
| `linux_hardening_sshd_settings` | dict | built from the variables above, plus X11Forwarding no, PermitEmptyPasswords no, ClientAliveInterval 300, ClientAliveCountMax 3 | Full set of sshd keywords written to the server. Override the whole dictionary to add keywords. |

## Task files

`tasks/main.yml` only sets the order. Each concern has its own file and tag, so one part can run alone, for example `--tags hardening_sshd`.

| File | Tag | Does |
|---|---|---|
| `tasks/sshd.yml` | `hardening_sshd` | Detect config layout, write settings, remove installer override |
| `tasks/session_timeout.yml` | `hardening_timeout` | TMOUT and logind StopIdleSessionSec |

## Ordering

Run after `linux_access`. Turning off root and password logins before key-based accounts exist locks Ansible out.

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_hardening
      vars:
        linux_hardening_idle_timeout: 600
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
