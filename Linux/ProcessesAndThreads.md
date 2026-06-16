# Processos e Threads

Deep dive em como o sistema operacional gerencia execução concorrente — do processo pesado à goroutine.

Related: [[Linux/ProcessManagement]], [[Linux/Kernel]], [[GO/GO]]

---

## Processo vs Thread

```
Processo:                          Thread:
┌──────────────────────┐           ┌──────────────────────┐
│ Address space (own)  │           │ Address space (SHARED)│
│ ┌──────┐ ┌────────┐ │           │ ┌──────┐ ┌────────┐ │
│ │ Heap │ │ Stack  │ │           │ │ Heap │ │Stack T1│ │
│ └──────┘ └────────┘ │           │ │(shared)│ │Stack T2│ │
│ File descriptors     │           │ └──────┘ └────────┘ │
│ PID, UID, signals    │           │ Shared FDs, PID      │
│ Memory mappings      │           │ Each has own: stack,  │
└──────────────────────┘           │ registers, PC, TLS   │
                                   └──────────────────────┘
```

| | Processo | Thread |
|--|---------|--------|
| Memória | Isolada (address space próprio) | Compartilhada (mesmo heap) |
| Criação | Pesada (~ms, copia page tables) | Leve (~μs, só aloca stack) |
| Comunicação | IPC: pipes, sockets, shared mem | Direto via memória compartilhada |
| Crash | Um processo morre, outros vivem | Uma thread crasha, mata o processo |
| Context switch | Caro (troca page tables, TLB flush) | Barato (mesmo address space) |

## Como o Kernel Vê

No Linux, threads e processos são a mesma coisa internamente — ambos são **tasks** (`task_struct`). A diferença é o que compartilham:

```c
// fork() — cria processo (nada compartilhado)
clone(SIGCHLD)

// pthread_create() — cria thread (compartilha quase tudo)
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD)
```

Flags do `clone()`:
- `CLONE_VM` — compartilha address space
- `CLONE_FILES` — compartilha file descriptors
- `CLONE_FS` — compartilha filesystem info
- `CLONE_SIGHAND` — compartilha signal handlers
- `CLONE_THREAD` — mesmo thread group (mesmo PID visível)

## Context Switch

O que acontece quando o scheduler troca de task:

```
Task A rodando
    │
    ▼ Timer interrupt / syscall / yield
Save state:
    ├── Registradores → task_struct de A
    ├── Stack pointer → task_struct de A
    └── Program counter → task_struct de A
    │
    ▼ Scheduler picks Task B
Restore state:
    ├── Registradores ← task_struct de B
    ├── Stack pointer ← task_struct de B
    └── Program counter ← task_struct de B
    │
    ▼ Se B é outro processo: flush TLB, troca page tables
    ▼ Se B é outra thread do mesmo processo: só troca registradores
    │
Task B rodando
```

Thread switch: ~1-2μs. Process switch: ~3-5μs (TLB flush é caro).

## Threading Models

### 1:1 (Kernel-level threads)

```
User Thread 1 ──► Kernel Thread 1 ──► CPU
User Thread 2 ──► Kernel Thread 2 ──► CPU
User Thread 3 ──► Kernel Thread 3 ──► CPU
```

- Cada user thread = um kernel thread
- Usado por: Linux pthreads, Java (moderno), Rust
- Pro: paralelismo real, escalonamento preemptivo
- Con: caro criar muitas (stack ~2-8MB cada), context switch via kernel

### N:1 (User-level threads / Green threads)

```
User Thread 1 ┐
User Thread 2 ├──► 1 Kernel Thread ──► CPU
User Thread 3 ┘
```

- Runtime gerencia scheduling em user space
- Pro: criação ultra rápida, context switch sem kernel
- Con: uma thread bloqueia = todas bloqueiam, sem paralelismo real

### M:N (Hybrid)

```
User Thread 1 ┐         ┌──► Kernel Thread 1 ──► CPU core 1
User Thread 2 ├── M:N ──┤
User Thread 3 ├── map ──┤
User Thread 4 ┘         └──► Kernel Thread 2 ──► CPU core 2
```

- M user threads mapeadas em N kernel threads
- Usado por: **Go (goroutines)**, Erlang, Elixir
- Pro: leve + paralelismo real + escalável (milhões de goroutines)
- Con: runtime mais complexo

## Go Goroutines — M:N na Prática

```
┌──────────── Go Runtime Scheduler ───────────┐
│                                              │
│  G1 G2 G3 G4 ... Gn    (goroutines, ~2KB)  │
│       │  │  │                                │
│       ▼  ▼  ▼                                │
│      P1    P2           (logical processors) │
│       │     │            GOMAXPROCS = 2      │
│       ▼     ▼                                │
│      M1    M2           (OS threads)         │
│       │     │                                │
└───────┼─────┼────────────────────────────────┘
        ▼     ▼
     CPU 1  CPU 2
```

- **G** (Goroutine): tarefa leve, stack de ~2KB (cresce dinamicamente)
- **P** (Processor): processador lógico, limitado por GOMAXPROCS
- **M** (Machine): OS thread

```go
// Criar goroutine — custo: ~2KB
go func() {
    fmt.Println("running concurrently")
}()

// vs OS thread — custo: ~2-8MB
// Pode ter milhões de goroutines, mas só centenas de OS threads
```

Quando uma goroutine bloqueia (I/O, syscall), o runtime move outras goroutines pra outro M thread. O scheduler é **cooperativo** com pontos de preempção inseridos pelo compilador.

## POSIX Threads (pthreads)

A API padrão de threading em C/C++ no Linux:

```c
#include <pthread.h>

void* worker(void* arg) {
    int id = *(int*)arg;
    printf("Thread %d running\n", id);
    return NULL;
}

int main() {
    pthread_t threads[4];
    int ids[4];
    
    for (int i = 0; i < 4; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }
    
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);  // wait for completion
    }
}
```

```bash
# Compile
gcc -pthread program.c -o program

# View threads of a process
ps -T -p <pid>
ls /proc/<pid>/task/
```

## Sincronização

Quando threads compartilham memória, precisam coordenar acesso:

### Mutex (Mutual Exclusion)

```
Thread A          Mutex          Thread B
   │                │                │
   ├── Lock() ─────►│                │
   │   (acquired)   │◄── Lock() ────┤
   │                │   (BLOCKED)    │
   ├── critical     │                │
   │   section      │                │
   ├── Unlock() ───►│                │
   │                ├── (acquired) ──┤
   │                │   critical     │
   │                │   section      │
   │                │◄── Unlock() ──┤
```

### Semaphore

- Mutex generalizado — permite N acessos simultâneos
- `sem_wait()` decrementa, bloqueia se 0
- `sem_post()` incrementa, acorda waiters

### Condition Variable

```
Thread A (producer):              Thread B (consumer):
  lock(mutex)                       lock(mutex)
  add_item(queue)                   while queue_empty:
  signal(cond)    ──────────►         wait(cond, mutex)  // releases mutex, sleeps
  unlock(mutex)                     item = get(queue)
                                    unlock(mutex)
```

### Read-Write Lock

- Múltiplos readers simultâneos OK
- Writer precisa de acesso exclusivo
- Bom quando reads >> writes

### Problemas Clássicos

- **Deadlock**: A espera B que espera A
- **Livelock**: threads ficam mudando de estado sem progredir
- **Starvation**: uma thread nunca consegue acesso
- **Race condition**: resultado depende da ordem de execução
- **Priority inversion**: thread de alta prioridade espera por thread de baixa

## IPC — Inter-Process Communication

Como processos (address spaces separados) se comunicam:

| Mecanismo | Velocidade | Uso típico |
|-----------|-----------|------------|
| Pipe | Rápido | Parent → child (unidirecional) |
| Named pipe (FIFO) | Rápido | Processos não-relacionados |
| Unix socket | Rápido | Local client-server |
| TCP socket | Médio | Network communication |
| Shared memory | Mais rápido | Alto throughput, precisa sync |
| Message queue | Médio | Structured messages |
| Signal | Rápido | Notifications (limited data) |
| mmap | Rápido | File-backed shared memory |

```bash
# Pipe
ls | grep ".md"    # stdout de ls → stdin de grep

# Named pipe
mkfifo /tmp/myfifo
echo "hello" > /tmp/myfifo &
cat /tmp/myfifo

# Shared memory
# /dev/shm/ — tmpfs mounted, used for POSIX shared memory
ls /dev/shm/
```

## Concorrência vs Paralelismo

```
Concorrência:                    Paralelismo:
(lidar com várias coisas)        (fazer várias coisas ao mesmo tempo)

  Task A ──►   ──►               Task A ────────►
  Task B    ──►   ──►            Task B ────────►
        (1 CPU, alternando)          (2 CPUs, simultâneo)
```

- Concorrência é sobre **estrutura** (design)
- Paralelismo é sobre **execução** (hardware)
- Você pode ter concorrência sem paralelismo (1 core, multitask)
- Go: concorrência fácil com goroutines, paralelismo via GOMAXPROCS

## Observando Threads no Linux

```bash
# Threads de um processo
ps -eLf | grep myapp
top -H -p <pid>              # thread view in top
htop                          # press H to toggle threads

# Thread count
ls /proc/<pid>/task/ | wc -l

# Stack trace de todas as threads
cat /proc/<pid>/task/*/stack

# strace com threads
strace -f -p <pid>            # -f follows child threads
```

## Related

- [[Linux/ProcessManagement]]
- [[Linux/Kernel]]
- [[Linux/MemoryManagement]]
- [[Linux/SystemCalls]]
- [[GO/GarbageCollector]]
- [[DistributedSystems/Concurrency]]

## Recursos

- https://pages.cs.wisc.edu/~remzi/OSTEP/ (Operating Systems: Three Easy Pieces — gratuito)
- Linux Programming Interface — Michael Kerrisk (livro)
- https://go.dev/blog/waza-talk (padrões de concorrência em Go)

#### Meus comentários
- 
