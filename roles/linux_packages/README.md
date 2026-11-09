# linux_packages

Base administration and troubleshooting packages, one list per OS family.

## What it does

- Loads `vars/<os_family>.yml` (RedHat, Debian or Suse), which maps the same tools to each family's package names, for example `bind-utils` on RHEL and SUSE and `dnsutils` on Ubuntu.
- Installs that list plus any site-specific extras with the generic `package` module.

## Why it works this way

Distribution differences live in data files instead of `when:` conditions in the tasks, so adding a distribution means adding one file. `telnet` is left out on purpose: `nc` covers port checks without shipping a cleartext client.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_packages_extra` | list/str | `[]` | Additional packages installed on every host, on top of the OS family list. |

Internal: `linux_packages_base` is defined in `vars/RedHat.yml`, `vars/Debian.yml` and `vars/Suse.yml`. It is internal to the role; override `linux_packages_extra` instead.

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_packages
      vars:
        linux_packages_extra: [htop, jq]
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
