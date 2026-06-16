# Memory Management in Go

Go gerencia memória automaticamente — você aloca, o runtime faz a limpeza. Mas entender como funciona internamente permite escrever código mais rápido e previsível.

Três conceitos: **Stack**, **Heap** e **Garbage Collector**.

---

## Stack

A stack é uma memória rápida e automática vinculada às chamadas de função.

- Quando uma função é chamada → um stack frame é criado
- Quando uma função retorna → o stack frame é destruído
- Cada goroutine tem sua própria stack (começa em ~2KB, cresce conforme necessário)

```go
func add(a, b int) int {
    result := a + b   // lives on the stack
    return result     // stack frame destroyed on return
}
```

**Propriedade chave:** memória na stack é gratuita — sem alocador, sem GC. Apenas um ponteiro movendo para cima e para baixo.

**Analogia:** um quadro branco que você apaga após cada reunião. Instantâneo para escrever, instantâneo para apagar.

---

## Heap

O heap é a memória para dados que precisam **sobreviver a uma função** ou têm um **tamanho desconhecido em tempo de compilação**.

```go
func newUser(name string) *User {
    u := &User{Name: name}  // escapes to heap
    return u                // function returns, but User survives
}
```

Se `User` vivesse na stack, seria destruído quando `newUser` retornasse. Como retornamos um ponteiro para ele, Go o coloca no heap.

Alocações no heap são mais lentas — passam pelo alocador de memória (Go usa um alocador customizado inspirado no tcmalloc) e eventualmente precisam ser limpas pelo GC.

**Analogia:** um depósito que você aluga. Mais espaço, mas alguém precisa gerenciá-lo.

---

## Escape Analysis

Go decide em **tempo de compilação** se uma variável vai para a stack ou para o heap.

> Se a variável é usada apenas dentro da função → **stack**
> Se a variável "escapa" da função → **heap**

```go
// stack — used only locally
func localOnly() {
    x := 42
    fmt.Println(x)
}

// heap — pointer returned, x outlives the function
func escapes() *int {
    x := 42
    return &x
}

// heap — sent to a channel, outlives the function
func toChannel(ch chan *User) {
    u := &User{}
    ch <- u
}
```

### Visualizando as decisões do escape analysis

```bash
go build -gcflags="-m" ./...

# output:
# ./main.go:8:2: x escapes to heap
# ./main.go:14:6: u does not escape
```

Isso é útil ao otimizar hot paths — cada escape para o heap tem um custo de GC.

---

## Garbage Collector

O GC libera automaticamente a memória heap que nada mais aponta.

### Como funciona — Tri-color Mark and Sweep

**Mark phase:** começando pelas roots (variáveis de stack, globals), seguir todos os ponteiros e marcar tudo que é acessível:

```
roots → User{} ✓ → name string ✓ → ...
      → cache map ✓ → entries ✓ → ...
      → (old buffer nobody points to) ✗ — never reached
```

**Sweep phase:** liberar tudo que não foi marcado:

```
[User ✓][string ✓][buffer ✗ freed][map ✓][[]byte ✗ freed]
```

### Concurrent GC — a abordagem do Go

O GC do Go roda **concorrentemente** com o seu programa. A maior parte do trabalho acontece enquanto as goroutines continuam executando:

```
your program:  ────────────────────────────────────────
GC:                  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                              |short STW|   ← stop-the-world pause (< 1ms)
```

As pausas STW (stop-the-world) no Go moderno são tipicamente abaixo de **1ms**. Mas sob alocação intensa, o GC roda com mais frequência — e as pausas se acumulam na latência P99.

### GOGC — controlando a frequência do GC

```bash
GOGC=100   # default — GC triggers when heap doubles in size
GOGC=200   # GC runs less often — uses more memory, less CPU
GOGC=50    # GC runs more often — less memory, more CPU
GOGC=off   # disable GC entirely (only for special cases)
```

GOGC maior = menos trabalho de GC = menor custo de CPU = maior uso de memória.
GOGC menor = mais trabalho de GC = maior custo de CPU = menor uso de memória.

---

## Reduzindo a Pressão no GC

Cada alocação no heap é trabalho futuro para o GC. Reduzir alocações = reduzir pressão no GC = menor latência P99.

### 1. sync.Pool — reutilizar objetos

Em vez de alocar um novo buffer a cada requisição, reutilizá-los:

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handleRequest() {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)
    // use buf to parse command, write response...
}
```

O buffer é reutilizado entre requisições — o alocador e o GC nunca o veem após a primeira alocação.

### 2. Evitar ponteiros desnecessários

Cada ponteiro que o GC precisa seguir é mais trabalho. Structs planas com tipos por valor são amigáveis ao GC:

```go
// GC-heavy — pointer to every field
type Heavy struct {
    Name    *string
    Age     *int
    Active  *bool
}

// GC-friendly — value types, no pointers
type Light struct {
    Name    string
    Age     int
    Active  bool
}
```

### 3. Pré-alocar slices

```go
// bad — slice grows, reallocates multiple times
results := []string{}
for _, item := range items {
    results = append(results, process(item))
}

// good — one allocation upfront
results := make([]string, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}
```

### 4. Evitar concatenação com + em loops

```go
// bad — allocates a new string on every iteration
s := ""
for _, part := range parts {
    s += part  // new allocation each time
}

// good — one allocation at the end
var sb strings.Builder
for _, part := range parts {
    sb.WriteString(part)
}
s := sb.String()
```

---

## Profiling — Encontrando Onde as Alocações Acontecem

Não adivinhe. Meça.

```bash
# run with profiling
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap

# shows which functions allocate the most
```

```go
// expose pprof endpoint in your server
import _ "net/http/pprof"

go http.ListenAndServe(":6060", nil)
```

### Estatísticas de memória em tempo de execução

```go
var stats runtime.MemStats
runtime.ReadMemStats(&stats)

fmt.Println("heap in use:", stats.HeapInuse)
fmt.Println("GC cycles:", stats.NumGC)
fmt.Println("GC pause total:", stats.PauseTotalNs)
```

---

## Stack vs Heap — Resumo

| | **Stack** | **Heap** |
|---|---|---|
| Velocidade | Instantâneo | Mais lento (alocador + GC) |
| Tamanho | Pequeno, por goroutine | Grande, compartilhado |
| Tempo de vida | Escopo da função | Até o GC liberar |
| Envolvimento do GC | Nenhum | Sim |
| Como usar | Variáveis locais | Ponteiros, interfaces, closures |

---

## No Contexto do MiniRedisGo

Cada comando `SET key value` que o seu servidor processa envolve alocações:

```
parse command  → []byte buffer  → heap
store key      → string         → heap
store value    → []byte         → heap
write response → []byte buffer  → heap
```

Sob carga intensa, isso se torna pressão significativa no GC. Otimizações:

- Usar `sync.Pool` para buffers de parsing de comandos
- Reutilizar buffers de resposta
- Fazer profiling com pprof sob carga para encontrar os maiores alocadores
- Monitorar `mem_fragmentation_ratio` se você adicionar uma camada de persistência

---

## Notas Relacionadas

- [[DistributedSystems/MemoryManagement]]
- [[DistributedSystems/Performance]]
- [[Concurrency]]
