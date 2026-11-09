# linux_access

Accounts, SSH keys, sudo rules and password ageing.

## What it does

- Sets password ageing defaults in `/etc/login.defs`.
- Creates the admin group and the human administrator accounts, installs their SSH keys and applies password ageing to each one.
- Creates the automation account with a locked password, so it can only be reached by SSH key.
- Writes two sudoers drop-ins, each checked with `visudo` before it is installed: admins use sudo with their own password, and only the automation account gets passwordless sudo.
- On SUSE, adds `!targetpw` for the admin group, because SUSE otherwise asks for root's password.
- Optionally sets a vaulted break-glass root password that ages like any other.

## Why it works this way

Passwordless sudo is limited to an account nobody can log in to interactively. Every interactive account, root included, has an expiring password, which is what PCI DSS 8.3.9 and ISO 27001 A.5.17 auditors check.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_access_admin_group` | str | `opsadmins` | Group whose members may use sudo with their own password. |
| `linux_access_admins` | list/dict | `[]` | Human administrators. Each item: `name` (required), `ssh_key` (required while present), `comment`, `password_hash` (SHA-512, from Vault, set only when the account is created), `state` (`present` or `absent`; absent also removes the home directory). |
| `linux_access_automation_user` | str | `ansible` | Account Ansible connects as after bootstrap. |
| `linux_access_automation_key` | str | `""` | Public key for the automation account. Empty skips creating the account. |
| `linux_access_pass_max_days` | int | `90` | Maximum password age in days. |
| `linux_access_pass_min_days` | int | `1` | Minimum days between password changes. |
| `linux_access_pass_warn_age` | int | `7` | Days of warning before a password expires. |
| `linux_access_root_password_hash` | str | `""` | SHA-512 hash for root's break-glass password, from Vault. Empty leaves root untouched. |

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_access
      vars:
        linux_access_automation_key: "{{ lookup('file', '~/.ssh/ansible_automation.pub') }}"
        linux_access_admins:
          - name: jdoe
            ssh_key: "ssh-ed25519 AAAA... jdoe@laptop"
            password_hash: "{{ vault_linux_admin_password_hashes['jdoe'] }}"
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
