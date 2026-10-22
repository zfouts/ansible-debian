# Troubleshooting

- [The play stops in preflight](#the-play-stops-in-preflight)
- [Locked out of SSH](#locked-out-of-ssh)
- [Banned by fail2ban](#banned-by-fail2ban)
- [SSH connections reset after a few attempts](#ssh-connections-reset-after-a-few-attempts)
- ["Too many authentication failures"](#too-many-authentication-failures)
- [sudo rejects the correct password](#sudo-rejects-the-correct-password)
- [su: Permission denied](#su-permission-denied)
- [A new password is rejected](#a-new-password-is-rejected)
- [Container or VPN networking stopped working](#container-or-vpn-networking-stopped-working)
- [A program fails with errors about /dev/shm](#a-program-fails-with-errors-about-devshm)
- [Audit rules don't change](#audit-rules-dont-change)
- [Check mode fails on a new host](#check-mode-fails-on-a-new-host)

## The play stops in preflight

Before touching SSH, the role checks that the new settings won't lock out the
user Ansible is logged in as.

| Message | Fix |
|---|---|
| *Ansible is logged in as root, and hardening_ssh_permit_root_login is "no"* | Connect as a sudo user (`ansible_user=...`), or set `hardening_ssh_permit_root_login: prohibit-password`. |
| *... has no ~/.ssh/authorized_keys, and password logins are about to be disabled* | Install a key with `ssh-copy-id`. If keys come from `AuthorizedKeysCommand` or certificates, set `hardening_ssh_lockout_check: false`. |
| *... is not listed in hardening_ssh_allow_users* | Add the user to the list. |
| *... is in none of hardening_ssh_allow_groups* | Add one of the user's groups (`id -nG`) to the list. |
| *This role supports Debian 12 (bookworm) and newer* | The host isn't Debian 12 or 13. |

## Locked out of SSH

Use the provider's console (or IPMI, or a hypervisor console) to log in locally,
then:

```sh
sshd -T | grep -Ei 'port|permitrootlogin|passwordauthentication|allow'   # effective settings
journalctl -u ssh -n 50                                                    # why logins fail
ufw status numbered                                                        # is the port open?
fail2ban-client status sshd                                                # is your IP banned?
```

To restore the stock SSH configuration:

```sh
rm /etc/ssh/sshd_config.d/00-hardening.conf
systemctl reload ssh
```

## Banned by fail2ban

Connect from another address or the console, then:

```sh
fail2ban-client status sshd             # list banned IPs
fail2ban-client unban 203.0.113.7       # unban one
fail2ban-client unban --all             # unban everything
```

To stop it happening again, add trusted ranges to `hardening_fail2ban_ignoreip`.
Bans expire after `hardening_fail2ban_bantime` (default 1 hour).

## SSH connections reset after a few attempts

UFW's `limit` rule drops a source address that opens more than 6 SSH
connections in 30 seconds. The block lifts after 30 seconds. Ansible normally
reuses one connection per host (`ControlPersist`), but tools that open many
parallel connections can hit the limit. Set
`hardening_firewall_ssh_rate_limit: false` to use a plain `allow` rule instead.

## "Too many authentication failures"

`MaxAuthTries` is 4, and an SSH agent holding more keys than that offers them
one by one until the server disconnects. Tell the client which key to use:

```
# ~/.ssh/config
Host server1
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

## sudo rejects the correct password

The account is locked by pam_faillock after `hardening_faillock_deny` failures.
It unlocks on its own after `hardening_faillock_unlock_time` (15 minutes). To
unlock it now, as root or another admin:

```sh
faillock --user alice            # show recorded failures
faillock --user alice --reset    # unlock
```

root is never locked by faillock.

## su: Permission denied

`su` is limited to members of `hardening_su_group` (`sudo` by default). Add the
user to the group (`usermod -aG sudo alice`), use `sudo -i` instead, or set
`hardening_restrict_su: false`.

## A new password is rejected

Passwords need at least 14 characters from 3 character classes, can't repeat
a character more than 3 times in a row, and can't be dictionary words or
contain the username. This applies to root too. Check a candidate with:

```sh
echo 'candidate password' | pwscore
```

## Container or VPN networking stopped working

IP forwarding was turned off. Set `hardening_sysctl_ip_forwarding: true`,
re-run the role, then put forwarding back (or reboot):

```sh
sysctl -w net.ipv4.ip_forward=1
systemctl restart docker      # if Docker is installed
```

## A program fails with errors about /dev/shm

Some software runs code from shared memory and breaks with `noexec` on
`/dev/shm`. Set `hardening_dev_shm_noexec: false`, remove the `/dev/shm` line
from `/etc/fstab`, and run `mount -o remount,exec /dev/shm`.

## Audit rules don't change

If `hardening_auditd_immutable` was ever enabled, the kernel has locked the
rules until the next reboot. `auditctl -s` shows `enabled 2` when that's the
case. Reboot to load the new rules.

## Check mode fails on a new host

`--check` doesn't install anything, so on a host the role has never run on it
stops early:

- *python3-apt must be installed to use check mode*: Ansible's apt module needs
  `python3-apt` for check mode. A real run installs it automatically. Install
  it yourself (`apt-get install python3-apt`) to get further.
- After that, tasks that use a tool the real run would install first, such as
  `ufw`, fail because the tool isn't there yet.

Run the role for real on a test host first. From then on, check mode works on
that host and shows only the differences.
