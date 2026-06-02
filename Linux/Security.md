# Linux Security

Hardening, access control, and security mechanisms built into the Linux kernel and userspace.

Related: [[Linux/Kernel]], [[Linux/ProcessManagement]]

---

## Permissions Model

```
-rwxr-x--- 1 user group 4096 Jan 1 file.txt
│└┬┘└┬┘└┬┘
│ │   │   └── others: no access
│ │   └────── group: read + execute
│ └────────── owner: read + write + execute
└──────────── type: - file, d directory, l symlink
```

```bash
chmod 750 file.txt        # rwxr-x---
chown user:group file.txt
umask 027                 # default permissions for new files
```

## Special Permissions

- **SUID** (4xxx): run as file owner (e.g., `passwd` runs as root)
- **SGID** (2xxx): run as file group / inherit group on directories
- **Sticky bit** (1xxx): only owner can delete files in directory (`/tmp`)

```bash
chmod u+s binary          # SUID
chmod g+s directory       # SGID
chmod +t directory        # sticky bit
find / -perm -4000        # find all SUID binaries (audit this!)
```

## Capabilities

Instead of giving a process full root access, give it specific capabilities:

```bash
# View capabilities of a binary
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep

# Set capability
setcap cap_net_bind_service=+ep ./myserver
# now it can bind to port 80 without root

# List all capabilities
capsh --print
```

Key capabilities:
- `CAP_NET_BIND_SERVICE` — bind to ports < 1024
- `CAP_NET_RAW` — raw sockets (ping)
- `CAP_SYS_ADMIN` — catch-all admin (dangerous)
- `CAP_DAC_OVERRIDE` — bypass file permission checks

## SELinux / AppArmor

Mandatory Access Control (MAC) — enforces policies even for root:

**SELinux** (Red Hat, Fedora):
```bash
getenforce                    # Enforcing, Permissive, Disabled
setenforce 0                  # switch to permissive (temporary)
ls -Z file.txt                # view SELinux context
ausearch -m avc               # view denied actions
```

**AppArmor** (Ubuntu, SUSE):
```bash
aa-status                     # view loaded profiles
aa-enforce /etc/apparmor.d/profile
aa-complain /etc/apparmor.d/profile  # log only, don't block
```

## Namespaces for Isolation

Foundation of container security ([[DevOps/Docker/Docker]]):

| Namespace | Isolates |
|-----------|----------|
| PID | Process IDs |
| NET | Network interfaces, IPs, routes |
| MNT | Filesystem mount points |
| USER | UID/GID mappings |
| UTS | Hostname |
| IPC | Inter-process communication |
| cgroup | cgroup root |

## seccomp

Restrict which syscalls a process can make:

```bash
# Docker uses seccomp profiles to block dangerous syscalls
docker run --security-opt seccomp=profile.json myapp
```

Default Docker seccomp blocks ~44 syscalls including `reboot`, `mount`, `kexec_load`.

## Firewall

```bash
# iptables (legacy)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP

# nftables (modern)
nft add rule inet filter input tcp dport 22 ip saddr 10.0.0.0/8 accept

# ufw (Ubuntu simplified)
ufw allow from 10.0.0.0/8 to any port 22
ufw enable
```

## SSH Hardening

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no        # keys only
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers deploy admin
Protocol 2
```

## Audit

```bash
# auditd — kernel-level auditing
auditctl -w /etc/passwd -p wa -k passwd_changes
ausearch -k passwd_changes

# Check login history
last
lastb                            # failed logins
journalctl -u sshd --since today
```

## Hardening Checklist

- Disable root SSH, use key-based auth only
- Keep packages updated (`unattended-upgrades`)
- Remove unnecessary SUID binaries
- Enable firewall, allow only needed ports
- Enable SELinux/AppArmor
- Configure fail2ban for brute-force protection
- Audit SUID files, open ports, running services regularly
- Use capabilities instead of root where possible

## Related

- [[Linux/Kernel]]
- [[Linux/SystemCalls]]
- [[Security/IAM]]

## Resources

- https://www.cisecurity.org/benchmark/ubuntu_linux (CIS Benchmarks)
- https://madaidans-insecurities.github.io/guides/linux-hardening.html

#### My commentaries
- 
