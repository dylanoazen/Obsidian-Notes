# Linux Security

Hardening, controle de acesso e mecanismos de segurança integrados ao kernel Linux e ao userspace.

Related: [[Linux/Kernel]], [[Linux/ProcessManagement]]

---

## Modelo de Permissões

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

## Permissões Especiais

- **SUID** (4xxx): executa como o dono do arquivo (ex.: `passwd` executa como root)
- **SGID** (2xxx): executa como o grupo do arquivo / herda o grupo em diretórios
- **Sticky bit** (1xxx): apenas o dono pode deletar arquivos no diretório (`/tmp`)

```bash
chmod u+s binary          # SUID
chmod g+s directory       # SGID
chmod +t directory        # sticky bit
find / -perm -4000        # find all SUID binaries (audit this!)
```

## Capabilities

Em vez de conceder acesso root completo a um processo, conceda capabilities específicas:

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

Capabilities principais:
- `CAP_NET_BIND_SERVICE` — fazer bind em portas < 1024
- `CAP_NET_RAW` — raw sockets (ping)
- `CAP_SYS_ADMIN` — admin genérico (perigoso)
- `CAP_DAC_OVERRIDE` — ignorar verificações de permissão de arquivo

## SELinux / AppArmor

Mandatory Access Control (MAC) — aplica políticas mesmo para root:

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

## Namespaces para Isolamento

Base da segurança de containers ([[DevOps/Docker/Docker]]):

| Namespace | Isola |
|-----------|-------|
| PID | IDs de processo |
| NET | Interfaces de rede, IPs, rotas |
| MNT | Pontos de montagem do filesystem |
| USER | Mapeamentos de UID/GID |
| UTS | Hostname |
| IPC | Comunicação entre processos |
| cgroup | Raiz do cgroup |

## seccomp

Restringe quais syscalls um processo pode fazer:

```bash
# Docker uses seccomp profiles to block dangerous syscalls
docker run --security-opt seccomp=profile.json myapp
```

O perfil seccomp padrão do Docker bloqueia ~44 syscalls, incluindo `reboot`, `mount` e `kexec_load`.

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

## Hardening do SSH

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no        # keys only
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers deploy admin
Protocol 2
```

## Auditoria

```bash
# auditd — kernel-level auditing
auditctl -w /etc/passwd -p wa -k passwd_changes
ausearch -k passwd_changes

# Check login history
last
lastb                            # failed logins
journalctl -u sshd --since today
```

## Checklist de Hardening

- Desabilitar SSH como root, usar somente autenticação por chave
- Manter pacotes atualizados (`unattended-upgrades`)
- Remover binários SUID desnecessários
- Habilitar firewall, permitir apenas as portas necessárias
- Habilitar SELinux/AppArmor
- Configurar fail2ban para proteção contra brute-force
- Auditar arquivos SUID, portas abertas e serviços em execução regularmente
- Usar capabilities em vez de root sempre que possível

## Related

- [[Linux/Kernel]]
- [[Linux/SystemCalls]]
- [[Security/IAM]]

## Recursos

- https://www.cisecurity.org/benchmark/ubuntu_linux (CIS Benchmarks)
- https://madaidans-insecurities.github.io/guides/linux-hardening.html

#### Meus comentários
- 
