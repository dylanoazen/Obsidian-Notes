# File System

Como o Linux organiza, armazena e acessa arquivos — do VFS aos blocos de disco.

Related: [[Linux/Kernel]]

---

## Tudo é um Arquivo

No Linux, quase tudo é representado como um arquivo:

- Arquivos regulares e diretórios
- Dispositivos (`/dev/sda`, `/dev/tty`)
- Processos (`/proc/<pid>/`)
- Sockets, pipes, symlinks

Essa abstração permite usar a mesma interface `open/read/write/close` para tudo.

## VFS — Virtual File System

O VFS é a camada de abstração do kernel que fica acima de todas as implementações de filesystem:

```
User space:  open(), read(), write()
                    │
                    ▼
Kernel:      ┌─── VFS (Virtual File System) ───┐
             │  inodes, dentries, superblocks   │
             └──┬──────┬──────┬──────┬─────────┘
                │      │      │      │
              ext4   btrfs   xfs   tmpfs   procfs
                │      │      │
              Block device layer
                │
              Disk / SSD
```

- **Superblock**: metadados sobre o filesystem (tamanho, contagem de blocos, status)
- **Inode**: metadados sobre um arquivo (permissões, tamanho, timestamps, ponteiros de bloco — NÃO o nome)
- **Dentry**: entrada de diretório — mapeia um nome de arquivo para um inode (cacheado em memória)

```bash
# View inode info
stat file.txt
ls -i file.txt    # show inode number

# View filesystem info
df -hT            # disk usage with filesystem types
lsblk             # block devices
```

## Filesystems Comuns

| FS | Caso de Uso | Características |
|----|-------------|-----------------|
| ext4 | Padrão para a maioria das distros | Journaling, estável, bem testado |
| btrfs | Moderno, recursos avançados | Snapshots, compressão, RAID |
| xfs | Alta performance, arquivos grandes | Escalável, usado pelo RHEL |
| tmpfs | Dados temporários em RAM | Rápido, volátil |
| procfs | Informações de processos (`/proc`) | Virtual, gerado pelo kernel |
| sysfs | Informações de hardware (`/sys`) | Virtual, gerado pelo kernel |

## Journaling

Protege contra corrupção de dados em caso de crash:

- Antes de escrever dados, grava uma **entrada de journal** descrevendo a mudança
- Se o sistema travar no meio de uma escrita, o journal é reexecutado no boot
- O ext4 faz journal de metadados por padrão; pode fazer journal dos dados também (`data=journal`)

## Mount

Anexar um filesystem à árvore de diretórios:

```bash
# Mount a disk
mount /dev/sdb1 /mnt/data

# View current mounts
mount | column -t
cat /proc/mounts

# Persistent mounts — /etc/fstab
# <device>     <mountpoint>  <type>  <options>       <dump> <pass>
/dev/sdb1      /mnt/data     ext4    defaults,noatime  0      2

# Unmount
umount /mnt/data
```

## File Descriptors

Todo arquivo aberto recebe um **file descriptor** (inteiro):

- `0` — stdin
- `1` — stdout
- `2` — stderr
- `3+` — qualquer outro arquivo aberto

```bash
# See open file descriptors of a process
ls -la /proc/<pid>/fd/

# System-wide limit
cat /proc/sys/fs/file-max

# Per-process limit
ulimit -n
```

## Related

- [[Linux/Kernel]]
- [[Linux/MemoryManagement]]
- [[Linux/SystemCalls]]

#### Meus comentários
- 
