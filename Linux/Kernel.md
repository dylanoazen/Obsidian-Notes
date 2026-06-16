# Kernel

O kernel é o núcleo do sistema operacional. É a camada entre o hardware e os programas em userspace. Cada system call, cada leitura de arquivo, cada pacote enviado passa pelo kernel.

Related: [[Linux/Linux|Linux Internals]]

---

## O que o Kernel Faz

- **Gerenciamento de processos**: criar, escalonar e encerrar processos
- **Gerenciamento de memória**: memória virtual, paginação, alocação
- **Gerenciamento de dispositivos**: drivers, abstração de hardware
- **Filesystems**: camada VFS, ext4, btrfs, tmpfs
- **Redes**: pilha TCP/IP, sockets, roteamento
- **Segurança**: permissões, capabilities, namespaces, cgroups

## Kernel Space vs User Space

```
┌─────────────────────────────┐
│       User Space            │
│  Applications, Libraries    │
│  (each process in its own   │
│   virtual address space)    │
├─────────────────────────────┤  ← System Call Interface
│       Kernel Space          │
│  Process scheduler          │
│  Memory manager             │
│  Device drivers             │
│  File system layer          │
│  Network stack              │
├─────────────────────────────┤
│       Hardware              │
│  CPU, RAM, Disk, NIC        │
└─────────────────────────────┘
```

- **User space**: acesso restrito, executa no CPU Ring 3
- **Kernel space**: acesso total ao hardware, executa no CPU Ring 0
- A transição entre eles ocorre via **system calls** (custosa — context switch)

## Arquitetura do Kernel

O Linux é um **kernel monolítico** com módulos carregáveis:

- **Monolítico**: todo o kernel executa em um único address space (rápido, sem overhead de IPC)
- **Módulos**: drivers e funcionalidades podem ser carregados/descarregados em tempo de execução sem reinicialização
- Contraste com **microkernel** (Minix, QNX): apenas código mínimo no kernel space, todo o resto em userspace via IPC

```bash
# List loaded modules
lsmod

# Load a module
modprobe module_name

# Info about a module
modinfo module_name

# Kernel version
uname -r
```

## Compilação do Kernel

Você pode compilar seu próprio kernel — útil para entender o funcionamento ou para sistemas embarcados:

```bash
# Get source
git clone https://github.com/torvalds/linux.git
cd linux

# Configure
make menuconfig    # interactive config
# or copy current config:
cp /boot/config-$(uname -r) .config
make olddefconfig

# Build
make -j$(nproc)

# Install
make modules_install
make install
```

## /proc e /sys — Interfaces do Kernel

O kernel expõe informações de runtime via filesystems virtuais:

```bash
# Process info
cat /proc/cpuinfo          # CPU details
cat /proc/meminfo          # memory stats
cat /proc/1/status         # PID 1 info
cat /proc/self/maps        # current process memory map

# Kernel parameters (tunable at runtime)
cat /proc/sys/vm/swappiness
echo 10 > /proc/sys/vm/swappiness   # reduce swap aggressiveness

# Hardware/driver info
ls /sys/class/net/         # network interfaces
ls /sys/block/             # block devices
```

## Rings do Kernel (Proteção da CPU)

```
Ring 0: Kernel         — full access to hardware
Ring 1: (unused in Linux)
Ring 2: (unused in Linux)
Ring 3: User space     — restricted, must ask kernel via syscall
```

O x86 moderno usa apenas Ring 0 e Ring 3. O ARM usa EL0 (user) e EL1 (kernel).

## Related

- [[Linux/ProcessManagement]]
- [[Linux/MemoryManagement]]
- [[Linux/SystemCalls]]

## Recursos

- https://www.kernel.org (fonte oficial)
- https://0xax.gitbooks.io/linux-insides/content/

#### Meus comentários
- 
