# Memory Management

How Linux manages memory — virtual memory, paging, allocation, and swap.

Related: [[Linux/Kernel]], [[GO/MemoryManagement|Go Memory Management]]

---

## Virtual Memory

Every process thinks it has the entire address space to itself. The kernel + MMU (Memory Management Unit) translate virtual addresses to physical addresses.

```
Process A sees:          Physical RAM:
┌──────────────┐         ┌──────────────┐
│ 0x0000 Stack │────────►│ Frame 42     │
│ 0x1000 Heap  │────────►│ Frame 107    │
│ 0x2000 Code  │────────►│ Frame 3      │
└──────────────┘         └──────────────┘

Process B sees:          Same Physical RAM:
┌──────────────┐         ┌──────────────┐
│ 0x0000 Stack │────────►│ Frame 88     │
│ 0x1000 Heap  │────────►│ Frame 12     │
│ 0x2000 Code  │────────►│ Frame 3      │ ← shared! (same libc)
└──────────────┘         └──────────────┘
```

Benefits:
- **Isolation**: processes can't read each other's memory
- **Overcommit**: total virtual memory can exceed physical RAM
- **Shared libraries**: libc loaded once in RAM, mapped into every process

## Paging

Memory is divided into fixed-size **pages** (typically 4KB on x86):

- **Page table**: per-process mapping of virtual → physical pages
- **TLB** (Translation Lookaside Buffer): CPU cache for recent translations
- **Page fault**: access to a page not in RAM → kernel loads it from disk or allocates

```bash
# Page size
getconf PAGE_SIZE    # usually 4096

# Memory info
cat /proc/meminfo

# Per-process memory map
cat /proc/self/maps
pmap -x <pid>
```

## Huge Pages

- Default 4KB pages mean large page tables for processes with lots of memory
- **Huge Pages**: 2MB or 1GB pages — fewer TLB misses, better performance for large workloads
- **Transparent Huge Pages (THP)**: kernel auto-promotes 4KB pages to 2MB when beneficial

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Check huge page usage
grep Huge /proc/meminfo
```

## Memory Allocation

User space requests memory via:
- `malloc()` → C library allocator (glibc uses ptmalloc2)
- `brk()`/`sbrk()` → expand heap (small allocations)
- `mmap()` → map new memory region (large allocations, file mapping)

The kernel manages physical pages via the **buddy allocator** (powers of 2 page blocks) and **slab allocator** (efficient allocation of same-size kernel objects).

## Swap

When physical RAM is full, the kernel moves inactive pages to disk:

```
RAM full → find least recently used pages → write to swap → free RAM
         → when needed again → page fault → read back from swap
```

```bash
# View swap usage
swapon --show
free -h

# Swappiness: how aggressively to use swap (0-100)
cat /proc/sys/vm/swappiness    # default 60
echo 10 > /proc/sys/vm/swappiness   # prefer keeping in RAM
```

## OOM Killer

When RAM + swap are exhausted, the kernel's **Out-Of-Memory Killer** picks a process to kill:

- Scores each process by memory usage (higher score = more likely to be killed)
- Check a process's OOM score: `cat /proc/<pid>/oom_score`
- Protect a process: `echo -1000 > /proc/<pid>/oom_score_adj`

```bash
# View OOM logs
dmesg | grep -i oom
journalctl | grep -i "out of memory"
```

## Related

- [[Linux/ProcessManagement]]
- [[Linux/Kernel]]
- [[Linux/FileSystem]]

#### My commentaries
- 
