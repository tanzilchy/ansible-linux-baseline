# host_vars

Put a file here only when one host must differ from its groups, named after the host as it appears in `hosts.yml`:

```yaml
# host_vars/rhel-prod-01.example.net.yml
linux_patching_allow_reboot: false   # vendor appliance, rebooted by its owner
```

Keep these rare. A value that applies to more than one host belongs in a group.
