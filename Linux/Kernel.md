# Kernel

The kernel is the core of the operating system. It's the layer between hardware and userspace programs. Every system call, every file read, every packet sent goes through the kernel.

Related: [[Linux/Linux|Linux Internals]]

---

## What the Kernel Does

- **Process management**: create, schedule, and kill processes
- **Memory management**: virtual memory, paging, allocation
- **Device management**: drivers, hardware abstraction
- **File systems**: VFS layer, ext4, btrfs, tmpfs
- **Networking**: TCP/IP stack, sockets, routing
- **Security**: permissions, capabilities, namespaces, cgroups

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

- **User space**: restricted access, runs in CPU Ring 3
- **Kernel space**: full hardware access, runs in CPU Ring 0
- Transition between them happens via **system calls** (expensive — context switch)

## Kernel Architecture

Linux is a **monolithic kernel** with loadable modules:

- **Monolithic**: entire kernel runs in one address space (fast, no IPC overhead)
- **Modules**: drivers and features can be loaded/unloaded at runtime without reboot
- Contrast with **microkernel** (Minix, QNX): only minimal code in kernel space, everything else in userspace via IPC

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

## Kernel Compilation

You can build your own kernel — useful for understanding or embedded systems:

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

## /proc and /sys — Kernel Interfaces

The kernel exposes runtime info via virtual filesystems:

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

## Kernel Rings (CPU Protection)

```
Ring 0: Kernel         — full access to hardware
Ring 1: (unused in Linux)
Ring 2: (unused in Linux)
Ring 3: User space     — restricted, must ask kernel via syscall
```

Modern x86 only uses Ring 0 and Ring 3. ARM uses EL0 (user) and EL1 (kernel).

## Related

- [[Linux/ProcessManagement]]
- [[Linux/MemoryManagement]]
- [[Linux/SystemCalls]]

## Resources

- https://www.kernel.org (official source)
- https://0xax.gitbooks.io/linux-insides/content/

#### My commentaries
- 
