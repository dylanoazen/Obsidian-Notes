# Garbage Collector

O Garbage Collector (GC) é a parte do runtime do Go que libera automaticamente a memória heap que não é mais acessível pelo programa.

Related: [[GO/MemoryManagement|Memory Management in Go]]

---

## Por que o GC importa

- Sem GC, você precisa liberar memória manualmente (como em C/C++) — propenso a erros
- O GC troca um pouco de tempo de CPU por segurança: sem dangling pointers, sem double frees
- Entender o GC ajuda a reduzir alocações e evitar picos de latência

## O Algoritmo de GC do Go

Go usa um coletor **concurrent, tri-color, mark-and-sweep**.

### Mark Phase

- Começa a partir dos **root objects** (globals, variáveis de stack, stacks de goroutines)
- Percorre todos os objetos acessíveis, marcando-os como "vivos"
- Roda **concorrentemente** com a aplicação (não stop-the-world)

### Sweep Phase

- Percorre toda a memória heap
- Libera qualquer objeto que **não foi marcado** como vivo
- Também roda concorrentemente

```
Roots (stacks, globals)
    │
    ▼
  Mark: walk all references, mark reachable objects
    │
    ▼
  Sweep: free unmarked objects
    │
    ▼
  Done — memory reclaimed
```

## Tri-Color Marking

O GC classifica objetos em três cores:

- **White**: ainda não visitado — candidatos à coleta
- **Grey**: visitado, mas referências ainda não totalmente escaneadas
- **Black**: visitado, todas as referências escaneadas — definitivamente vivo

```
Start:   all objects are White
Process: pick Grey → scan its references → move to Black
         referenced Whites → move to Grey
End:     remaining Whites are garbage → free them
```

O invariante chave: **um objeto Black nunca aponta diretamente para um objeto White** (mantido pelos write barriers).

## Write Barrier

Como o GC roda concorrentemente com a aplicação, o programa pode modificar ponteiros enquanto o GC está escaneando.

- O **write barrier** é um pequeno hook que roda a cada escrita de ponteiro durante o GC
- Ele garante que novas referências sejam rastreadas para que o GC não perca objetos vivos
- Pequeno custo de performance, mas habilita a coleta concorrente

## GC Triggers

O GC roda quando:

- O tamanho do heap cresce para **2x** o heap vivo após a última coleta (padrão)
- Controlado pela variável de ambiente `GOGC` (padrão = 100, ou seja, crescimento de 100%)
- `GOGC=50` → GC roda com mais frequência (menos memória, mais CPU)
- `GOGC=200` → GC roda com menos frequência (mais memória, menos CPU)
- `GOGC=off` → desabilita o GC completamente

```go
// Force a GC cycle
runtime.GC()

// Read GC stats
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Println("HeapAlloc:", stats.HeapAlloc)
fmt.Println("NumGC:", stats.NumGC)
fmt.Println("PauseTotalNs:", stats.PauseTotalNs)
```

## GOMEMLIMIT (Go 1.19+)

- Define um **limite de memória soft** para todo o processo Go
- O GC se torna mais agressivo conforme a memória se aproxima do limite
- Melhor do que GOGC sozinho para ambientes com memória limitada (containers)

```bash
GOMEMLIMIT=512MiB ./myapp
```

## STW Pauses (Stop-the-World)

O GC do Go é majoritariamente concorrente, mas tem **duas breves pausas STW**:

1. **Mark start**: habilita o write barrier, escaneia as stacks (~microssegundos)
2. **Mark termination**: finaliza o marking, desabilita o write barrier (~microssegundos)

Pausas STW típicas: **< 1ms**, frequentemente **< 100μs** nas versões modernas do Go.

## Reduzindo a Pressão no GC

Técnicas práticas para minimizar alocações:

- **sync.Pool**: reutiliza objetos temporários em vez de alocar novos
- **Pre-alocar slices**: `make([]T, 0, expectedSize)` evita crescimento repetido
- **Evitar estruturas com muitos ponteiros**: menos ponteiros = menos trabalho para o GC escanear
- **Stack allocation**: manter valores locais, evitar escapar para o heap
- **String builders**: usar `strings.Builder` em vez de concatenação com `+`
- **Escape analysis**: rodar `go build -gcflags="-m"` para ver o que escapa para o heap

```go
// sync.Pool example
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)
    // use buf...
}
```

## Profiling GC

```bash
# GC trace — see every GC cycle
GODEBUG=gctrace=1 ./myapp

# Output example:
# gc 1 @0.012s 2%: 0.019+0.45+0.003 ms clock, ...
#    ^ GC#  ^time ^CPU%  ^STW1 ^concurrent ^STW2

# pprof — heap profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

## Comparação com Outras Linguagens

| Language | GC Type | STW Pauses |
|----------|---------|------------|
| Go | Concurrent mark-sweep | < 1ms |
| Java (G1) | Generational, concurrent | Configurável, pode ser baixo |
| Python | Reference counting + cycle detector | GIL pauses |
| Rust | No GC (ownership system) | None |
| C/C++ | Manual | None |

## Resources

- https://tip.golang.org/doc/gc-guide
- https://go.dev/blog/ismmkeynote (Go GC design talk)

#### Meus comentários
- 
