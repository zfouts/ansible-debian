# What the role changes

Every file, package and service the role touches, grouped by tag, with how to
undo each change by hand. Files the role creates start with an
`Ansible managed` header.

- [packages](#packages)
- [banners](#banners)
- [ssh](#ssh)
- [firewall](#firewall)
- [fail2ban](#fail2ban)
- [sysctl](#sysctl)
- [kernel_modules](#kernel_modules)
- [filesystem](#filesystem)
- [accounts](#accounts)
- [services](#services)
- [auditd](#auditd)
- [logging](#logging)
- [aide](#aide)

## packages

| Change | Details |
|---|---|
| Upgrade | `apt-get upgrade --with-new-pkgs` (when `hardening_upgrade_packages`) |
| Purged | `hardening_remove_packages` |
| Installed | `unattended-upgrades`, `apt-listchanges` |
| `/etc/apt/apt.conf.d/20auto-upgrades` | Daily package list refresh and unattended upgrade, weekly autoclean |
| `/etc/apt/apt.conf.d/52unattended-upgrades-local` | Removes unused kernels, sets the auto-reboot policy, logs to syslog |

**Revert:** delete both files, or set `APT::Periodic::Unattended-Upgrade "0";`.

## banners

| Change | Details |
|---|---|
| `/etc/issue` | Replaced with `hardening_banner` |
| `/etc/issue.net` | Replaced with `hardening_banner` |

**Revert:** write your own text to both files. Debian's originals are
`Debian GNU/Linux <version> \n \l` (`/etc/issue`) and
`Debian GNU/Linux <version>` (`/etc/issue.net`).

## ssh

| Change | Details |
|---|---|
| Installed | `openssh-server` |
| `/etc/ssh/sshd_config` | Gains `Include /etc/ssh/sshd_config.d/*.conf` at the top if it's missing (Debian already has it) |
| `/etc/ssh/sshd_config.d/00-hardening.conf` | All SSH settings. sshd uses the first value it reads, so this file wins over the main config and over later drop-ins such as cloud-init's `50-cloud-init.conf`. |
| `ssh.service` | Enabled. Reloaded (not restarted) after the full configuration passes `sshd -t`. |

**Revert:** `rm /etc/ssh/sshd_config.d/00-hardening.conf && systemctl reload ssh`

## firewall

| Change | Details |
|---|---|
| Installed | `ufw` |
| Rules | SSH (`limit` or `allow`) on `hardening_ssh_port`, plus `hardening_firewall_allowed_ports`. Rules are only added; changing a port leaves the old rule in place. |
| Policy | Deny incoming, allow outgoing. UFW's default of denying routed traffic is unchanged. |
| State | Enabled, logging `low` |

**Revert:** `ufw disable`, or `ufw reset` to also delete every rule.

## fail2ban

| Change | Details |
|---|---|
| Installed | `fail2ban`, `python3-systemd`, `nftables` |
| `/etc/fail2ban/jail.d/hardening.local` | Ban settings, nftables ban actions, `sshd` jail on the systemd journal |
| `fail2ban.service` | Enabled and running |

Debian 12 and 13 log to the journal only; without `/var/log/auth.log`, the
stock Debian 12 configuration fails to start. The journal backend fixes that.

**Revert:** `rm /etc/fail2ban/jail.d/hardening.local && systemctl restart fail2ban`,
or `apt-get purge fail2ban`.

## sysctl

| Change | Details |
|---|---|
| `/etc/sysctl.d/99-hardening.conf` | `hardening_sysctl_settings` merged with the forwarding settings and `hardening_sysctl_extra`. Applied immediately. |

**Revert:** delete the file and reboot. Values stay active until then, and
`kernel.unprivileged_bpf_disabled=1` can't be reverted without a reboot.

## kernel_modules

| Change | Details |
|---|---|
| `/etc/modprobe.d/hardening.conf` | `install <module> /bin/false` and `blacklist <module>` for each of `hardening_blacklisted_modules` |

Modules that are already loaded stay loaded until the next reboot.

**Revert:** delete the file.

## filesystem

| Change | Details |
|---|---|
| `/etc/fstab` | `tmpfs /dev/shm tmpfs defaults,nodev,nosuid,noexec 0 0`, and `/dev/shm` is remounted |
| Permissions | `/etc/passwd`, `/etc/group` and their backups: `0644 root:root`. `/etc/shadow`, `/etc/gshadow` and their backups: `0640 root:shadow`. `/etc/crontab`: `0600`. `/etc/cron.{hourly,daily,weekly,monthly,d}`: `0700`. |
| `/etc/security/limits.d/10-hardening.conf` | `* hard core 0` |
| `/etc/systemd/coredump.conf.d/hardening.conf` | `Storage=none`, `ProcessSizeMax=0` |

**Revert `/dev/shm`:** remove the line from `/etc/fstab`, then
`mount -o remount,exec,suid,dev /dev/shm`.
**Revert core dumps:** delete both files.

## accounts

| Change | Details |
|---|---|
| `/etc/login.defs` | `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`, `UMASK`. Aging only applies to accounts created afterwards. |
| `/usr/share/pam-configs/umask` | Debian 12 only: a pam-auth-update profile that enables pam_umask, which Debian 13 already enables |
| Installed | `libpam-pwquality`, which adds `pam_pwquality` to `common-password` |
| `/etc/security/pwquality.conf.d/50-hardening.conf` | Password quality rules |
| `/etc/security/faillock.conf` | Lockout thresholds |
| `/usr/share/pam-configs/faillock`, `faillock_notify` | pam-auth-update profiles that add pam_faillock to `common-auth` and `common-account` |
| `/etc/pam.d/su` | `auth required pam_wheel.so use_uid group=sudo` |
| `/etc/sudoers.d/50-hardening` | `Defaults use_pty` and `Defaults logfile=...` (only if sudo is installed). Checked with `visudo -c` before it's written. |

The PAM stacks are only changed through `pam-auth-update`, Debian's supported
mechanism. That keeps them consistent and upgradeable. With faillock enabled,
`common-auth` reads:

```
auth  requisite                   pam_faillock.so preauth
auth  [success=2 default=ignore]  pam_unix.so nullok try_first_pass
auth  [default=die]               pam_faillock.so authfail
auth  requisite                   pam_deny.so
auth  required                    pam_permit.so
```

**Revert faillock:** `pam-auth-update --disable faillock faillock_notify`, then
delete both profiles.
**Revert pam_umask (Debian 12):** `pam-auth-update --disable umask`, then delete the profile.
**Revert su:** comment out the `pam_wheel.so` line in `/etc/pam.d/su`.
**Revert sudo:** delete `/etc/sudoers.d/50-hardening`.

## services

| Change | Details |
|---|---|
| Installed | `apparmor`, `apparmor-utils` |
| `apparmor.service` | Enabled |
| `ctrl-alt-del.target` | Masked, so Ctrl-Alt-Del on the console doesn't reboot |
| `hardening_disabled_services` | Stopped, disabled and masked if installed |

**Revert:** `systemctl unmask <unit>`.

## auditd

| Change | Details |
|---|---|
| Installed | `auditd`, `audispd-plugins` |
| `auditd.service` | Enabled and running |
| `/etc/audit/rules.d/50-hardening.rules` | The audit rules, loaded with `augenrules --load` |
| `/etc/audit/rules.d/99-finalize.rules` | `-e 2`, only when `hardening_auditd_immutable` is set. Removed otherwise. |
| `/etc/default/grub.d/50-hardening-audit.cfg` | Appends `audit=1 audit_backlog_limit=8192` to `GRUB_CMDLINE_LINUX`, then runs `update-grub`. Only on hosts with GRUB. |

**Revert:** delete the rule files and run `augenrules --load` (after a reboot
if the rules were locked). Delete the GRUB file and run `update-grub`.

## logging

| Change | Details |
|---|---|
| `/etc/systemd/journald.conf.d/50-hardening.conf` | `Storage=persistent`, `Compress=yes` |

**Revert:** delete the file and `systemctl restart systemd-journald`.

## aide

Only when `hardening_aide: true`.

| Change | Details |
|---|---|
| Installed | `aide`, `aide-common` |
| `/var/lib/aide/aide.db` | Initial baseline built with `aideinit`. aide-common's `dailyaidecheck.timer` compares against it every day. |

After intended changes such as package upgrades, refresh the baseline with
`aideinit --yes --force`.
