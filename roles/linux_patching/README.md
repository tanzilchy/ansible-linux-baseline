# linux_patching

Full OS update with a pre-check, a conditional reboot and a kernel report.

## What it does

- Checks that the package mirror for this OS release is reachable before changing anything.
- Applies all updates with dnf, apt (`dist-upgrade`) or zypper.
- Asks the OS whether a reboot is needed (`needs-restarting -r`, `/var/run/reboot-required`, or `zypper needs-rebooting`) and reboots only then.
- Reports the kernel version before and after.

## Why it works this way

Rebooting only when the OS asks for it avoids needless downtime. The playbook around this role (`playbooks/patch.yml`) adds batching and `max_fail_percentage: 0`, so one failed host stops the rollout.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_patching_repo_targets` | dict | `{}` | Mirror to check per `<os_family>-<major>` key (for example `RedHat-9`), with a `default` fallback. Each value is `{host, port}`. Empty skips the check. |
| `linux_patching_allow_reboot` | bool | `true` | Allow the role to reboot when the OS reports one is needed. |
| `linux_patching_reboot_timeout` | int | `900` | Seconds to wait for the host to come back. |

Playbook variable: `linux_patching_batch` is read by `playbooks/patch.yml`, not by this role. It sets `serial`, which Ansible evaluates before inventory variables load, so pass it with `-e linux_patching_batch=1`.

## Task files

`tasks/main.yml` only sets the order. Each concern has its own file and tag, so one part can run alone, for example `--tags patching_precheck`.

| File | Tag | Does |
|---|---|---|
| `tasks/precheck.yml` | `patching_precheck` | Mirror reachability, kernel before |
| `tasks/update_redhat.yml` | `patching_update` | dnf update and needs-restarting |
| `tasks/update_debian.yml` | `patching_update` | apt dist-upgrade and reboot-required flag |
| `tasks/update_suse.yml` | `patching_update` | zypper update and needs-rebooting |
| `tasks/reboot.yml` | `patching_reboot` | Reboot only if needed and allowed |
| `tasks/report.yml` | `patching_report` | Kernel before and after |

Only one `update_<family>.yml` runs: `main.yml` includes it by name from `ansible_facts.os_family`, so adding a distribution family means adding one file.

## Example

```yaml
- hosts: linux
  become: true
  serial: 1
  max_fail_percentage: 0
  roles:
    - role: linux_patching
      vars:
        linux_patching_allow_reboot: false
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
