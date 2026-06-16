# Memory Management

Como o Linux gerencia memória — memória virtual, paginação, alocação e swap.

Related: [[Linux/Kernel]], [[GO/MemoryManagement|Go Memory Management]]

---

## Memória Virtual

Todo processo acredita ter o address space inteiro para si mesmo. O kernel + MMU (Memory Management Unit) traduzem endereços virtuais para endereços físicos.

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

Benefícios:
- **Isolamento**: processos não conseguem ler a memória uns dos outros
- **Overcommit**: a memória virtual total pode exceder a RAM física
- **Bibliotecas compartilhadas**: a libc é carregada uma vez na RAM e mapeada em todos os processos

## Paginação

A memória é dividida em **pages** de tamanho fixo (tipicamente 4KB no x86):

- **Page table**: mapeamento por processo de páginas virtuais → físicas
- **TLB** (Translation Lookaside Buffer): cache da CPU para traduções recentes
- **Page fault**: acesso a uma página que não está na RAM → o kernel a carrega do disco ou aloca

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

- Pages padrão de 4KB geram page tables grandes para processos que usam muita memória
- **Huge Pages**: pages de 2MB ou 1GB — menos TLB misses, melhor desempenho para cargas de trabalho grandes
- **Transparent Huge Pages (THP)**: o kernel promove automaticamente pages de 4KB para 2MB quando benéfico

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Check huge page usage
grep Huge /proc/meminfo
```

## Alocação de Memória

O user space solicita memória via:
- `malloc()` → alocador da biblioteca C (glibc usa ptmalloc2)
- `brk()`/`sbrk()` → expande o heap (alocações pequenas)
- `mmap()` → mapeia uma nova região de memória (alocações grandes, mapeamento de arquivo)

O kernel gerencia páginas físicas via o **buddy allocator** (blocos de páginas em potências de 2) e o **slab allocator** (alocação eficiente de objetos de tamanho fixo do kernel).

## Swap

Quando a RAM física está cheia, o kernel move páginas inativas para o disco:

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

Quando RAM + swap estão esgotados, o **Out-Of-Memory Killer** do kernel escolhe um processo para encerrar:

- Atribui uma pontuação a cada processo com base no uso de memória (pontuação mais alta = mais chance de ser encerrado)
- Verificar a pontuação OOM de um processo: `cat /proc/<pid>/oom_score`
- Proteger um processo: `echo -1000 > /proc/<pid>/oom_score_adj`

```bash
# View OOM logs
dmesg | grep -i oom
journalctl | grep -i "out of memory"
```

## Related

- [[Linux/ProcessManagement]]
- [[Linux/Kernel]]
- [[Linux/FileSystem]]

#### Meus comentários
- 
