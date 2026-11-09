# linux_preflight

Refuse unsupported distributions and releases before any change is made.

## What it does

- Reads the gathered facts and compares the distribution and major version with a supported-version table.
- Fails the play for that host with a message listing what is supported.
- Checks that the host's real OS family matches the distribution group it is in, so a misplaced host is caught before it is configured with the wrong user or playbook.
- Runs first in every playbook, so an end-of-life host is never half-configured.

## Why it works this way

CentOS 7, RHEL 7 and SLES 12 are end of life and ship Python versions that current ansible-core cannot manage. Failing early with a clear message is better than failing halfway through the access role.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_preflight_min_major` | dict | see `defaults/main.yml` | Oldest supported major release per distribution, keyed by `ansible_facts.distribution`. Distributions not listed are refused. |
| `linux_preflight_expected_os_family` | str | `""` | OS family the inventory group claims for this host. Set in `group_vars/rhel.yml`, `ubuntu.yml` and `suse.yml`. A mismatch with the facts fails the host. Empty skips the check. |

## Example

```yaml
- hosts: linux
  roles:
    - linux_preflight
```

In this repository the values come from the inventory's `group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
