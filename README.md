# ansible-debian

An Ansible role and playbook that hardens Debian 12 (bookworm) and Debian 13
(trixie) servers.

It follows the [CIS Debian Linux Benchmark](https://www.cisecurity.org/benchmark/debian_linux)
where that's practical on a production server. It departs from it, on purpose,
where a control is likely to lock you out or break common workloads. Each
departure is [listed below](#design-decisions) and can be switched back on.

- **Lockout protection**: the role refuses to apply SSH settings that would
  cut off the account Ansible is connected as.
- **Idempotent**: a second run reports no changes, so it's safe to run on a schedule.
- **Configurable**: every component can be tuned with variables or skipped with tags.
- **Debian-native**: PAM is changed through `pam-auth-update`, and configuration
  goes into drop-in directories wherever Debian supports them, so package
  upgrades don't fight the role.

## Contents

- [What it does](#what-it-does)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Lockout protection](#lockout-protection)
- [Configuration](#configuration)
- [Running part of the role](#running-part-of-the-role)
- [After the first run](#after-the-first-run)
- [Design decisions](#design-decisions)
- [Testing changes to the role](#testing-changes-to-the-role)
- [Repository layout](#repository-layout)
- [Further documentation](#further-documentation)

## What it does

| Tag | Area | Summary |
|---|---|---|
| `packages` | Updates | Upgrades packages, purges telnet, rsh, NIS, talk, xinetd and tftpd, enables unattended-upgrades |
| `banners` | Login banners | Replaces `/etc/issue` and `/etc/issue.net`, which reveal the OS release, with a legal notice |
| `ssh` | OpenSSH | Key-only logins, no root login, no forwarding, modern key exchange (including post-quantum hybrids), ciphers and MACs, `MaxAuthTries 4`, idle timeout, verbose logging |
| `firewall` | UFW | Deny inbound, allow outbound, rate-limited SSH, plus the ports you list |
| `fail2ban` | Brute-force protection | Bans IPs after repeated SSH failures, enforced with nftables and read from the systemd journal |
| `sysctl` | Kernel | ASLR, hidden kernel pointers and log, restricted ptrace and BPF, protected links, no ICMP redirects or source routing, reverse-path filtering, SYN cookies, no forwarding |
| `kernel_modules` | Kernel | Blocks rarely used filesystems and network protocols (cramfs, hfs, jffs2, dccp, sctp, rds, tipc and others) |
| `filesystem` | Filesystem | `/dev/shm` mounted `nodev,nosuid,noexec`, strict permissions on account and cron files, core dumps off |
| `accounts` | Authentication | umask 027, password quality (14+ characters), account lockout after 5 failures, `su` limited to the `sudo` group, sudo logging with `use_pty` |
| `services` | Services | AppArmor enabled, Ctrl-Alt-Del reboot masked, avahi and cups masked |
| `auditd` | Auditing | CIS-style audit rules for identity, time, network, login, sudo, permission, access, deletion, mount and module events, plus `audit=1` at boot |
| `logging` | Logging | Persistent, compressed systemd journal |
| `aide` | File integrity | AIDE baseline and daily check. **Opt-in.** |

[docs/changes.md](docs/changes.md) lists every file, package and service the
role touches, and how to undo each change.

## Requirements

**Control node**

- ansible-core 2.15 or newer (tested with 2.21)
- The `ansible.posix` and `community.general` collections (see [Quick start](#quick-start))

**Managed hosts**

- Debian 12 or 13. The audit syscall rules cover amd64 and arm64; other
  architectures get the file-watch rules only.
- Python 3, which every standard Debian installation includes
- A non-root account with sudo rights that logs in with an SSH key

## Quick start

1. Install the collections:

   ```sh
   ansible-galaxy collection install -r requirements.yml
   ```

2. Add your hosts to [`inventory.ini`](inventory.ini):

   ```ini
   [debian]
   web1.example.com ansible_user=admin
   ```

3. Put any overrides in `group_vars/debian.yml` (see [Configuration](#configuration)).
   Docker, Kubernetes and VPN hosts **must** set `hardening_sysctl_ip_forwarding: true`.

4. Run it against one host first, with console access to that host available:

   ```sh
   ansible-playbook site.yml --limit web1.example.com
   ```

5. Reboot that host (see [After the first run](#after-the-first-run)), check it,
   then roll out to the rest:

   ```sh
   ansible-playbook site.yml
   ```

Once a host has been hardened, `--check --diff` previews later changes
accurately. On a host the role has never run on, check mode stops early,
because it can't install the packages the role depends on. See
[troubleshooting](docs/troubleshooting.md#check-mode-fails-on-a-new-host).

## Lockout protection

Before it changes SSH, the role works out which user Ansible logged in as and
stops the play if the new configuration would shut that user out:

- You're connected as **root** while `hardening_ssh_permit_root_login` is `"no"`.
- The user has **no `~/.ssh/authorized_keys`** while password logins are being disabled.
- `hardening_ssh_allow_users` or `hardening_ssh_allow_groups` is set and **leaves the user out**.

SSH changes also go through two validation steps. Each file is checked with
`sshd -t` before it's written, and the combined configuration is checked again
before sshd is reloaded. A reload leaves open sessions untouched.

The firewall allows SSH before it switches on, so the session running the play
stays connected.

These checks can't cover everything, such as keys served by
`AuthorizedKeysCommand`. Keep a console session available the first time you run it.

## Configuration

Set variables in `group_vars/debian.yml`, `host_vars/<host>.yml`, or with `-e`.
[docs/variables.md](docs/variables.md) is the full reference. Common cases:

**Docker, Kubernetes, VPN gateway or router.** Keeps IP forwarding on:

```yaml
hardening_sysctl_ip_forwarding: true
```

**Open more ports:**

```yaml
hardening_firewall_allowed_ports:
  - { port: 80, comment: HTTP }
  - { port: 443, comment: HTTPS }
  - { port: 51820, proto: udp, comment: WireGuard }
  - { port: 9100, from: 10.0.0.0/8, comment: node_exporter }
```

**Allow SSH only for one group:**

```yaml
hardening_ssh_allow_groups: [sudo]
```

**Never ban your own networks:**

```yaml
hardening_fail2ban_ignoreip: [127.0.0.1/8, "::1", 198.51.100.0/24]
```

**Move SSH to another port.** The role opens the new port in the firewall, and
sshd and fail2ban move to it at the end of the run. The rule for the old port
is left in place, so you can't lock yourself out by mistake.

```yaml
hardening_ssh_port: 2222
```

Then set `ansible_port: 2222` in the inventory. Once you've confirmed you can
log in on the new port, remove the old rule with `sudo ufw delete limit 22/tcp`.

**Closer to CIS**, at the cost of the trade-offs described under
[Design decisions](#design-decisions):

```yaml
hardening_auditd_immutable: true
hardening_password_max_days: 365
hardening_pwquality_minclass: 4
hardening_aide: true
```

## Running part of the role

Each area has a tag (see [What it does](#what-it-does)). The preflight checks always run.

```sh
ansible-playbook site.yml --tags ssh,firewall       # only these areas
ansible-playbook site.yml --skip-tags auditd        # everything else
ansible-playbook site.yml --check --diff            # preview changes on an already-hardened host
```

`firewall`, `fail2ban`, `sysctl`, `auditd` and `aide` can also be switched off
permanently with their `hardening_<area>` variable.

## After the first run

**Reboot.** The reboot enables `audit=1` on the kernel command line, unloads
blocked modules that were already loaded, and starts the upgraded kernel if
there was one.

**Check the result:**

```sh
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kexalgorithms) '
sudo ufw status verbose
sudo fail2ban-client status sshd
sudo auditctl -l | wc -l          # number of loaded audit rules
sudo aa-status | head -3          # AppArmor profiles
umask                             # 0027 (0007 for private-group users on Debian 13)
```

For an outside view of the SSH configuration, run
[ssh-audit](https://github.com/jtesta/ssh-audit) from your workstation:
`ssh-audit web1.example.com`.

If something stops working, see [docs/troubleshooting.md](docs/troubleshooting.md).

## Design decisions

Where this role differs from the CIS benchmark, and why:

| CIS recommendation | This role | Reason |
|---|---|---|
| Passwords expire after 365 days | Debian default (no expiry) | Through PAM, an expired password also blocks **SSH key** logins. Key-only accounts would be locked out a year after they're created. |
| Audit rules immutable (`-e 2`) | Off by default | Locked rules can't be updated by later runs until a reboot. Set `hardening_auditd_immutable: true` once the rules are settled. |
| Ignore IPv6 router advertisements | Left accepted | Many networks, including most clouds, assign IPv6 addresses through router advertisements. |
| Block `udf`, `squashfs`, `usb-storage` | Not blocked | Azure provisions VMs from a UDF image, snaps need squashfs, and some servers need USB storage. Add them to `hardening_blacklisted_modules` if they don't apply to you. |
| Separate partitions for `/tmp`, `/var`, `/home` | Not managed | Partitioning is decided at install time and differs by environment. New Debian 13 installs already mount `/tmp` as tmpfs. |
| Shell idle timeout (`TMOUT`) | Not set | It also kills idle shells inside tmux and screen, which is rarely what anyone wants on a server. |
| AIDE installed | Opt-in | The first scan can take a long time on large filesystems. |
| umask 027 for every user | 027, or 007 for user-private-group accounts on Debian 13 | Debian 13's pam_umask follows `USERGROUPS_ENAB`. Each user's private group contains only that user, so the result is the same. Forcing 027 would change how `useradd` creates accounts. |

Other choices:

- **UFW** rather than hand-written nftables rules, because it's easier to
  inspect and extend. fail2ban uses its own nftables table, separate from UFW's.
- **SSH algorithms are filtered per host** with `ssh -Q`, so one list works on
  both OpenSSH 9.2 (Debian 12) and 10.0 (Debian 13). Debian 13 gets
  `mlkem768x25519-sha256`.
- **SSH settings go in `sshd_config.d/00-hardening.conf`.** sshd uses the first
  value it reads, so the role's settings win over the stock config and over
  drop-ins such as cloud-init's `50-cloud-init.conf`, which often re-enables
  password logins.

## Testing changes to the role

Test in a disposable VM before using a change in production. Containers can
test most of the role: a privileged container running systemd, with sshd and a
sudo user, behaves much like a server. Skip two tags there:

- `sysctl`: in a privileged container it would change the kernel shared with
  the container host.
- `auditd`: the kernel only accepts audit configuration from the host's
  initial PID namespace.

```sh
ansible-playbook -i test-inventory.ini site.yml --skip-tags sysctl,auditd
ansible-playbook -i test-inventory.ini site.yml --skip-tags sysctl,auditd   # expect changed=0
```

## Repository layout

```
.
├── site.yml                      # playbook: applies the role to the [debian] group
├── inventory.ini                 # your hosts
├── requirements.yml              # collections
├── ansible.cfg
├── docs/
│   ├── variables.md              # every variable, with defaults
│   ├── changes.md                # every file and service touched, and how to revert
│   └── troubleshooting.md        # lockouts, bans and other problems
└── roles/hardening/
    ├── defaults/main.yml         # tunable variables
    ├── vars/main.yml             # internal data (file permissions, audit syscalls)
    ├── tasks/                    # one file per area, matching the tags
    ├── handlers/main.yml         # sshd validate/reload, fail2ban, audit, GRUB, journald
    ├── templates/                # configuration files
    └── files/pam-configs/        # pam-auth-update profiles
```

## Further documentation

- [docs/variables.md](docs/variables.md): every variable, its default and what it does
- [docs/changes.md](docs/changes.md): what the role changes on a host, and how to undo it
- [docs/troubleshooting.md](docs/troubleshooting.md): recovering from lockouts, bans and other problems
