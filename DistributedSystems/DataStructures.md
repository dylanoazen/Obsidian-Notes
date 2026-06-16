# Data Structures

Uma estrutura de dados é uma forma de organizar e armazenar dados para que possam ser acessados e modificados de forma eficiente. A escolha da estrutura impacta diretamente a performance — a errada pode transformar uma operação O(1) em O(n).

---

## Redis Data Structures — Quando e Por quê

Cada estrutura existe para resolver um problema específico. Escolher a errada significa mais código, mais lentidão, e soluções gambiarra.

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

## Ordem de Estudo

1. **String** — key/value simples
2. **TTL** — expiração
3. **Hash** — múltiplos campos por chave
4. **List** — sequências ordenadas
5. **Set** — itens únicos
6. **Sorted Set** — ranking com scores

---

## String

A estrutura mais simples — uma chave mapeia diretamente para um valor.

**Implementação em Go:**
```go
map[string]string
```

**Exemplo:**
```go
store := map[string]string{}
store["name"] = "Dylan"
fmt.Println(store["name"]) // → "Dylan"
```

**Custo:** O(1) para get e set.

**Use quando:** armazenar valores simples — sessões de usuário, flags de configuração, contadores.

---

## Hash

Um mapa dentro de um mapa. Cada chave guarda múltiplos campos em vez de um único valor.

**Implementação em Go:**
```go
map[string]map[string]string
```

**Exemplo:**
```go
store := map[string]map[string]string{}
store["user:1"] = map[string]string{
    "name": "Dylan",
    "age":  "25",
}
fmt.Println(store["user:1"]["name"]) // → "Dylan"
```

**Custo:** O(1) para get e set.

**Use quando:** armazenar objetos com múltiplos campos — perfis de usuário, dados de produto, informações de sessão.

---

## List

Uma sequência ordenada onde a ordem de inserção é preservada. Suporta operações eficientes em ambas as extremidades.

**Implementação em Go:**
```go
[]string
```

**Exemplo:**
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

**Custo:** O(1) nas extremidades, O(n) no meio.

**Use quando:** filas, pilhas, logs de histórico, feeds de atividade.

---

## Set

Uma coleção de itens únicos — duplicatas são ignoradas.

**Implementação em Go:**
```go
map[string]struct{}
```

> `struct{}` ocupa zero bytes — é a forma idiomática de implementar um set em Go.

**Exemplo:**
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

**Custo:** O(1) para adicionar, remover e buscar.

**Use quando:** visitantes únicos, tags, permissões, listas de amigos.

---

## Sorted Set

Como um Set, mas cada item tem um **score** que determina sua posição. Itens são sempre mantidos em ordem por score.

**Implementação em Go:**
Duas estruturas trabalhando juntas:
```go
type SortedSet struct {
    scores  map[string]float64 // member → score (O(1) lookup)
    members []string           // kept sorted by score
}
```

> Uma **skip list** ou **heap** pode ser usada internamente para inserções e buscas O(log n).

**Exemplo:**
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

**Custo:** O(log n) para inserção e busca.

**Use quando:** leaderboards, rankings, rate limiting, dados de séries temporais.

---

## TTL / Expiration

Uma forma de anexar um limite de tempo a qualquer valor armazenado — ele se torna automaticamente inválido após o período.

**Implementação em Go:**
Dois mapas trabalhando juntos:
```go
data       map[string]string    // main store
expiration map[string]time.Time // key → expiry time
```

**Exemplo:**
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

**Custo:** O(1) por acesso + limpeza periódica de chaves expiradas.

**Use quando:** sessões, caches, rate limiters, tokens temporários.

---

## Resumo de Complexidade

| Estrutura | Get | Insert | Delete | Caso de uso |
|-----------|-----|--------|--------|----------|
| String | O(1) | O(1) | O(1) | Valores simples |
| Hash | O(1) | O(1) | O(1) | Objetos com campos |
| List | O(n) | O(1) extremidades | O(1) extremidades | Filas, pilhas |
| Set | O(1) | O(1) | O(1) | Itens únicos |
| Sorted Set | O(log n) | O(log n) | O(log n) | Rankings |
| TTL | O(1) | O(1) | O(1) | Chaves com expiração |
