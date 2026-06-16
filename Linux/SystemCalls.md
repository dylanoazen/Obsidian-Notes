# System Calls

System calls (syscalls) são a interface entre o user space e o kernel space. Toda vez que seu programa precisa que o kernel faça algo — ler um arquivo, alocar memória, enviar um pacote — ele faz uma syscall.

Related: [[Linux/Kernel]]

---

## Como uma Syscall Funciona

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

O context switch (user → kernel → user) custa ~100-1000ns. Por isso minimizar syscalls importa para a performance.

## Syscalls Comuns

| Syscall | Propósito |
|---------|-----------|
| `read` / `write` | I/O em file descriptors |
| `open` / `close` | Abrir / fechar arquivos |
| `fork` | Criar processo filho |
| `execve` | Executar um programa |
| `mmap` / `munmap` | Mapear memória |
| `brk` | Expandir o heap |
| `socket` / `bind` / `listen` / `accept` | Redes |
| `clone` | Criar thread ou processo (usado internamente pelo fork) |
| `ioctl` | Operações específicas de dispositivo |
| `epoll_create` / `epoll_wait` | I/O multiplexing (event loops) |

## Rastreando Syscalls

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

Como servidores lidam com milhares de conexões sem milhares de threads:

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

#### Meus comentários
- 
