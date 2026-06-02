# File System

How Linux organizes, stores, and accesses files — from VFS to disk blocks.

Related: [[Linux/Kernel]]

---

## Everything is a File

In Linux, almost everything is represented as a file:

- Regular files and directories
- Devices (`/dev/sda`, `/dev/tty`)
- Processes (`/proc/<pid>/`)
- Sockets, pipes, symlinks

This abstraction lets you use the same `open/read/write/close` interface for everything.

## VFS — Virtual File System

VFS is the kernel's abstraction layer that sits above all filesystem implementations:

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

- **Superblock**: metadata about the filesystem (size, block count, status)
- **Inode**: metadata about a file (permissions, size, timestamps, block pointers — NOT the name)
- **Dentry**: directory entry — maps a filename to an inode (cached in memory)

```bash
# View inode info
stat file.txt
ls -i file.txt    # show inode number

# View filesystem info
df -hT            # disk usage with filesystem types
lsblk             # block devices
```

## Common Filesystems

| FS | Use Case | Features |
|----|----------|----------|
| ext4 | Default for most distros | Journaling, stable, well-tested |
| btrfs | Modern, advanced features | Snapshots, compression, RAID |
| xfs | High-performance, large files | Scalable, used by RHEL |
| tmpfs | Temporary data in RAM | Fast, volatile |
| procfs | Process info (`/proc`) | Virtual, kernel-generated |
| sysfs | Hardware info (`/sys`) | Virtual, kernel-generated |

## Journaling

Protects against data corruption on crash:

- Before writing data, write a **journal entry** describing the change
- If system crashes mid-write, replay the journal on boot
- ext4 journals metadata by default; can journal data too (`data=journal`)

## Mount

Attaching a filesystem to the directory tree:

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

Every open file gets a **file descriptor** (integer):

- `0` — stdin
- `1` — stdout
- `2` — stderr
- `3+` — any other open file

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

#### My commentaries
- 
