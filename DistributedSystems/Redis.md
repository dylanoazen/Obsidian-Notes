# Redis

Redis (Remote Dictionary Server) é um armazenamento de estruturas de dados em memória usado como banco de dados, cache, message broker e engine de streaming.

Ele oferece:
- armazenamento key-value com latência sub-milissegundo
- estruturas de dados ricas além de simples strings
- opções de persistência para durabilidade
- replicação e clustering integrados

---

## Por que Redis?

- Extremamente rápido — todas as operações acontecem em memória
- Estruturas de dados versáteis (não apenas key-value)
- Pub/sub integrado para mensageria em tempo real
- Opções de persistência (snapshots RDB, logs AOF)
- Replicação e alta disponibilidade prontos para uso
- Amplamente adotado — usado pelo Twitter, GitHub, Stack Overflow, etc.

## Estruturas de Dados Principais

- **String**: tipo mais simples, pode armazenar texto, números ou binário (máx 512MB)
- **List**: coleção ordenada, suporta push/pop em ambas as extremidades (filas, pilhas)
- **Set**: elementos únicos sem ordem, suporta uniões, interseções e diferenças
- **Sorted Set (ZSet)**: como Set mas cada membro tem um score, ordenado automaticamente
- **Hash**: pares campo-valor dentro de uma única chave (como um mini objeto/documento)
- **Stream**: log append-only, usado para event sourcing e filas de mensagens

## Comandos Básicos

```
SET key value              # store a string
GET key                    # retrieve a string
DEL key                    # delete a key
EXPIRE key seconds         # set TTL
TTL key                    # check remaining TTL

LPUSH list value           # push to head of list
RPOP list                  # pop from tail

SADD set member            # add to set
SMEMBERS set               # list all members

HSET hash field value      # set field in hash
HGETALL hash               # get all fields

ZADD zset score member     # add to sorted set
ZRANGE zset 0 -1           # get all sorted
```

## Arquitetura

```
Client ──► Redis Server (single-threaded event loop)
                │
                ├── Memory (primary storage)
                ├── RDB (periodic snapshots to disk)
                ├── AOF (append-only log to disk)
                └── Replication → Replica nodes
```

- Single-threaded para comandos — sem locks, sem race conditions
- I/O multiplexing lida com milhares de conexões concorrentes
- Threads em background cuidam de persistência e operações lentas

## Tópicos

- [[DistributedSystems/Persistence|Persistence (RDB & AOF)]]
- [[DistributedSystems/Replication|Replication]]
- [[DistributedSystems/PubSub|Pub/Sub]]
- [[DistributedSystems/TransactionsPipeline|Transactions & Pipeline]]
- [[DistributedSystems/SecurityAuth|Security & Auth]]
- [[DistributedSystems/TLS|TLS]]
- [[DistributedSystems/Performance|Performance]]
- [[DistributedSystems/MemoryManagement|Memory Management]]
- [[DistributedSystems/Concurrency|Concurrency]]
- [[DistributedSystems/DataStructures|Data Structures]]
- [[DistributedSystems/ExpirationTTL|Expiration & TTL]]

## Casos de Uso

- **Caching**: armazenamento de sessão, cache de resposta de API, cache de página
- **Rate Limiting**: rastreia contagens de requisições com INCR + EXPIRE
- **Leaderboards**: sorted sets com scores
- **Chat em tempo real**: channels pub/sub
- **Filas de Jobs**: lists com padrão LPUSH/BRPOP
- **Session Store**: leitura/escrita rápida para sessões de usuário

---

## Gerenciamento de Sessão — Como Funciona na Prática

Redis não tem tabelas ou SQL. O "vínculo" com o usuário é construído no próprio nome da chave:

```
key:   "session:token:abc32323"   ← identifica de quem é a sessão
value: "123"                       ← o ID do usuário
TTL:   3600 seconds                ← expira automaticamente
```

### Fluxo Completo

**Login:**
```
Usuário faz login com senha correta
→ servidor gera um token aleatório: "abc32323"
→ SET session:token:abc32323 "123" EX 3600
→ retorna o token para o browser (cookie/header)
```

**Em cada requisição:**
```
Browser envia o token "abc32323"
→ servidor pergunta ao Redis: GET session:token:abc32323
→ Redis retorna: "123" (o ID do usuário)
→ autenticado ✓

Se Redis retornar nil → sessão expirada ou nunca existiu → 401
```

**Logout:**
```
DEL session:token:abc32323
→ próxima requisição: GET session:token:abc32323 → nil → não autenticado
```

### Padrão de Middleware (PHP)

A verificação de sessão não fica dentro de cada ação — ela fica em um único arquivo incluído em todo lugar. Uma verificação roda antes de qualquer outra coisa em cada página protegida.

```php
// config.php — included in every protected page

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$token = $_COOKIE['token'] ?? null;

if (!$token) {
    header('Location: /login.php');
    exit;
}

$user_id = $redis->get("session:token:$token");

if (!$user_id) {
    header('Location: /login.php');
    exit;
}

// if we got here, $user_id is available for the entire page
```

Cada página protegida:
```php
<?php
require_once 'config.php';  // ← verificação feita, $user_id disponível

// resto da página normalmente
echo "Welcome, user $user_id";
?>
```

A página de login é a única que **não** inclui este arquivo — ou inclui sem o redirecionamento, já que o usuário ainda não tem sessão.

> Redis não te notifica quando uma sessão expira — apenas retorna nil. Sua aplicação decide o que fazer: redirecionar para o login, retornar 401, etc.

## Recursos

- https://redis.io/docs
- https://try.redis.io (tutorial interativo)
- https://university.redis.io (cursos gratuitos)

#### Meus comentários
-  
