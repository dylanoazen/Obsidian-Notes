# Concurrency & Ownership

Concurrency é a capacidade de lidar com múltiplas tarefas ao mesmo tempo — ou pelo menos dar a ilusão de fazê-lo.

---

## Hardware Threads vs Software Threads

São duas coisas completamente diferentes que compartilham o mesmo nome — uma fonte comum de confusão.

### Hardware Threads — Fixos

Hardware threads são físicas — determinadas pela sua CPU. Este é o limite real do que executa verdadeiramente em paralelo.

```
4 cores × 2 threads por core (hyper-threading) = 8 hardware threads
```

A qualquer momento, apenas 8 coisas podem realmente rodar ao mesmo tempo nesta máquina. Isso nunca muda — é físico.

### Software Threads — Ilimitados

Software threads são criadas pelo seu programa. Você pode criar quantas quiser:

```go
for i := 0; i < 10000; i++ {
    go handleRequest()  // created 10,000 goroutines
}
```

Um servidor web pode ter milhares de software threads "rodando ao mesmo tempo" em uma máquina com apenas 8 hardware threads.

### The OS Scheduler — O Malabarista

O OS alterna constantemente qual software thread roda em qual hardware thread:

```
Você tem 8 "trilhos"
Você tem 10.000 "trens"

O OS fica trocando qual trem roda em qual trilho —
tão rápido que parece que todos os trens estão se movendo ao mesmo tempo
```

```
Momento 1:  [req1][req2][req3][req4][req5][req6][req7][req8]   ← 8 rodando
req3 vai buscar no DB → dorme, libera o trilho
Momento 2:  [req1][req2][req9][req4][req5][req6][req7][req8]   ← req9 entra
req1 termina → libera o trilho
Momento 3:  [req10][req2][req9][req4][req5][req6][req7][req8]  ← req10 entra
```

### Por que 8 Cores Conseguem Lidar com 10.000 Conexões

A maioria das software threads passa **mais tempo dormindo do que rodando**:

```
req1: received → fetched from DB (slept 10ms) → responded
      |← CPU 0.1ms →|←——— sleeping 10ms ———→|← CPU 0.1ms →|
```

Quando uma thread espera pelo banco de dados, rede ou disco — ela não está usando CPU. O OS coloca outra thread naquele hardware thread enquanto espera. Esse é o insight fundamental por trás de servidores de alta concorrência.

### Parallelism vs Concurrency

| | **Concurrency** | **Parallelism** |
|---|---|---|
| Definição | Lidar com muitas coisas ao mesmo tempo | Fazer muitas coisas ao mesmo tempo |
| Requer | Apenas um scheduler | Múltiplos hardware threads |
| Exemplo | 1 chef equilibrando 10 pedidos | 10 chefs cada um cozinhando 1 pedido |
| Limitado por | Design / scheduler | Hardware |

> Concurrency é sobre estrutura. Parallelism é sobre execução.

---

## Single-thread vs Multi-thread

### Single-thread
O programa tem apenas uma linha de execução. Tudo roda sequencialmente — uma coisa por vez.

```
task 1 → task 2 → task 3 → task 4
```

Se a task 2 levar 5 segundos, as tasks 3 e 4 esperam.

---

### Multi-thread
Múltiplas linhas de execução rodam em paralelo. Cada thread cuida da sua própria tarefa de forma independente.

```
thread 1 → task 1 → task 3
thread 2 → task 2 → task 4
```

**Problema:** threads compartilham memória, o que cria race conditions — duas threads modificando os mesmos dados ao mesmo tempo leva a resultados imprevisíveis.

---

## Event Loop

O event loop é o modelo usado por runtimes single-threaded (como JavaScript/Node.js) para lidar com concorrência sem múltiplas threads.

Em vez de bloquear e esperar, ele registra um callback e segue em frente. Quando o resultado fica pronto, o callback é colocado na fila e executado.

```
┌─────────────────────────────────────────┐
│              Call Stack                 │
│  (executes one thing at a time)         │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│              Event Loop                 │
│  (checks if stack is empty,             │
│   then picks next from queue)           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│            Callback Queue               │
│  (tasks ready to be executed)           │
└─────────────────────────────────────────┘
```

**Como funciona:**
1. O código roda na call stack
2. Quando uma operação assíncrona começa (I/O, timer), ela é delegada
3. A stack continua com a próxima tarefa
4. Quando a operação assíncrona termina, seu callback vai para a fila
5. O event loop o pega quando a stack está vazia

---

## I/O and Blocking

**I/O (Input/Output)** refere-se a operações que se comunicam com algo fora da CPU — disco, rede, banco de dados, teclado.

Essas operações são **lentas** comparadas à execução da CPU. As duas formas de lidar com elas:

### Blocking I/O
A thread para e espera a operação terminar.

```
thread: ──── read file ░░░░░░░░░░░░░░ done ──── continue
                        ↑ waiting, doing nothing
```

Desperdiça tempo de CPU. Comum em servidores tradicionais — uma thread por requisição.

### Non-blocking I/O
A thread inicia a operação e segue em frente. Uma notificação chega quando ela termina.

```
thread: ──── start read ──── other work ──── handle result
                                         ↑ callback/signal
```

Eficiente. Uma thread consegue lidar com milhares de conexões.

---

## A Abordagem do Go: Goroutines

Go não usa um event loop. Em vez disso, usa **goroutines** — threads leves gerenciadas pelo runtime do Go, não pelo OS.

```go
go doSomething() // spawns a goroutine
```

**Diferenças principais em relação às OS threads:**
- Uma goroutine começa com ~2KB de stack (OS thread ~1MB)
- O runtime do Go agenda goroutines nos cores de CPU disponíveis
- Você pode ter milhões de goroutines rodando simultaneamente

```
Go Runtime
┌──────────────────────────────────────┐
│  goroutine 1 → goroutine 4 → ...     │  ← OS thread 1
│  goroutine 2 → goroutine 5 → ...     │  ← OS thread 2
│  goroutine 3 → goroutine 6 → ...     │  ← OS thread 3
└──────────────────────────────────────┘
```

Quando uma goroutine bloqueia em I/O, o runtime a remove da thread e roda outra goroutine no lugar — para que a thread nunca fique ociosa.

---

## Ownership & Data Races

Quando múltiplas goroutines acessam os mesmos dados, uma **race condition** pode ocorrer — ambas leem e escrevem ao mesmo tempo, produzindo resultados imprevisíveis.

```go
// DANGEROUS — race condition
counter := 0
go func() { counter++ }()
go func() { counter++ }()
// result could be 1 or 2
```

### Channels — transferência de ownership
A forma idiomática do Go de compartilhar dados entre goroutines. Em vez de compartilhar memória, você **envia** dados por um channel — transferindo o ownership. Pense como um cano: uma goroutine coloca dados dentro, outra retira.

```go
ch := make(chan int)

go func() {
    ch <- 42 // send — blocks until someone receives
}()

value := <-ch // receive — blocks until something is sent
```

Ambos os lados bloqueiam e esperam um pelo outro — é isso que torna a transferência segura.

Um padrão comum é uma **goroutine coordenadora** que coleta resultados de muitos workers rodando em paralelo (fan-out / fan-in):

```go
ch := make(chan int)

// workers run in parallel
go func() { ch <- processOrder(1) }()
go func() { ch <- processOrder(2) }()
go func() { ch <- processOrder(3) }()

// coordinator collects all results
for i := 0; i < 3; i++ {
    result := <-ch
    fmt.Println("result:", result)
}
```

```
goroutine 1 ──► ch ──┐
goroutine 2 ──► ch ──┼──► coordinator (collects results)
goroutine 3 ──► ch ──┘
```

O coordenador não sabe qual goroutine termina primeiro — apenas espera o próximo resultado chegar. Quem terminar primeiro envia primeiro.

**Isso tem custo de performance?**
Sim — cada envio/recebimento envolve sincronização entre goroutines. Mas o custo é mínimo na prática. O gargalo real é quase sempre o trabalho que as goroutines estão fazendo, não o channel em si.

> "Do not communicate by sharing memory; share memory by communicating." — Provérbio Go

### Channels vs Mutex

| | Channel | Mutex |
|---|---|---|
| Modelo | Transfere os dados | Compartilha os dados |
| Custo | Sincronização na transferência | Lock no acesso |
| Risco | Deadlock se mal utilizado | Race condition se mal utilizado |
| Legibilidade | Alta — fluxo explícito | Baixa — difícil de rastrear |
| Melhor para | Comunicação entre goroutines | Proteger estado compartilhado |

Use **channel** quando goroutines precisam se **comunicar**.
Use **mutex** quando goroutines precisam **compartilhar estado**.

### Mutex — controle de acesso compartilhado
Quando você precisa compartilhar memória, use um mutex para garantir que apenas uma goroutine a acesse por vez.

```go
var mu sync.Mutex
counter := 0

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

---

## Resumo

| Modelo | Thread | I/O | Caso de uso |
|-------|--------|-----|----------|
| Single-thread + Event Loop | 1 | Non-blocking | Node.js, JS |
| Multi-thread | Muitas | Blocking | Servidores tradicionais |
| Goroutines (Go) | Muitas (leves) | Non-blocking | Servidores Go |
