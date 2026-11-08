# ansible-linux-baseline

The baseline I apply to a new RHEL or Ubuntu server before it goes into a regulated environment: who can log in and how, password ageing, SSH settings, idle timeouts, time sync, log forwarding, and rolling OS patching.

Supported targets: RHEL 8 and 9 (and rebuilds such as Rocky and AlmaLinux), Ubuntu 22.04 and 24.04.

## Playbooks

| Playbook | When | What happens |
|---|---|---|
| `playbooks/bootstrap.yml` | Once per new VM, as the install-time user with `--ask-pass` | Creates the key-only automation account and the admin accounts |
| `playbooks/site.yml` | Any time; preview with `--check --diff` | Access, packages, chrony, SSH and session hardening, log collector check |
| `playbooks/patch.yml` | Patch windows | Mirror reachability check, full update, reboot only if the OS needs it, kernel before and after |

## Design decisions

- **Order prevents lockouts.** Bootstrap creates key-based accounts first. Only then does `site.yml` set `PermitRootLogin no` and `PasswordAuthentication no`, and the sshd config is validated with `sshd -t` before the service reloads.
- **People and automation are separate accounts.** Admins log in with their own key and type their own password for sudo. Passwordless sudo is limited to the automation account, which has a locked password and can only be reached by key.
- **Password ageing is real.** 90 day maximum, 1 day minimum and 7 day warning in `login.defs` and on each account, including root's vaulted break-glass password. Nothing is set to never expire.
- **One time daemon.** chrony on every distro. `ntpd` and `systemd-timesyncd` are stopped so two services never fight over the clock, and the run waits until chrony has a usable source.
- **Controls only where they work.** `StopIdleSessionSec` is only written when systemd is 252 or newer, because older versions ignore it silently. `TMOUT` covers every shell.
- **Version checks are numeric.** `distribution_major_version | int`, never string comparison, which would sort "10" before "8".
- **Patching is rolling.** Batches of 25% by default with `max_fail_percentage: 0`, so one failed host stops the rollout. Reboots only happen when `needs-restarting -r` or `/var/run/reboot-required` says so.
- **Idempotent by design.** A second run of `site.yml` should report no changes; that is the first thing to check on a new lab VM.

## Quick start

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install ansible-core
ansible-galaxy collection install -r requirements.yml

cp inventory/example/group_vars/linux/vault.yml.example inventory/example/group_vars/linux/vault.yml
ansible-vault encrypt inventory/example/group_vars/linux/vault.yml

ansible-playbook playbooks/bootstrap.yml -l app-uat-01.example.net -u root --ask-pass --ask-vault-pass
ansible-playbook playbooks/site.yml -l uat --check --diff --ask-vault-pass
ansible-playbook playbooks/site.yml -l uat --ask-vault-pass
ansible-playbook playbooks/patch.yml -l uat -e linux_patching_batch=1 --ask-vault-pass
```

RHEL 8 ships Python 3.6 as its platform Python, which recent ansible-core releases no longer support on targets. Install `python3.11` from AppStream and set `ansible_python_interpreter`, or pin an older ansible-core.

## Testing

CI runs `yamllint`, `ansible-lint` (production profile) and a syntax check on every push.

## Roles

| Role | Purpose |
|---|---|
| `linux_access` | Admin group, admin and automation accounts, SSH keys, sudoers drop-ins, password ageing, root break-glass password |
| `linux_packages` | Base admin and troubleshooting tools, one list per OS family |
| `linux_timesync` | chrony install, config, competing services off, source check |
| `linux_hardening` | sshd baseline (drop-in or `sshd_config`), idle shell and logind timeouts |
| `linux_log_forwarding` | rsyslog running, collector reachability |
| `linux_patching` | Mirror check, updates, conditional reboot, kernel report |
