# Memory Management

Gerenciamento de memória é sobre entender **onde os dados vivem, como chegam lá e como são limpos**.

Todo programa precisa de memória para armazenar dados enquanto executa. Como essa memória é requisitada, usada e liberada determina a performance e a confiabilidade do sistema.

---

## Stack vs Heap

Todo programa tem dois lugares principais para armazenar dados:

```
┌─────────────────────────────────┐
│             STACK               │  ← rápida, pequena, automática
├─────────────────────────────────┤
│              HEAP               │  ← mais lenta, grande, gerenciada
└─────────────────────────────────┘
```

---

## Stack

A stack é memória rápida e automática vinculada a chamadas de função.

- Quando uma função é chamada → um stack frame é criado
- Quando uma função retorna → o stack frame é destruído
- Nenhum alocador envolvido — apenas um ponteiro subindo e descendo
- Cada thread tem sua própria stack

```
main() calls handleRequest()
  handleRequest() calls parseCommand()

Stack:
┌──────────────────┐
│  parseCommand()  │  ← topo (função atual)
├──────────────────┤
│ handleRequest()  │
├──────────────────┤
│     main()       │
└──────────────────┘

parseCommand() retorna → frame destruído
handleRequest() retorna → frame destruído
```

### O que vive na stack

- Variáveis locais com tamanho fixo e conhecido
- Argumentos de função e valores de retorno
- Variáveis que não sobrevivem além da função

**Analogia:** um quadro branco que você apaga após cada reunião. Instantâneo para escrever, instantâneo para apagar, mas com espaço limitado.

---

## Heap

A heap é um grande pool de memória para dados que precisam **sobreviver além de uma função** ou que têm **tamanho desconhecido em tempo de compilação**.

- Mais lenta para alocar — passa por um alocador de memória (jemalloc, tcmalloc, etc.)
- Vive até ser explicitamente liberada ou coletada pelo garbage collector
- Compartilhada entre threads

Se uma variável precisa sobreviver após o retorno da função que a criou, ela deve viver na heap.

**Analogia:** um depósito que você aluga. Mais espaço, mas alguém precisa gerenciá-lo — e o aluguel precisa ser pago até você sair.

---

## Gerenciamento Manual vs Automático de Memória

Linguagens diferentes lidam com a limpeza da heap de formas distintas:

| Abordagem | Linguagem | Como |
|---|---|---|
| **Manual** | C, C++ | Programador chama `free()` — rápido mas propenso a erros |
| **Garbage Collected** | Go, Java, Python | Runtime limpa automaticamente |
| **Sistema de Ownership** | Rust | Compilador impõe regras em tempo de compilação — sem GC necessário |

### Manual (C / C++)

```c
char *buf = malloc(1024);  // allocate
// use buf...
free(buf);                 // you must free it
// if you forget → memory leak
// if you free twice → crash
// if you use after free → undefined behavior
```

Rápido, mas perigoso. Redis é escrito em C — cada alocação é manual.

### Garbage Collector

O runtime periodicamente varre a heap, encontra dados para os quais nada mais aponta e os libera automaticamente.

```
heap:
[User A ✓][string ✓][int ✗][User B ✓][[]byte ✗]
                      ↑ nada aponta aqui    ↑ nada aponta aqui

GC roda → libera os blocos ✗
```

Mais seguro e mais fácil, mas o GC tem um custo — roda na CPU e pode causar pausas de latência.

### Ownership (Rust)

O compilador rastreia quem possui cada pedaço de memória. Quando o dono sai do escopo, a memória é liberada automaticamente — em tempo de compilação, sem custo em runtime.

```rust
{
    let s = String::from("hello");  // s owns the memory
    // use s...
}  // s goes out of scope → memory freed automatically, no GC needed
```

---

## Garbage Collector — Como Funciona

O algoritmo de GC mais comum é o **Mark and Sweep**:

**Fase 1 — Mark:**
A partir das raízes (variáveis na stack, globais), segue todos os ponteiros e marca tudo que é alcançável:

```
roots → User{} ✓ → name string ✓
      → cache map ✓ → entries ✓
      → (old buffer nobody points to) ✗ — nunca alcançado
```

**Fase 2 — Sweep:**
Libera tudo que não foi marcado:

```
[User ✓][string ✓][buffer ✗ freed][map ✓][[]byte ✗ freed]
```

### Stop-the-World vs GC Concorrente

Implementações mais antigas de GC pausavam o programa inteiro durante a coleta:

```
Stop-the-world:
────────────────|░░░░░░░GC░░░░░░|────────────────
programa roda    tudo pausa       programa retoma
```

Isso causava picos de latência. GCs modernos (Go, JVM G1) rodam **concorrentemente** — a maior parte do trabalho acontece enquanto o programa continua rodando, com apenas breves pausas.

---

## Memory Leaks

Um memory leak acontece quando memória é alocada mas nunca liberada — o programa consome cada vez mais RAM até travar ou ser encerrado.

Causas comuns:
- **Manual:** esquecer de chamar `free()`
- **Linguagens com GC:** manter uma referência a dados que você não precisa mais (o GC não consegue coletar se algo ainda aponta para eles)
- **Caches sem eviction:** armazenar dados indefinidamente sem TTL

```
// leak in a GC language — cache grows forever
var cache = map[string][]byte{}

func store(key string, data []byte) {
    cache[key] = data  // never removed → GC never collects it
}
```

---

## Stack vs Heap — Resumo

| | **Stack** | **Heap** |
|---|---|---|
| Velocidade | Instantânea | Mais lenta (alocador + GC) |
| Tamanho | Pequena, por thread | Grande, compartilhada |
| Tempo de vida | Escopo da função | Até ser liberada ou coletada pelo GC |
| Gerenciada por | Automaticamente | Alocador + GC ou programador |
| Fragmentação | Nenhuma | Sim |
| Envolvimento do GC | Nenhum | Sim |

---

## Notas de Design

- Alocações na stack são essencialmente gratuitas — prefira-as quando possível
- Alocações na heap têm custo: tempo do alocador + trabalho futuro de limpeza
- GC não é gratuito — roda na CPU e compete com seu programa
- Memory leaks em servidores de longa duração são fatais — se acumulam ao longo do tempo
- Meça antes de otimizar — profilers mostram exatamente onde as alocações acontecem

---

## Notas por Linguagem

- **Go** → veja [[GO/MemoryManagement]]
- **Redis (C)** → gerenciamento manual de memória via jemalloc, veja [[Performance]]

---

## Notas Relacionadas

- [[Performance]]
- [[Concurrency]]
- [[Persistence]]
