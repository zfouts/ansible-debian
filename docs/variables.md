# Variable reference

All variables live in [`roles/hardening/defaults/main.yml`](../roles/hardening/defaults/main.yml)
and can be overridden in `group_vars/`, `host_vars/`, or with `-e` on the command line.

Overriding a list or dictionary **replaces** it rather than merging with it. To
add sysctl keys, use `hardening_sysctl_extra` instead of redefining
`hardening_sysctl_settings`. To extend a list, copy the default and add to it.

- [Packages and updates](#packages-and-updates)
- [SSH](#ssh)
- [Firewall](#firewall)
- [fail2ban](#fail2ban)
- [Kernel](#kernel)
- [Filesystem](#filesystem)
- [Accounts and authentication](#accounts-and-authentication)
- [Auditing and logging](#auditing-and-logging)

## Packages and updates

| Variable | Default | Description |
|---|---|---|
| `hardening_upgrade_packages` | `true` | Upgrade installed packages (`apt-get upgrade --with-new-pkgs`) during the run. |
| `hardening_unattended_upgrades` | `true` | Install and enable unattended-upgrades. Debian's stock origins (stable point releases and the security archive) are kept. |
| `hardening_unattended_upgrades_reboot` | `false` | Reboot automatically when an installed update requires it. |
| `hardening_unattended_upgrades_reboot_time` | `"03:00"` | Time of day for those automatic reboots. |
| `hardening_remove_packages` | telnet, inetutils-telnet, telnetd, inetutils-telnetd, rsh-client, rsh-server, rsh-redone-client, nis, talk, talkd, xinetd, tftpd-hpa | Packages that are purged. Names missing from the archive are ignored. |
| `hardening_disabled_services` | `[avahi-daemon, cups, cups-browsed]` | Services that are stopped, disabled and masked if they are installed. Use names without the `.service` suffix. |

## SSH

| Variable | Default | Description |
|---|---|---|
| `hardening_ssh_port` | `22` | Port sshd listens on. fail2ban follows it, and the firewall opens it. A rule for a previous port isn't removed. |
| `hardening_ssh_permit_root_login` | `"no"` | `"no"` or `"prohibit-password"` (root may log in with a key only). |
| `hardening_ssh_password_authentication` | `false` | Allow password and keyboard-interactive logins. |
| `hardening_ssh_allow_tcp_forwarding` | `false` | Allow TCP port forwarding (`ssh -L` / `-R`). |
| `hardening_ssh_allow_agent_forwarding` | `false` | Allow `ssh -A` agent forwarding. |
| `hardening_ssh_x11_forwarding` | `false` | Allow X11 forwarding. |
| `hardening_ssh_max_auth_tries` | `4` | Authentication attempts per connection. Clients with many keys loaded in an agent may hit this limit; use `IdentitiesOnly yes` in your SSH client config. |
| `hardening_ssh_max_sessions` | `10` | Sessions multiplexed over one connection. |
| `hardening_ssh_login_grace_time` | `30` | Seconds a client has to finish logging in. |
| `hardening_ssh_client_alive_interval` | `300` | Seconds of silence before sshd sends a keepalive probe to the client. |
| `hardening_ssh_client_alive_count_max` | `2` | Unanswered probes before sshd drops the connection as dead (here after 10 minutes). Idle sessions whose client still responds are not affected. |
| `hardening_ssh_allow_users` | `[]` | If set, only these users may log in (`AllowUsers`). |
| `hardening_ssh_allow_groups` | `[]` | If set, only members of these groups may log in (`AllowGroups`). |
| `hardening_ssh_kex_algorithms` | ML-KEM and sntrup761 hybrids, curve25519, DH group 16/18 | Key exchange algorithms, strongest first. |
| `hardening_ssh_ciphers` | chacha20-poly1305, AES-GCM, AES-CTR | Ciphers, strongest first. |
| `hardening_ssh_macs` | HMAC-SHA2-512/256-ETM, UMAC-128-ETM | MACs, strongest first. |
| `hardening_ssh_lockout_check` | `true` | Abort before changing SSH if the settings would lock out the user Ansible logs in as. See [lockout protection](../README.md#lockout-protection). |

Algorithms the installed OpenSSH doesn't support are dropped automatically.
Debian 12 (OpenSSH 9.2) has no ML-KEM, for example, so the same list works on
both releases.

## Firewall

| Variable | Default | Description |
|---|---|---|
| `hardening_firewall` | `true` | Install and enable UFW: deny inbound, allow outbound. |
| `hardening_firewall_ssh_rate_limit` | `true` | Use UFW's `limit` rule for SSH, which drops a source IP after 6 connections in 30 seconds. Set to `false` if automation opens many SSH connections. |
| `hardening_firewall_allowed_ports` | `[]` | Extra inbound rules (see below). |

Each entry in `hardening_firewall_allowed_ports` takes:

| Key | Required | Default | Description |
|---|---|---|---|
| `port` | yes | | Port number or range (`"60000:61000"`). |
| `proto` | no | `tcp` | `tcp`, `udp` or `any`. |
| `from` | no | `any` | Source address or CIDR. |
| `comment` | no | | Comment shown in `ufw status`. |

```yaml
hardening_firewall_allowed_ports:
  - { port: 443, comment: HTTPS }
  - { port: 51820, proto: udp, comment: WireGuard }
  - { port: 9100, from: 10.0.0.0/8, comment: node_exporter }
```

## fail2ban

| Variable | Default | Description |
|---|---|---|
| `hardening_fail2ban` | `true` | Install fail2ban with an SSH jail. Bans are enforced with nftables. |
| `hardening_fail2ban_bantime` | `1h` | Ban duration. |
| `hardening_fail2ban_findtime` | `10m` | Window in which failures are counted. |
| `hardening_fail2ban_maxretry` | `5` | Failures within `findtime` that trigger a ban. |
| `hardening_fail2ban_ignoreip` | `[127.0.0.1/8, "::1"]` | Addresses that are never banned. Add your office or bastion ranges here. |

## Kernel

| Variable | Default | Description |
|---|---|---|
| `hardening_sysctl` | `true` | Apply the kernel parameters below. They are written to `/etc/sysctl.d/99-hardening.conf`. |
| `hardening_sysctl_ip_forwarding` | `false` | Leave IP forwarding alone. **Set to `true` on Docker, Kubernetes, VPN and router hosts.** When `false`, `net.ipv4.ip_forward` and `net.ipv6.conf.all.forwarding` are set to 0. |
| `hardening_sysctl_settings` | see below | Base parameter set. |
| `hardening_sysctl_extra` | `{}` | Parameters merged over the base set. Use it to add or override keys. |
| `hardening_blacklisted_modules` | cramfs, freevxfs, hfs, hfsplus, jffs2, dccp, rds, sctp, tipc | Kernel modules that can't be loaded (`install <mod> /bin/false`). |

Default `hardening_sysctl_settings`:

| Parameter | Value | Purpose |
|---|---|---|
| `kernel.randomize_va_space` | 2 | Full address space layout randomization |
| `kernel.kptr_restrict` | 2 | Hide kernel pointers from everyone, including root |
| `kernel.dmesg_restrict` | 1 | Only root can read the kernel log |
| `kernel.yama.ptrace_scope` | 1 | Processes can only ptrace their own descendants |
| `kernel.sysrq` | 0 | Disable magic SysRq keys |
| `kernel.unprivileged_bpf_disabled` | 1 | Only root can load BPF programs (can't be undone without a reboot) |
| `net.core.bpf_jit_harden` | 2 | Harden the BPF JIT for all users |
| `fs.protected_hardlinks`, `fs.protected_symlinks` | 1 | Block hardlink and symlink attacks in shared directories |
| `fs.protected_fifos`, `fs.protected_regular` | 2 | Block FIFO and regular-file spoofing in sticky directories |
| `fs.suid_dumpable` | 0 | No core dumps from setuid programs |
| `net.ipv4/6.conf.{all,default}.accept_redirects` | 0 | Ignore ICMP redirects |
| `net.ipv4.conf.{all,default}.secure_redirects` | 0 | Ignore redirects even from gateways |
| `net.ipv4.conf.{all,default}.send_redirects` | 0 | Don't send redirects |
| `net.ipv4/6.conf.{all,default}.accept_source_route` | 0 | Drop source-routed packets |
| `net.ipv4.conf.{all,default}.log_martians` | 1 | Log packets with impossible addresses |
| `net.ipv4.conf.{all,default}.rp_filter` | 1 | Strict reverse-path filtering |
| `net.ipv4.icmp_echo_ignore_broadcasts` | 1 | Ignore broadcast pings |
| `net.ipv4.icmp_ignore_bogus_error_responses` | 1 | Ignore bogus ICMP errors |
| `net.ipv4.tcp_syncookies` | 1 | SYN flood protection |

Multi-homed hosts with asymmetric routing may need loose reverse-path filtering:

```yaml
hardening_sysctl_extra:
  net.ipv4.conf.all.rp_filter: 2
  net.ipv4.conf.default.rp_filter: 2
```

## Filesystem

| Variable | Default | Description |
|---|---|---|
| `hardening_dev_shm_noexec` | `true` | Mount `/dev/shm` with `nodev,nosuid,noexec` (added to `/etc/fstab`). |
| `hardening_disable_core_dumps` | `true` | Set a hard core-size limit of 0 and stop systemd-coredump from storing dumps. |

Permissions on `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`, their
backups, `/etc/crontab` and `/etc/cron.*` are always enforced. The list is in
[`roles/hardening/vars/main.yml`](../roles/hardening/vars/main.yml).

## Accounts and authentication

| Variable | Default | Description |
|---|---|---|
| `hardening_umask` | `"027"` | Default umask (`UMASK` in `/etc/login.defs`, applied through pam_umask). On Debian 13, users whose primary group is their own private group get `007`; that group only contains them. |
| `hardening_password_min_days` | `1` | Minimum days between password changes, for new accounts. |
| `hardening_password_warn_age` | `7` | Days of warning before a password expires, for new accounts. |
| `hardening_password_max_days` | `99999` | Maximum password age for new accounts. Kept at Debian's default on purpose: an expired password also blocks SSH key logins through PAM. |
| `hardening_pwquality_minlen` | `14` | Minimum password length. |
| `hardening_pwquality_minclass` | `3` | Character classes (lower, upper, digit, other) a password must contain. |
| `hardening_faillock` | `true` | Lock accounts after repeated authentication failures (pam_faillock). root is never locked. |
| `hardening_faillock_deny` | `5` | Failures before the account is locked. |
| `hardening_faillock_unlock_time` | `900` | Seconds until a locked account unlocks automatically. |
| `hardening_faillock_fail_interval` | `900` | Window, in seconds, in which failures are counted. |
| `hardening_restrict_su` | `true` | Only members of `hardening_su_group` can use `su`. |
| `hardening_su_group` | `sudo` | Group allowed to use `su`. |
| `hardening_sudo_logfile` | `/var/log/sudo.log` | Where sudo logs commands. sudo also runs every command in a pseudo-terminal (`use_pty`). |
| `hardening_banner` | legal notice | Text of `/etc/issue` (local console) and `/etc/issue.net` (shown before SSH login). |

Password quality also enforces `maxrepeat = 3`, `dictcheck`, `usercheck` and
`enforce_for_root`. Those values are set in
[`templates/pwquality.conf.j2`](../roles/hardening/templates/pwquality.conf.j2).

## Auditing and logging

| Variable | Default | Description |
|---|---|---|
| `hardening_auditd` | `true` | Install auditd and the audit rules. |
| `hardening_auditd_immutable` | `false` | Lock the audit rules in the kernel until the next reboot (`-e 2`), as CIS requires. Rule changes from later runs then wait for a reboot. |
| `hardening_auditd_grub` | `true` | Add `audit=1 audit_backlog_limit=8192` to the kernel command line on GRUB systems, so processes that start before auditd are audited too. Takes effect after a reboot. |
| `hardening_journald_persistent` | `true` | Keep the systemd journal on disk across reboots, compressed. |
| `hardening_aide` | `false` | Install AIDE and build the file-integrity baseline. Opt-in because the first scan can take a long time on large filesystems. |

The audit rules use the following keys, for searching with `ausearch -k <key>`:

| Key | Records |
|---|---|
| `time-change` | Changes to the system clock and time zone |
| `identity` | Changes to users, groups and password files |
| `system-locale` | Changes to the hostname, `/etc/hosts`, `/etc/issue*` and network configuration |
| `MAC-policy` | Changes to AppArmor policy |
| `logins` | Writes to lastlog, faillock and the wtmpdb/lastlog2 databases (Debian 13) |
| `session` | Writes to utmp, wtmp and btmp |
| `scope` | Changes to `/etc/sudoers` and `/etc/sudoers.d/` |
| `sudo_log_file` | Writes to the sudo log |
| `perm_mod` | chmod/chown/xattr calls by regular users |
| `access` | File opens refused with EACCES or EPERM, for regular users |
| `delete` | Unlinks and renames by regular users |
| `mounts` | Mounts by regular users |
| `kernel_modules` | Module loads and unloads, and runs of `kmod` |

Syscall rules are written for x86_64 (64-bit and 32-bit ABIs) and aarch64.
Syscall names differ between architectures (aarch64 only has the `*at` variants),
and the per-architecture lists are in
[`roles/hardening/vars/main.yml`](../roles/hardening/vars/main.yml). Other
architectures get the file watches only.
