# Process Management

How Linux creates, schedules, and manages processes and threads.

Related: [[Linux/Kernel]]

---

## What is a Process?

A process is a running instance of a program. Each process has:

- **PID**: unique process ID
- **Address space**: its own virtual memory (isolated from others)
- **File descriptors**: open files, sockets, pipes
- **Credentials**: UID, GID, capabilities
- **State**: running, sleeping, stopped, zombie

```bash
# View processes
ps aux
top
htop

# Process tree
pstree -p
```

## Process Creation — fork() and exec()

Linux creates new processes in two steps:

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

- `fork()` is cheap thanks to **copy-on-write** (COW): memory pages are shared until one process modifies them
- `exec()` loads a new binary into the process address space
- PID 1 (`init`/`systemd`) is the ancestor of all processes

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

## Process States

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

## The Scheduler — CFS

Linux uses the **Completely Fair Scheduler** (CFS):

- Goal: give every process a fair share of CPU time
- Uses a **red-black tree** ordered by "virtual runtime" (vruntime)
- Process with lowest vruntime runs next
- Nice values (-20 to +19) adjust priority: lower = more CPU time

```bash
# Run with modified priority
nice -n 10 ./my_script.sh

# Change priority of running process
renice -n 5 -p 1234
```

## Threads vs Processes

- **Thread**: lightweight, shares memory with other threads in same process
- **Process**: heavyweight, isolated memory via fork()
- In Linux, threads are actually processes that share address space (created via `clone()`)
- Each thread has its own stack but shares heap, file descriptors, etc.

## Signals

Signals are software interrupts sent to processes:

```bash
kill -SIGTERM 1234    # graceful shutdown request
kill -SIGKILL 1234    # force kill (cannot be caught)
kill -SIGHUP 1234     # hangup — often used to reload config
kill -SIGSTOP 1234    # pause process
kill -SIGCONT 1234    # resume process
```

| Signal | Number | Default Action | Catchable? |
|--------|--------|---------------|------------|
| SIGTERM | 15 | Terminate | Yes |
| SIGKILL | 9 | Kill | No |
| SIGINT | 2 | Interrupt (Ctrl+C) | Yes |
| SIGSEGV | 11 | Segfault | Yes |
| SIGCHLD | 17 | Child stopped/exited | Yes |

## Namespaces and cgroups

Foundation of containers ([[DevOps/Docker/Docker]]):

- **Namespaces**: isolate what a process can see (PID, network, mount, user, etc.)
- **cgroups**: limit what a process can use (CPU, memory, I/O bandwidth)

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

#### My commentaries
- 
