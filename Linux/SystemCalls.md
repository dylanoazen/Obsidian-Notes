# System Calls

System calls (syscalls) are the interface between user space and kernel space. Every time your program needs the kernel to do something — read a file, allocate memory, send a packet — it makes a syscall.

Related: [[Linux/Kernel]]

---

## How a Syscall Works

```
User space program
    │
    ├── calls libc wrapper: read(fd, buf, size)
    │
    ├── libc puts syscall number in register (e.g., RAX = 0 for read on x86_64)
    │
    ├── executes 'syscall' instruction → CPU switches to Ring 0
    │
    ├── Kernel: looks up handler in syscall table
    │           executes sys_read()
    │           copies result back to user space
    │
    └── CPU returns to Ring 3 → program continues
```

The context switch (user → kernel → user) costs ~100-1000ns. That's why minimizing syscalls matters for performance.

## Common Syscalls

| Syscall | Purpose |
|---------|---------|
| `read` / `write` | I/O on file descriptors |
| `open` / `close` | Open / close files |
| `fork` | Create child process |
| `execve` | Execute a program |
| `mmap` / `munmap` | Map memory |
| `brk` | Expand heap |
| `socket` / `bind` / `listen` / `accept` | Networking |
| `clone` | Create thread or process (used by fork internally) |
| `ioctl` | Device-specific operations |
| `epoll_create` / `epoll_wait` | I/O multiplexing (event loops) |

## Tracing Syscalls

```bash
# Trace all syscalls of a command
strace ls -la

# Trace a running process
strace -p <pid>

# Count syscalls (summary)
strace -c ls -la

# Filter specific syscalls
strace -e trace=open,read,write ls

# With timestamps
strace -t -e trace=network curl example.com
```

## I/O Multiplexing

How servers handle thousands of connections without thousands of threads:

```
select()     → oldest, limited to 1024 fds
poll()       → no fd limit, but scans all fds every time
epoll()      → Linux-specific, O(1) for ready events, scales to millions
              → used by nginx, Redis, Node.js, Go runtime
```

```
epoll workflow:
  epoll_create()  → create epoll instance
  epoll_ctl()     → add/remove file descriptors to watch
  epoll_wait()    → block until events are ready (efficient)
```

## Related

- [[Linux/Kernel]]
- [[Linux/ProcessManagement]]
- [[Linux/Networking]]

#### My commentaries
- 
