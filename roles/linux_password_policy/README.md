# linux_password_policy

Password quality, password history and account lockout through PAM.

## What it does

- Writes `/etc/security/pwquality.conf`, `faillock.conf` and `pwhistory.conf`. These files are the same on every OS family, so the values live in one place.
- Switches the PAM modules on with each family's own tool: `authselect` features on RHEL, `pam-auth-update` profiles on Ubuntu and Debian, `pam-config` on SUSE.
- On RHEL 8 (PAM 1.3, which does not read `pwhistory.conf`), sets `remember=` on the `pam_pwhistory` line as well.
- Proves the quality rules with `pwscore`: a weak password must be rejected and a compliant one accepted, or the run fails.

## Why it works this way

Hand edits to `/etc/pam.d` are undone the next time `authselect` or `pam-auth-update` runs, so the role uses each family's PAM manager instead. Root is exempt from lockout by default so the console break-glass login always works. The default minimum length is 12, as PCI DSS v4 8.3.6 requires.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_password_policy_minlen` | int | `12` | Minimum password length. |
| `linux_password_policy_require_lower` | bool | `true` | Require at least one lowercase letter. |
| `linux_password_policy_require_upper` | bool | `true` | Require at least one uppercase letter. |
| `linux_password_policy_require_digit` | bool | `true` | Require at least one digit. |
| `linux_password_policy_require_special` | bool | `true` | Require at least one special character. |
| `linux_password_policy_reject_username` | bool | `true` | Reject passwords containing the username, forwards or reversed. |
| `linux_password_policy_enforce_for_root` | bool | `true` | Apply quality and history rules when root sets a password. |
| `linux_password_policy_remember` | int | `5` | Number of previous passwords that cannot be reused. |
| `linux_password_policy_lockout_deny` | int | `3` | Failed attempts before the account locks. |
| `linux_password_policy_lockout_fail_interval` | int | `300` | Window in seconds in which failures are counted. |
| `linux_password_policy_lockout_unlock_time` | int | `1800` | Seconds the account stays locked. |
| `linux_password_policy_lockout_root` | bool | `false` | Also lock root (`even_deny_root`). Off by default to keep the console break-glass login. |

Internal: `linux_password_policy_packages` in `vars/main.yml` maps each OS family to the packages that provide `pam_pwquality` and `pwscore`.

## Task files

`tasks/main.yml` only sets the order. Each concern has its own file and tag, so one part can run alone, for example `--tags password_policy_settings`.

| File | Tag | Does |
|---|---|---|
| `tasks/settings.yml` | `password_policy_settings` | pwquality.conf, faillock.conf, pwhistory.conf |
| `tasks/enable_redhat.yml` | `password_policy_enable` | authselect features with-faillock and with-pwhistory |
| `tasks/enable_debian.yml` | `password_policy_enable` | pam-auth-update profiles for faillock and pwhistory |
| `tasks/enable_suse.yml` | `password_policy_enable` | pam-config --pwquality --pwhistory --faillock (not yet tested on a live host) |
| `tasks/verify.yml` | `password_policy_verify` | pwscore: weak password rejected, compliant password accepted |

Only one `enable_<family>.yml` runs, chosen by `ansible_facts.os_family`.

## Ordering

Run after `linux_access`. Lockout counts failed password attempts only, so key-based logins, including the automation account, are never locked out.

## Testing lockout

Test lockout by hand while a second session is open, because a mistake in PAM can block every password login:

```bash
sudo useradd -m locktest && sudo passwd locktest      # a compliant password
for i in 1 2 3; do su - locktest -c true <<< "wrong"; done
sudo faillock --user locktest                         # shows 3 failures; the account is locked
sudo faillock --user locktest --reset && sudo userdel -r locktest
```

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_password_policy
      vars:
        linux_password_policy_minlen: 14
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.