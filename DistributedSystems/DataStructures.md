# Data Structures

A data structure is a way of organizing and storing data so it can be accessed and modified efficiently. The choice of structure directly impacts performance — the wrong one can turn an O(1) operation into O(n).

---

## Redis Data Structures — When and Why

Each structure exists to solve a specific problem. Choosing the wrong one means more código, mais lentidão, e soluções gambiarra.

### String

A estrutura mais simples — uma chave aponta pra um valor.

Serve pra qualquer coisa que caiba num único valor: texto, número, JSON serializado, binário.

```
SET user:123:name "Dylan"
SET user:123:age  "25"
SET config:feature_flag "true"
```

**Casos de uso:**
- Sessões de usuário
- Contadores (`INCR visits:page:home`)
- Feature flags
- Cache de qualquer coisa

**Analogia:** uma gaveta com etiqueta. Você abre pelo nome e tem uma coisa dentro.

---

### List

Uma coleção **ordenada** onde a ordem de inserção importa. Suporta inserção e remoção eficiente nas duas pontas.

```
LPUSH fila:emails "email1"   ← insere no início
RPUSH fila:emails "email2"   ← insere no final
RPOP  fila:emails            ← remove do final (consome a fila)
```

**Casos de uso:**
- Filas de jobs (LPUSH pra inserir, BRPOP pra consumir)
- Histórico de atividade (últimas 10 ações do usuário)
- Feed de notificações

**Analogia:** uma fila de banco — quem chegou primeiro sai primeiro. Ou uma pilha de pratos — você empilha e desempilha pelo topo.

---

### Set

Uma coleção de elementos **únicos sem ordem**. Duplicatas são ignoradas automaticamente.

```
SADD tags:post:1 "redis" "backend" "cache"
SADD tags:post:1 "redis"   ← ignorado, já existe

SMEMBERS tags:post:1       ← retorna todos
SISMEMBER tags:post:1 "redis"  ← verifica se existe
```

Operações poderosas entre sets:
```
SUNION  set1 set2   ← união (tudo de ambos)
SINTER  set1 set2   ← interseção (só o que está nos dois)
SDIFF   set1 set2   ← diferença (o que está no set1 mas não no set2)
```

**Casos de uso:**
- Tags de um post
- Usuários online
- Permissões de um usuário
- Amigos em comum (SINTER entre dois sets de amigos)

**Analogia:** uma sacola onde você joga itens — mas se jogar o mesmo item duas vezes, só fica um.

---

### Sorted Set (ZSet)

Como o Set, mas cada membro tem uma **pontuação (score)**. Os membros ficam sempre ordenados pela pontuação automaticamente.

```
ZADD ranking:vendedores 1500 "Dylan"
ZADD ranking:vendedores 3200 "Alice"
ZADD ranking:vendedores 800  "Bob"

ZRANGE ranking:vendedores 0 -1 WITHSCORES
→ Bob: 800
→ Dylan: 1500
→ Alice: 3200
```

**Casos de uso:**
- Placar de jogos (leaderboard)
- Ranking de vendedores
- Fila de prioridade (score = prioridade)
- Rate limiting por tempo (score = timestamp)

**Analogia:** uma competição com placar — você adiciona participantes com suas pontuações e o Redis mantém o ranking atualizado automaticamente.

---

### Hash

Um mapa dentro de uma chave — em vez de um valor único, a chave guarda **múltiplos campos**.

```
HSET user:123 name "Dylan" age "25" city "São Paulo"

HGET    user:123 name       ← "Dylan"
HGETALL user:123            ← todos os campos
HDEL    user:123 city       ← remove um campo
```

**Casos de uso:**
- Perfil de usuário (vários campos numa chave só)
- Dados de produto
- Configurações de sessão

**Sem Hash** você precisaria de uma chave pra cada campo:
```
user:123:name  "Dylan"
user:123:age   "25"
user:123:city  "São Paulo"
```

Com Hash, tudo numa chave só. Mais organizado e menos memória.

**Analogia:** uma ficha de cadastro — tem campos diferentes (nome, idade, cidade) todos agrupados num único formulário.

---

### Stream

Um log **append-only** — você só adiciona entradas, nunca edita ou remove. Cada entrada tem um ID com timestamp automático.

```
XADD eventos:pagamentos * user_id 123 valor 150.00 status "aprovado"
XADD eventos:pagamentos * user_id 456 valor 89.90  status "recusado"

XREAD COUNT 10 STREAMS eventos:pagamentos 0
→ retorna as últimas 10 entradas
```

**Casos de uso:**
- Event sourcing (log de tudo que aconteceu)
- Fila de mensagens com múltiplos consumidores
- Auditoria de ações
- Integração entre serviços (parecido com Kafka, mas simples)

**Analogia:** um livro de registro de portaria — você só escreve novas entradas, nunca apaga as antigas. Qualquer um pode ler o histórico completo.

---

### Resumo — Qual usar quando

| Estrutura | Use quando |
|---|---|
| **String** | Um valor simples por chave — contador, sessão, flag |
| **List** | Ordem importa — fila, histórico, feed |
| **Set** | Unicidade importa — tags, permissões, amigos |
| **Sorted Set** | Unicidade + ordem por pontuação — ranking, placar |
| **Hash** | Múltiplos campos por chave — objeto, perfil |
| **Stream** | Log imutável — eventos, auditoria, mensageria |

---

## Study Order

1. **String** — simple key/value
2. **TTL** — expiration
3. **Hash** — multiple fields per key
4. **List** — ordered sequences
5. **Set** — unique items
6. **Sorted Set** — ranking with scores

---

## String

The simplest structure — a key maps directly to a value.

**Go implementation:**
```go
map[string]string
```

**Example:**
```go
store := map[string]string{}
store["name"] = "Dylan"
fmt.Println(store["name"]) // → "Dylan"
```

**Cost:** O(1) for get and set.

**Use when:** storing simple values — user sessions, config flags, counters.

---

## Hash

A map inside a map. Each key holds multiple fields instead of a single value.

**Go implementation:**
```go
map[string]map[string]string
```

**Example:**
```go
store := map[string]map[string]string{}
store["user:1"] = map[string]string{
    "name": "Dylan",
    "age":  "25",
}
fmt.Println(store["user:1"]["name"]) // → "Dylan"
```

**Cost:** O(1) for get and set.

**Use when:** storing objects with multiple fields — user profiles, product data, session info.

---

## List

An ordered sequence where insertion order is preserved. Supports efficient operations at both ends.

**Go implementation:**
```go
[]string
```

**Example:**
```go
list := []string{}

// insert at end
list = append(list, "task1")

// insert at start
list = append([]string{"task0"}, list...)

// remove from start
first := list[0]
list = list[1:]

// remove from end
last := list[len(list)-1]
list = list[:len(list)-1]
```

**Cost:** O(1) at the ends, O(n) in the middle.

**Use when:** queues, stacks, history logs, activity feeds.

---

## Set

A collection of unique items — duplicates are ignored.

**Go implementation:**
```go
map[string]struct{}
```

> `struct{}` takes zero bytes — it's the idiomatic way to implement a set in Go.

**Example:**
```go
set := map[string]struct{}{}

// add
set["go"] = struct{}{}
set["go"] = struct{}{} // ignored — already exists

// check existence
_, exists := set["go"] // → true

// remove
delete(set, "go")
```

**Cost:** O(1) for add, remove and lookup.

**Use when:** unique visitors, tags, permissions, friend lists.

---

## Sorted Set

Like a Set, but each item has a **score** that determines its position. Items are always kept in order by score.

**Go implementation:**
Two structures working together:
```go
type SortedSet struct {
    scores  map[string]float64 // member → score (O(1) lookup)
    members []string           // kept sorted by score
}
```

> A **skip list** or **heap** can be used internally for O(log n) inserts and lookups.

**Example:**
```go
ss := SortedSet{
    scores:  map[string]float64{},
    members: []string{},
}

// add member with score
ss.scores["Dylan"] = 100
ss.scores["Alice"] = 250
// members slice must be kept sorted after each insert
```

**Cost:** O(log n) for insert and lookup.

**Use when:** leaderboards, rankings, rate limiting, time-series data.

---

## TTL / Expiration

A way to attach a time limit to any stored value — it automatically becomes invalid after the duration.

**Go implementation:**
Two maps working together:
```go
data       map[string]string    // main store
expiration map[string]time.Time // key → expiry time
```

**Example:**
```go
data := map[string]string{}
expiration := map[string]time.Time{}

// store with TTL
data["session:abc"] = "token123"
expiration["session:abc"] = time.Now().Add(1 * time.Hour)

// get with expiry check
func get(key string) (string, bool) {
    if exp, ok := expiration[key]; ok {
        if time.Now().After(exp) {
            delete(data, key)
            delete(expiration, key)
            return "", false // expired
        }
    }
    val, ok := data[key]
    return val, ok
}
```

**Cost:** O(1) per access + periodic cleanup for expired keys.

**Use when:** sessions, caches, rate limiters, temporary tokens.

---

## Complexity Summary

| Structure | Get | Insert | Delete | Use case |
|-----------|-----|--------|--------|----------|
| String | O(1) | O(1) | O(1) | Simple values |
| Hash | O(1) | O(1) | O(1) | Objects with fields |
| List | O(n) | O(1) ends | O(1) ends | Queues, stacks |
| Set | O(1) | O(1) | O(1) | Unique items |
| Sorted Set | O(log n) | O(log n) | O(log n) | Rankings |
| TTL | O(1) | O(1) | O(1) | Expiring keys |
