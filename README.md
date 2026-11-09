# ansible-linux-baseline

![lint](https://github.com/tanzilchy/ansible-linux-baseline/actions/workflows/lint.yml/badge.svg)

The baseline I apply to a new Linux server before it goes into a regulated environment: who can log in and how, password ageing, SSH settings, idle timeouts, time sync, log forwarding, and rolling OS patching.

## Supported systems

| Family | Distributions | Package tool | Reboot check |
|---|---|---|---|
| Red Hat | RHEL 8 to 10, CentOS Stream 9 and 10, Rocky, AlmaLinux | dnf | `needs-restarting -r` |
| Debian | Ubuntu 22.04 and 24.04, Debian 12 | apt | `/var/run/reboot-required` |
| SUSE | SLES 15, openSUSE Leap 15 | zypper | `zypper needs-rebooting` |

`linux_preflight` refuses anything older before a single change is made. CentOS 7, RHEL 7 and SLES 12 are end of life and ship Python versions that current ansible-core cannot manage.

## Repository layout

```
ansible.cfg                    defaults that are safe for anyone who clones the repo
requirements.yml               collections: ansible.posix, community.general
playbooks/
  bootstrap.yml                first contact: create the automation and admin accounts
  site.yml                     the full baseline: imports the three OS playbooks
  os/setup-rhel-vm.yml                  baseline for group rhel
  os/setup-ubuntu-vm.yml                baseline for group ubuntu (waits for cloud-init first)
  os/setup-suse-vm.yml                  baseline for group suse (installs Python 3.11 first)
  tasks/suse_python.yml        shared by bootstrap.yml and os/setup-suse-vm.yml
  patch.yml                    rolling OS patching
roles/
  <role>/
    README.md                  what the role does, why, and every variable it reads
    defaults/main.yml          default values (lowest precedence, meant to be overridden)
    vars/                      per-OS data internal to the role (not meant to be overridden)
    meta/argument_specs.yml    variable contract, validated before the first task runs
    meta/main.yml              author, licence, supported platforms
    tasks/main.yml             order only; imports one file per concern
    tasks/<concern>.yml        e.g. sudoers.yml, sshd.yml, update_redhat.yml
    handlers/ templates/
inventory/                     published example; copy it outside the repo per environment
  hosts.yml
  group_vars/
    all/connection.yml         how Ansible logs in
    all/baseline.yml           site-wide inputs for the roles
    all/vault.yml.example      names of the secrets, never the values
    rhel.yml ubuntu.yml suse.yml   per-distribution login user and Python path
    uat.yml prod.yml           per-environment behaviour
  host_vars/                   rare one-host exceptions
```

## Roles

| Role | Purpose |
|---|---|
| [linux_preflight](roles/linux_preflight/README.md) | Refuses unsupported distributions and releases |
| [linux_access](roles/linux_access/README.md) | Admin and automation accounts, SSH keys, sudoers drop-ins, password ageing, root break-glass password |
| [linux_password_policy](roles/linux_password_policy/README.md) | Password length and character rules, no username, history of 5, lockout after 3 failures |
| [linux_packages](roles/linux_packages/README.md) | Base admin and troubleshooting tools, one list per OS family |
| [linux_timesync](roles/linux_timesync/README.md) | chrony as the only time service, with a source check |
| [linux_hardening](roles/linux_hardening/README.md) | sshd baseline (drop-in or main file), idle shell and logind timeouts |
| [linux_log_forwarding](roles/linux_log_forwarding/README.md) | rsyslog running, collector reachability |
| [linux_patching](roles/linux_patching/README.md) | Mirror check, updates, conditional reboot, kernel report |

## How variables are organised

Every variable has exactly one home, chosen by who decides its value:

| Who decides | Where it lives | Example |
|---|---|---|
| The role author: a safe default | `roles/<role>/defaults/main.yml` | `linux_access_pass_max_days: 90` |
| The role author: OS facts, not settings | `roles/<role>/vars/` | chrony's config path on SUSE |
| The site: same for every host | `group_vars/all/` | NTP servers, admin accounts |
| The distribution | `group_vars/rhel.yml`, `ubuntu.yml`, `suse.yml` | `bootstrap_user: ubuntu`, `linux_preflight_expected_os_family: Debian` |
| The environment | `group_vars/uat.yml`, `prod.yml` | `linux_log_forwarding_required: true` |
| One exceptional host | `host_vars/<host>.yml` | no automatic reboot on an appliance |
| A secret | `group_vars/all/vault.yml` in your own inventory copy, encrypted | root password hash |
| One run | `-e` on the command line | `-e linux_patching_batch=1` |

The rules behind this layout:

- **Role variables are prefixed with the role name** (`linux_access_*`, `linux_timesync_*`), so you can always tell which role reads a variable and two roles never collide. ansible-lint enforces this.
- **Inventory files set only what differs from the defaults.** Everything else falls through to `defaults/main.yml`, so the inventory stays short and reads as a list of site decisions.
- **Secrets are indirect.** Plain files reference `{{ vault_* }}` names, and the values live only in the encrypted `vault.yml`. You can search the repo for every place a secret is used without ever seeing one.
- **Inputs are validated.** Each role's `meta/argument_specs.yml` declares types, required fields and allowed values. A typo such as `linux_access_pass_max_days: "ninety"` or an admin without a `name` stops the run before the first change.
- **Hosts are grouped by distribution and by environment at the same time.** Distribution groups decide how to log in, environment groups decide how cautious to be, and `-l 'ubuntu:&prod'` combines them.

## OS playbooks

Each OS family has its own playbook under `playbooks/os/`, and `site.yml` imports all three. They run the same roles; only the steps that really differ per family live in the OS playbook:

| Playbook | Hosts | Extra steps |
|---|---|---|
| `os/rhel.yml` | group `rhel` | none |
| `os/ubuntu.yml` | group `ubuntu` | waits for cloud-init, which holds the apt lock on a fresh cloud image |
| `os/suse.yml` | group `suse` | installs Python 3.11 with `raw` before facts are gathered |

Everything else stays in the roles, which handle OS differences through `vars/<os_family>.yml` files. A fix to a role therefore reaches all three families at once, instead of being copied into three playbooks.

Group membership is checked too: each distribution group sets `linux_preflight_expected_os_family`, and `linux_preflight` fails a host whose real OS family does not match. An Ubuntu host listed under `rhel` stops before it is touched.

## Security checklist coverage

| Control | Setting | Where |
|---|---|---|
| Root login disabled | `PermitRootLogin no` | linux_hardening |
| Empty passwords denied | `PermitEmptyPasswords no` | linux_hardening |
| 3 authentication attempts per connection | `MaxAuthTries 3` | linux_hardening |
| Login grace time 1 minute | `LoginGraceTime 60` | linux_hardening |
| Idle session timeout 5 minutes | `TMOUT=300`, logind `StopIdleSessionSec=300` | linux_hardening |
| Password ageing for new accounts | 90 / 1 / 7 days in `login.defs` | linux_access |
| Minimum length 12 | `minlen = 12` | linux_password_policy |
| Lowercase, uppercase, digit and special character | `lcredit`, `ucredit`, `dcredit`, `ocredit = -1` | linux_password_policy |
| No username in the password, in any form | `usercheck = 1` | linux_password_policy |
| Last 5 passwords cannot be reused | `pam_pwhistory remember=5` | linux_password_policy |
| Lockout after 3 failures in 5 minutes, for 30 minutes | `pam_faillock deny=3 fail_interval=300 unlock_time=1800` | linux_password_policy |

On Ubuntu 24.04 the password and lockout rules were checked on a live host: weak, short and username-containing passwords are rejected by `pwscore`, and after three wrong passwords the correct one is refused with "30 minutes left to unlock". RHEL and SUSE runs are still to be done.

## Design decisions

- **Order prevents lockouts.** Bootstrap creates key-based accounts first. Only then does `site.yml` set `PermitRootLogin no` and `PasswordAuthentication no`, and the sshd config is validated with `sshd -t` before the service reloads.
- **People and automation are separate accounts.** Admins log in with their own key and type their own password for sudo. Passwordless sudo is limited to the automation account, which has a locked password and is reachable by key only.
- **Password ageing applies to every interactive account.** 90 day maximum, 1 day minimum and 7 day warning, including root's vaulted break-glass password.
- **Config layout is detected, not assumed.** The sshd role checks whether `sshd_config` includes `sshd_config.d/` instead of guessing from the distribution name.
- **Version checks are numeric.** `distribution_major_version | int`, never a string comparison, which would sort "10" before "8".
- **Patching is rolling.** Batches of 25% by default with `max_fail_percentage: 0`, so one failed host stops the rollout. Reboots happen only when the OS reports one is needed.

## Quick start

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install ansible-core
ansible-galaxy collection install -r requirements.yml

# Real inventories live outside the repo, one directory per environment
mkdir -p ~/inventories && cp -r inventory ~/inventories/lab
cp ~/inventories/lab/group_vars/all/vault.yml.example ~/inventories/lab/group_vars/all/vault.yml
ansible-vault encrypt ~/inventories/lab/group_vars/all/vault.yml
# edit hosts.yml and group_vars/all/*.yml there

I="-i ~/inventories/lab/hosts.yml --ask-vault-pass"
ansible-playbook $I playbooks/bootstrap.yml -l new-host
ansible-playbook $I playbooks/site.yml --check --diff
ansible-playbook $I playbooks/site.yml
ansible-playbook $I playbooks/os/ubuntu.yml          # one OS family only
ansible-playbook $I playbooks/patch.yml -l uat
ansible-playbook $I playbooks/patch.yml -l prod -e linux_patching_batch=1
```

Every task file has its own tag, so one part can be re-applied alone:

```bash
ansible-playbook $I playbooks/site.yml --tags access_sudoers     # only the sudoers drop-ins
ansible-playbook $I playbooks/site.yml --tags hardening_sshd     # only the sshd baseline
```

`bootstrap.yml` logs in with each distribution's launch user (`bootstrap_user`) and launch key, then every later run uses the automation account.

## Testing

CI runs `yamllint`, `ansible-lint` (production profile) and a syntax check of every playbook on each push.

## Licence

MIT
