# linux_timesync

chrony as the only time service, with a check that it has a usable source.

## What it does

- Installs chrony and writes its configuration from a template.
- Stops and disables `ntpd`, `ntp` and `systemd-timesyncd` if they exist, so two daemons never adjust the clock.
- Waits until `chronyc sources` shows a selected or combined source, retrying for one minute.

## Why it works this way

One daemon and one config format on every distribution. Logs, TLS certificates and Kerberos all break on clock drift, so the role proves synchronisation instead of assuming it.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_timesync_servers` | list/str | `0.pool.ntp.org, 1.pool.ntp.org, 2.pool.ntp.org` | NTP servers, each written as `server <name> iburst`. On AWS use `169.254.169.123`. |

Internal: `linux_timesync_os` in `vars/main.yml` maps each OS family to its chrony service name, config path and drift file. It is internal to the role.

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_timesync
      vars:
        linux_timesync_servers: [ntp1.example.net, ntp2.example.net]
```

In this repository the values come from `inventory/<env>/group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
