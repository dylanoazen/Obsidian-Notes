# Performance

Performance é sobre entender **onde o tempo e a memória vão** — e reduzir ambos.

Três pilares: **latência**, **alocação de memória** e **benchmarking**.

---

## Latência

Latência é o tempo entre uma requisição ser enviada e uma resposta ser recebida.

```
Client sends request  →  [tempo passa]  →  Client receives response
                              ↑
                          latência
```

**Analogia:** você pede comida em um restaurante. Latência é o tempo entre o pedido e a comida chegar na sua mesa.

### Latência vs Throughput

Esses conceitos são relacionados mas diferentes:

|          | **Latência**                       | **Throughput**                |
| -------- | --------------------------------- | ----------------------------- |
| Pergunta | Quão rápida é uma requisição? | Quantas requisições por segundo? |
| Unidade  | milissegundos (ms)                | requisições/seg (RPS)            |
| Analogia | Quão rápido um carro atravessa a ponte | Quantos carros atravessam por hora |

Você pode ter alto throughput com alta latência — muitas requisições processadas, mas cada uma lentamente. O objetivo geralmente é os dois: rápido e em grande quantidade.

### Fontes de Latência

```
Latência total = rede + fila + processamento + resposta

Rede         → distância física, saltos entre servidores
Fila         → esperar atrás de outras requisições
Processamento → tempo de computação real
Resposta     → enviar o resultado de volta
```

### Números de Latência que Vale Conhecer

| Operação                 | Latência Aproximada |
| ------------------------- | ------------------- |
| Acesso ao cache L1           | ~1 ns               |
| Acesso à RAM                 | ~100 ns             |
| Leitura de SSD               | ~100 µs             |
| Rede (mesmo datacenter)      | ~1 ms               |
| Rede (intercontinental)      | ~100 ms             |
| Leitura de HDD               | ~10 ms              |

É por isso que Redis é rápido — ele lê da RAM (~100 ns), não do disco.

### P50, P95, P99 — Latência por Percentil

Médias mentem. Um sistema com média de 10ms de latência pode ter 1% das requisições levando 2 segundos.

Percentis contam a história real:

```
P50  = 50% das requisições terminam em menos de X ms  (mediana)
P95  = 95% das requisições terminam em menos de X ms
P99  = 99% das requisições terminam em menos de X ms
P999 = 99,9% das requisições terminam em menos de X ms
```

**Analogia:** salário médio em uma empresa com um CEO bilionário parece alto. A mediana (P50) diz o que a maioria dos funcionários realmente ganha.

Em produção, você otimiza o P99 — o pior 1% ainda afeta usuários reais.

---

## Memory Allocator

Um memory allocator gerencia como seu programa requisita e libera memória do OS.

Seu programa nunca fala diretamente com o hardware. Quando você precisa de memória, fala com o allocator, e ele fala com o OS:

```
Your code  →  malloc(64)  →  Allocator  →  OS (only when necessary)
```

O OS trabalha em páginas (geralmente 4KB). Alocar 4KB toda vez que você precisar de 64 bytes seria desperdício. O allocator requisita páginas grandes do OS e as subdivide internamente.

### Como a Alocação de Memória Funciona

Quando seu programa precisa de memória:

```
Programa  →  "Preciso de 64 bytes"  →  Allocator
Allocator → encontra bloco livre → retorna ponteiro
Programa usa a memória
Programa  →  "Terminei com isso"  →  Allocator
Allocator → marca bloco como livre → disponível para reutilização
```

### O Problema: Fragmentação

Com o tempo, a memória fica parecendo queijo suíço — muitos buracos pequenos que são difíceis de reutilizar:

```
Antes:
[████████████████████████████████] 100MB contíguo

Depois de muitas alocações/liberações:
[used][    ][used][used][    ][used][    ]
       16MB        8MB        4MB
```

Você tem 28MB livres no total, mas em pedaços separados. Se precisar de 20MB contíguo — não está disponível.

### Como ptmalloc (glibc) Funciona — e Por que É Lento

ptmalloc foi projetado em 1987 para cargas de trabalho single-threaded e teve suporte a threads adicionado depois. Seu modelo é simples demais para servidores modernos:

```
Thread 1 quer alocar  →  trava a heap  →  aloca  →  destrava
Thread 2 quer alocar  →  esperando na fila...
Thread 3 quer alocar  →  esperando na fila...
```

Em sistemas com muitas threads, as threads ficam dormindo esperando o lock. Isso vira latência.

### Como jemalloc Resolve Isso — Arenas

jemalloc divide a memória em **arenas** independentes. Cada thread é atribuída a uma arena, então threads raramente competem pelo mesmo lock:

```
Thread 1  →  Arena 1  (lock próprio)
Thread 2  →  Arena 2  (lock próprio)
Thread 3  →  Arena 1  (compartilha com Thread 1, mas raramente conflita)
```

Dentro de cada arena, jemalloc divide alocações em **classes de tamanho** — tamanhos fixos predefinidos:

```
Small  (≤ 14KB)   →  cache por thread (tcache) — zero locks
Large  (14KB–4MB) →  diretamente da arena
Huge   (> 4MB)    →  mmap'd diretamente do OS
```

Para alocações pequenas (a maioria), jemalloc usa um **cache por thread** (tcache) — zero contenção, zero lock.

jemalloc também resolve fragmentação com classes de tamanho — em vez de alocar exatamente o que foi requisitado, arredonda para o tamanho fixo mais próximo:

```
You request 57 bytes  →  jemalloc allocates 64 bytes (next size class)
You request 100 bytes →  jemalloc allocates 128 bytes
```

Resultado: blocos do mesmo tamanho ficam juntos, fáceis de reutilizar:

```
[64][64][64][64][64]  ← todos iguais, qualquer um serve para qualquer requisição ≤64 bytes
```

### Allocators Comuns

| Allocator | Usado por | Notas |
|---|---|---|
| **glibc (ptmalloc)** | Padrão no Linux | Uso geral, lento sob contenção |
| **jemalloc** | Redis, Firefox, Rust | Melhor tratamento de fragmentação para servidores de longa duração |
| **tcmalloc** | Google, Go | Cache privado por thread, melhor throughput para alta concorrência |
| **mimalloc** | Microsoft | Moderno, muito rápido em diversas cargas de trabalho |

### jemalloc vs tcmalloc

| | **jemalloc** | **tcmalloc** |
|---|---|---|
| Forte em | Fragmentação — servidores de longa duração | Throughput — muitas threads, muitas alocações pequenas |
| Modelo de thread | Arenas compartilhadas | Cache totalmente privado por thread |
| Ideal para | Redis, bancos de dados | Servidores HTTP, aplicações estilo Google |
| Fraco em | Alocações pequenas de alta frequência | Longa duração com padrões variados de alocação |

### Como Isso Aparece no Redis

Redis reporta duas métricas de memória:

```bash
INFO memory

used_memory:     1000000000   # 1GB — o que o Redis acha que está usando
used_memory_rss: 1500000000   # 1.5GB — o que o OS vê

mem_fragmentation_ratio: 1.5  # used_by_OS / used_by_Redis
```

**Fragmentation ratio acima de 1.5 = problema.** jemalloc está mantendo memória em seus bins internos que o OS conta como usada, mas Redis não está ativamente usando.

**Como corrigir:**

```bash
MEMORY PURGE  # Redis 4.0+ — força jemalloc a retornar páginas ociosas ao OS
```

Redis 4.0+ também tem **active defragmentation** — realoca automaticamente chaves fragmentadas em background.

### Por que Redis Escolheu jemalloc

Redis é um processo que roda por semanas ou meses, fazendo milhões de pequenas alocações (chaves, valores, estruturas internas). Esse é exatamente o cenário onde jemalloc se destaca — fragmentação controlada ao longo do tempo.

Com ptmalloc, após semanas de execução o processo consumiria 3-4x mais RAM do que os dados reais ocupam.

### Por que Isso Importa

Um alocador ruim em um sistema de alto tráfego pode:
- Fragmentar a memória → processo usa 2GB de RAM mas apenas 1GB são dados reais
- Lentificar alocações → latência extra em toda operação
- Causar OOM (out of memory) inesperado

**Referências:**
- [Memory Allocators: malloc vs. tcmalloc vs. jemalloc](https://howtech.substack.com/p/memory-allocators-malloc-vs-tcmalloc)
- [Exploring Different Memory Allocators](https://dev.to/frosnerd/libmalloc-jemalloc-tcmalloc-mimalloc-exploring-different-memory-allocators-4lp3)
- [How Redis jemalloc Memory Allocator Works](https://oneuptime.com/blog/post/2026-03-31-redis-jemalloc-memory-allocator/view)
- [How to Troubleshoot Redis Memory Fragmentation](https://oneuptime.com/blog/post/2026-03-31-redis-how-to-troubleshoot-redis-memory-fragmentation/view)
- [Testing Memory Allocators: ptmalloc2 vs tcmalloc vs hoard vs jemalloc](http://ithare.com/testing-memory-allocators-ptmalloc2-tcmalloc-hoard-jemalloc-while-trying-to-simulate-real-world-loads/)

---

## Benchmarking

Benchmarking é o processo de **medir performance em condições controladas** para entender onde seu sistema está e onde pode melhorar.

### Por que Fazer Benchmark

- Verificar se seu sistema atende aos requisitos de performance
- Comparar duas implementações (qual é mais rápida?)
- Detectar regressões — o novo código tornou as coisas mais lentas?
- Encontrar gargalos — onde o tempo está realmente sendo gasto?

### Tipos de Benchmarks

**Microbenchmark** — mede uma única operação pequena isoladamente

```go
// how fast is a single SET command?
for i := 0; i < 1000000; i++ {
    redis.Set("key", "value")
}
```

**Load test** — simula tráfego real em escala

```
Envia 10.000 requisições/seg por 60 segundos
Mede: latência P50/P95/P99, taxa de erro, throughput
```

**Stress test** — empurra além dos limites esperados para encontrar pontos de ruptura

```
Continua aumentando a carga até o sistema falhar
Observa: quando ele degrada? quando quebra?
```

### Ferramentas Comuns de Benchmarking

| Ferramenta | Caso de uso |
|---|---|
| **redis-benchmark** | Benchmark integrado do Redis |
| **wrk / wrk2** | Teste de carga HTTP |
| **k6** | Teste de carga com scripts |
| **pprof** (Go) | Profiling de CPU e memória |
| **perf** (Linux) | Análise de performance em nível de sistema |

### Exemplo de Benchmark Redis

```bash
# run 100,000 SET commands with 50 concurrent clients
redis-benchmark -t set -n 100000 -c 50

# output:
# 100000 requests completed in 1.23 seconds
# 50 parallel clients
# Latency: avg 0.5ms, P99 1.2ms
```

### Armadilhas de Benchmarking

- **Faça benchmark em condições similares à produção** — seu laptop não é seu servidor
- **Aqueça primeiro** — cold start distorce resultados (JIT, caches, conexões)
- **Meça P99, não apenas a média** — médias escondem outliers
- **Isole variáveis** — mude uma coisa por vez
- **Execute múltiplas vezes** — variância é real, uma execução não é suficiente

---

## Como Eles Se Conectam

```
Latência      →  o que você mede (quão lento está?)
Allocator     →  uma fonte de latência (operações de memória)
Benchmarking  →  como você encontra de onde vem a latência
```

O ciclo:

```
Benchmark → encontra gargalo → corrige (ajusta allocator, reduz cópias, cache) → benchmark novamente
```

---

## No Contexto Redis

- Redis é rápido porque opera inteiramente em **RAM** — latência em microssegundos
- Usa **jemalloc** para minimizar fragmentação ao longo de longas execuções
- **redis-benchmark** é integrado para verificações rápidas de performance
- Picos de latência no Redis geralmente vêm de: chaves grandes, comandos bloqueantes, flushes de persistência (AOF fsync)

---

## Notas de Design

- Otimização prematura é a raiz de todo mal — faça benchmark primeiro, otimize o que os dados mostram
- Orçamento de latência: saiba quanta latência cada camada pode adicionar
- Fragmentation ratio de memória acima de 1.5 no Redis é um sinal de alerta
- Sempre meça antes e depois de uma otimização para confirmar que ajudou

---

## Notas Relacionadas

- [[Replication]]
- [[Persistence]]
- [[TLS]]
