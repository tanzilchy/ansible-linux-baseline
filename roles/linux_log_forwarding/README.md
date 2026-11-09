# linux_log_forwarding

rsyslog running, and a reachability check for the central log collector.

## What it does

- Installs, enables and starts rsyslog.
- Opens a TCP connection to the collector and reports whether it is reachable.
- Fails the run when the collector is required and unreachable.

## Why it works this way

Audit requirements depend on logs leaving the host. A TCP check cannot prove delivery over UDP 514, so TCP or RELP is the better transport for audit logs.

## Supported systems

RHEL 8 to 10 and rebuilds (Rocky, AlmaLinux, CentOS Stream 9 and 10), Ubuntu 22.04 and 24.04, Debian 12, SLES 15 and openSUSE Leap 15. `linux_preflight` refuses anything else.

## Variables

Inputs are validated against `meta/argument_specs.yml` before the first task runs. Defaults live in `defaults/main.yml`.

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_log_forwarding_target` | dict | `{}` | Collector as `{host: ..., port: ...}`. Empty skips the check. |
| `linux_log_forwarding_required` | bool | `true` | Fail the run if the collector is unreachable. The example inventory sets `false` for uat and `true` for prod. |

## Example

```yaml
- hosts: linux
  become: true
  roles:
    - role: linux_log_forwarding
      vars:
        linux_log_forwarding_target: {host: siem.example.net, port: 514}
```

In this repository the values come from `inventory/<env>/group_vars/` rather than play `vars`. See the top-level README for the layout.

## Dependencies

None. Collections used are listed in the top-level `requirements.yml`.
