# Mutex in Go

Um **Mutex** (mutual exclusion) é um lock que garante que apenas uma goroutine acesse um dado compartilhado por vez.

---

## O Problema — Race Condition

Quando múltiplas goroutines leem e escrevem a mesma variável simultaneamente, o resultado é imprevisível:

```go
counter := 0

go func() { counter++ }()
go func() { counter++ }()

// expected: 2
// actual: could be 1 or 2 — race condition
```

Por quê? `counter++` não é uma operação — são três:

```
1. read counter  (gets 0)
2. add 1         (gets 1)
3. write counter (writes 1)
```

Se duas goroutines fazem isso simultaneamente, ambas leem `0`, ambas somam `1`, ambas escrevem `1` — e o resultado final é `1` em vez de `2`.

---

## Mutex — A Solução

Um Mutex é um lock. Apenas uma goroutine pode segurar o lock por vez. As outras aguardam.

```go
var mu sync.Mutex
counter := 0

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

Agora:
```
goroutine 1: Lock() → reads 0 → adds 1 → writes 1 → Unlock()
goroutine 2: waits... Lock() → reads 1 → adds 1 → writes 2 → Unlock()
result: 2 ✓
```

---

## sync.Mutex vs sync.RWMutex

Go tem dois tipos de mutex:

### sync.Mutex — lock exclusivo

Apenas uma goroutine por vez, tanto para leituras quanto para escritas.

```go
var mu sync.Mutex

mu.Lock()    // acquire exclusive lock
// critical section — only one goroutine here
mu.Unlock()  // release
```

### sync.RWMutex — lock de leitura/escrita

Múltiplas goroutines podem ler simultaneamente, mas escritas são exclusivas.

```go
var mu sync.RWMutex

// reading — multiple goroutines allowed at the same time
mu.RLock()
value := store[key]
mu.RUnlock()

// writing — exclusive, blocks all readers and writers
mu.Lock()
store[key] = value
mu.Unlock()
```

**Quando usar RWMutex:** quando leituras são muito mais frequentes que escritas. Um cache é o exemplo clássico — milhares de leituras, escritas ocasionais.

| | **sync.Mutex** | **sync.RWMutex** |
|---|---|---|
| Leituras | Uma por vez | Muitas simultaneamente |
| Escritas | Uma por vez | Uma por vez |
| Usar quando | Leituras/escritas igualmente frequentes | Leituras >> escritas |

---

## defer mu.Unlock() — Sempre Faça Unlock

Se sua função retornar cedo ou entrar em panic sem fazer unlock, o programa entra em deadlock. Use `defer` para garantir o unlock:

```go
func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()  // always runs, even if function panics or returns early

    val, ok := s.data[key]
    return val, ok
}
```

---

## Exemplo Real — Cache Thread-Safe

```go
type Store struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewStore() *Store {
    return &Store{
        data: make(map[string]string),
    }
}

func (s *Store) Set(key, value string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = value
}

func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    val, ok := s.data[key]
    return val, ok
}

func (s *Store) Delete(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.data, key)
}
```

Esse é exatamente o padrão para o seu store do goCache — múltiplas goroutines gerenciando conexões, todas acessando o mesmo map com segurança.

---

## Deadlock — O Principal Perigo

Um deadlock acontece quando duas goroutines estão cada uma esperando a outra liberar um lock — para sempre.

```go
var mu1, mu2 sync.Mutex

// goroutine 1
go func() {
    mu1.Lock()
    mu2.Lock()   // waiting for goroutine 2 to release mu2
    // ...
}()

// goroutine 2
go func() {
    mu2.Lock()
    mu1.Lock()   // waiting for goroutine 1 to release mu1
    // ...
}()

// both wait forever — deadlock
```

**Como evitar:**
- Sempre adquirir locks na mesma ordem
- Manter a seção crítica (código entre Lock e Unlock) o mais curta possível
- Usar `defer mu.Unlock()` para nunca esquecer de fazer unlock

---

## Detectando Race Conditions

Go tem um race detector embutido:

```bash
go run -race ./cmd/server
go test -race ./...
```

Ele instrumenta o código em tempo de execução e reporta qualquer acesso concorrente a dados compartilhados sem sincronização adequada:

```
WARNING: DATA RACE
Write at 0x00c0000b4010 by goroutine 7:
  main.main.func1()
      /main.go:12

Previous read at 0x00c0000b4010 by goroutine 6:
  main.main.func2()
      /main.go:17
```

Rode com `-race` durante o desenvolvimento. Tem um custo de performance (~5x mais lento), portanto não use em produção.

---

## Mutex vs Channel — Quando Usar Cada Um

```
Use Mutex when:   protecting shared state (a map, a counter, a struct)
Use Channel when: communicating between goroutines (sending data, signaling)
```

```go
// Mutex — protecting a shared map
var mu sync.RWMutex
var cache map[string]string

// Channel — signaling between goroutines
results := make(chan string)
go func() { results <- process() }()
```

> "Do not communicate by sharing memory; share memory by communicating." — provérbio do Go
>
> Na prática: se você precisa de uma estrutura de dados compartilhada → Mutex. Se você precisa de coordenação entre goroutines → Channel.

---

## No Contexto do goCache

Seu store é acessado por múltiplas goroutines simultaneamente — uma por conexão de cliente. Sem um Mutex, dois clientes fazendo `SET` ao mesmo tempo corromperiam o map.

```go
// store.go — the pattern to use
type Store struct {
    mu   sync.RWMutex
    data map[string]string
    ttl  map[string]time.Time
}

// SET — exclusive write lock
func (s *Store) Set(key, value string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = value
}

// GET — shared read lock (multiple clients can read simultaneously)
func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    // check TTL, return value...
}
```

---

## Notas Relacionadas

- [[GO/MemoryManagement]]
- [[DistributedSystems/Concurrency]]
- [[DistributedSystems/Performance]]
