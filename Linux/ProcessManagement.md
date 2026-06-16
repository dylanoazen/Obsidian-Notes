# Process Management

Como o Linux cria, escalonai e gerencia processos e threads.

Related: [[Linux/Kernel]]

---

## O que é um Processo?

Um processo é uma instância em execução de um programa. Cada processo possui:

- **PID**: identificador único do processo
- **Address space**: sua própria memória virtual (isolada dos demais)
- **File descriptors**: arquivos abertos, sockets, pipes
- **Credentials**: UID, GID, capabilities
- **Estado**: running, sleeping, stopped, zombie

```bash
# View processes
ps aux
top
htop

# Process tree
pstree -p
```

## Criação de Processos — fork() e exec()

O Linux cria novos processos em duas etapas:

```
Parent process
    │
    ├── fork()  →  creates a copy of the parent (child process)
    │                same code, same memory (copy-on-write)
    │
    └── Child process
         │
         └── exec()  →  replaces the child's memory with a new program
```

- `fork()` é barato graças ao **copy-on-write** (COW): as páginas de memória são compartilhadas até que um processo as modifique
- `exec()` carrega um novo binário no address space do processo
- O PID 1 (`init`/`systemd`) é o ancestral de todos os processos

```c
pid_t pid = fork();
if (pid == 0) {
    // child process
    execvp("ls", args);  // replace with 'ls'
} else {
    // parent process
    wait(NULL);  // wait for child
}
```

## Estados de Processo

```
         ┌──── Running (R) ◄───── Scheduled by CPU
         │
New ────►├──── Sleeping (S) ◄──── Waiting for I/O or event
         │       │
         │       ├── Interruptible (S) — can be woken by signals
         │       └── Uninterruptible (D) — waiting for disk/hardware
         │
         ├──── Stopped (T) ◄────── SIGSTOP / SIGTSTP (Ctrl+Z)
         │
         └──── Zombie (Z) ◄────── Finished but parent hasn't called wait()
```

## O Scheduler — CFS

O Linux usa o **Completely Fair Scheduler** (CFS):

- Objetivo: dar a cada processo uma fatia justa do tempo de CPU
- Usa uma **red-black tree** ordenada por "virtual runtime" (vruntime)
- O processo com o menor vruntime executa em seguida
- Valores de nice (-20 a +19) ajustam a prioridade: menor = mais tempo de CPU

```bash
# Run with modified priority
nice -n 10 ./my_script.sh

# Change priority of running process
renice -n 5 -p 1234
```

## Threads vs Processos

- **Thread**: leve, compartilha memória com outras threads do mesmo processo
- **Processo**: pesado, memória isolada via fork()
- No Linux, threads são na verdade processos que compartilham address space (criados via `clone()`)
- Cada thread tem sua própria stack mas compartilha heap, file descriptors, etc.

## Signals

Signals são interrupções de software enviadas a processos:

```bash
kill -SIGTERM 1234    # graceful shutdown request
kill -SIGKILL 1234    # force kill (cannot be caught)
kill -SIGHUP 1234     # hangup — often used to reload config
kill -SIGSTOP 1234    # pause process
kill -SIGCONT 1234    # resume process
```

| Signal | Número | Ação Padrão | Interceptável? |
|--------|--------|-------------|----------------|
| SIGTERM | 15 | Encerrar | Sim |
| SIGKILL | 9 | Matar | Não |
| SIGINT | 2 | Interromper (Ctrl+C) | Sim |
| SIGSEGV | 11 | Segfault | Sim |
| SIGCHLD | 17 | Filho parou/encerrou | Sim |

## Namespaces e cgroups

Fundação de containers ([[DevOps/Docker/Docker]]):

- **Namespaces**: isolam o que um processo pode ver (PID, rede, mount, usuário, etc.)
- **cgroups**: limitam o que um processo pode usar (CPU, memória, largura de banda de I/O)

```bash
# View namespaces of a process
ls -la /proc/1/ns/

# View cgroup of a process
cat /proc/1/cgroup
```

## Related

- [[Linux/MemoryManagement]]
- [[Linux/SystemCalls]]
- [[Linux/Kernel]]

#### Meus comentários
- 
