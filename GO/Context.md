# Context

O `context` é o sistema de "desliga tudo" do Go. Quem cria o contexto tem o **interruptor**. Quem recebe o contexto só consegue **ver se foi desligado**.

Related: [[GO/GO]], [[GO/GarbageCollector]], [[Linux/ProcessesAndThreads]]

---

## O Problema que Resolve

Você tem um servidor. Ele recebe um request. O request dispara: uma query no banco, uma chamada HTTP pra API externa, e uma escrita no Redis. Daí o cliente cancela o request (fechou o browser).

**Sem context:** as 3 operações continuam rodando. Gastam CPU, memória, conexões — pra nada.

**Com context:** você cancela o contexto e as 3 operações param. Uma única chamada desliga tudo que tava pendurado naquele request.

## A Analogia do Interruptor

```
main.go cria o contexto ──► tem o INTERRUPTOR (cancel)
    │
    │  passa ctx pra baixo
    ▼
worker.Run(ctx) ──────────► só OBSERVA se desligou
    │
    │  passa ctx pra baixo
    ▼
ClaimBatch(ctx, 10) ──────► só OBSERVA se desligou
    │
    │  passa ctx pra baixo
    ▼
Send(ctx, ...) ───────────► só OBSERVA se desligou
```

Quando `cancel()` é chamado lá em cima, **TODOS** os que receberam esse `ctx` ficam sabendo. É como desligar o disjuntor geral — toda a casa apaga.

## Mas Quem Detecta o Cancelamento?

Você nunca faz polling manual. Sempre tem algo por baixo que detecta o evento e chama `cancel()` por você.

### Browser fechou (HTTP request)

O servidor Go (`net/http`) mantém uma conexão TCP com o browser. Quando o browser fecha, o TCP manda um FIN. O servidor detecta que a conexão morreu e **automaticamente cancela o `ctx` daquele request**.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // ← já vem com cancel automático!
    
    // Se o browser fechar, ctx cancela sozinho
    // Você não precisa fazer nada — o net/http cuida
    result, err := s.DoWork(ctx)
}
```

Você não monitora nada — o `r.Context()` que vem no request já tá plugado na conexão TCP. A stack de rede do Go faz o trabalho sujo.

### Ctrl+C no terminal

O sistema operacional manda um sinal (SIGINT). O Go escuta e chama `cancel()`:

```go
ctx, cancel := signal.NotifyContext(ctx, syscall.SIGINT)
// OS manda SIGINT → Go detecta → cancel() é chamado
```

### Timeout estourou

O runtime do Go tem um timer interno. Passou o tempo? Chama `cancel()` sozinho:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
// Timer interno do Go → estoura → cancel() é chamado
```

### Resumo: quem aperta o interruptor?

| Cenário | Quem detecta | Quem chama cancel() |
|---------|-------------|-------------------|
| Browser fechou | Stack TCP do Go (net/http) | Automático via `r.Context()` |
| Ctrl+C | OS envia SIGINT | `signal.NotifyContext` |
| Timeout | Timer do runtime | `context.WithTimeout` |
| Manual | Sua lógica decide | Você chama `cancel()` |

Seu código nunca precisa ficar "vigiando" — só precisa checar `ctx.Err()` ou escutar `<-ctx.Done()`.

## Os 3 Tipos de Context

### 1. `context.Background()` — o ponto zero

```go
ctx := context.Background()
// Context vazio, sem deadline, sem cancel
// Usado como raiz — o "primeiro da cadeia"
```

Nunca cancela sozinho. É o ponto de partida de onde você cria os outros.

### 2. `context.WithCancel` — interruptor manual

```go
ctx, cancel := context.WithCancel(context.Background())
//  │    │
//  │    └── função que DESLIGA tudo
//  └────── contexto que passa pra frente

// Quando quiser desligar:
cancel()
```

**Quem chama `cancel()`**: o main, o handler do request, quem estiver no controle.
**Quem recebe `ctx`**: workers, funções de I/O, goroutines — observam, não controlam.

### 3. `context.WithTimeout` / `WithDeadline` — interruptor com timer

```go
// Cancela automaticamente depois de 5 segundos
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel() // SEMPRE defer cancel, mesmo com timeout

// Ou com horário exato
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5*time.Second))
```

Se a operação demora mais que 5s, o contexto cancela sozinho. Não precisa de ninguém chamar `cancel()`.

## Como Verificar se Cancelou

Duas formas:

### `ctx.Err()` — checagem pontual

```go
func doWork(ctx context.Context) error {
    for i := 0; i < 1000; i++ {
        // Verifica antes de cada iteração
        if ctx.Err() != nil {
            fmt.Println("cancelado, parando")
            return ctx.Err()
        }
        processItem(i)
    }
    return nil
}
```

`ctx.Err()` retorna:
- `nil` → tá rodando, segue em frente
- `context.Canceled` → alguém chamou `cancel()`
- `context.DeadlineExceeded` → timeout estourou

### `ctx.Done()` — espera bloqueante (com select)

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            // Contexto cancelou — hora de parar
            fmt.Println("shutdown:", ctx.Err())
            return

        case job := <-jobs:
            // Novo job chegou — processa
            process(job)
        }
    }
}
```

`ctx.Done()` retorna um channel que **fecha** quando o contexto é cancelado. O `select` escuta vários channels ao mesmo tempo — quem fechar primeiro ganha.

## Exemplo Completo: Servidor com Graceful Shutdown

```go
func main() {
    // Cria contexto que cancela em SIGINT/SIGTERM (Ctrl+C)
    ctx, cancel := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    // Passa ctx pro worker
    go worker(ctx)

    // Bloqueia até cancelar
    <-ctx.Done()
    fmt.Println("Shutting down...")
}

func worker(ctx context.Context) {
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            fmt.Println("worker: parando")
            return
        case <-ticker.C:
            fmt.Println("worker: processando batch...")
            processBatch(ctx) // passa ctx pra frente!
        }
    }
}

func processBatch(ctx context.Context) {
    for i := 0; i < 10; i++ {
        if ctx.Err() != nil {
            return // cancelou no meio do batch, para
        }
        processItem(i)
    }
}
```

O Ctrl+C chega → `cancel()` é chamado → `worker` detecta via `ctx.Done()` → `processBatch` detecta via `ctx.Err()` → tudo para limpo.

## Propagação pela Cadeia

```go
// TODA função que faz I/O ou demora recebe ctx como PRIMEIRO parâmetro
func (s *Service) Transfer(ctx context.Context, from, to string, amount int) error {
    account, err := s.repo.FindById(ctx, from) // passa ctx pro banco
    if err != nil { return err }
    // ...
}

func (r *PostgresRepo) FindById(ctx context.Context, id string) (*Account, error) {
    row := r.db.QueryRowContext(ctx, "SELECT ...", id) // ctx vai pro driver SQL
    // Se ctx cancelar, a query no banco é ABORTADA
}

func (c *HTTPClient) CallExternal(ctx context.Context, url string) (*Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil) // ctx vai pro HTTP
    // Se ctx cancelar, a request HTTP é ABORTADA
}
```

É convenção em Go: **`ctx` é sempre o primeiro parâmetro**. Se a função faz qualquer coisa que demora (I/O, rede, banco), ela recebe `ctx`.

## Context com Valores (use com parcimônia)

```go
// Guardar metadata no contexto (request ID, user info)
ctx = context.WithValue(ctx, "requestID", "abc-123")

// Ler
reqID := ctx.Value("requestID").(string)
```

⚠️ Usar com cuidado:
- Só pra metadata que cruza camadas (request ID, trace ID, auth token)
- **Nunca** pra passar dados de negócio (use parâmetros normais)
- Não tem type safety — o Value retorna `any`

Padrão melhor com type safety:

```go
type ctxKey string
const requestIDKey ctxKey = "requestID"

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func GetRequestID(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}
```

## Regras de Ouro

1. **`ctx` é o primeiro parâmetro** — `func DoSomething(ctx context.Context, ...)`
2. **Nunca guarde ctx em struct** — passe por parâmetro sempre
3. **Sempre `defer cancel()`** — evita leak de goroutines
4. **Quem cria, cancela** — quem chamou `WithCancel` é responsável por chamar `cancel()`
5. **Propague pra baixo** — toda chamada de I/O recebe o ctx
6. **Não use `context.Background()` no meio** do código — se você tem um ctx, use ele

```go
// ❌ Errado: ignora o contexto do caller
func (s *Service) DoWork(ctx context.Context) {
    s.repo.Find(context.Background(), id) // ← ignora cancelamento!
}

// ✅ Certo: propaga o contexto
func (s *Service) DoWork(ctx context.Context) {
    s.repo.Find(ctx, id)
}
```

## Fluxo Mental Simplificado

```
Pergunta: "Quem tem o interruptor?"
Resposta: quem chamou WithCancel/WithTimeout

Pergunta: "Quem só observa?"
Resposta: todo mundo que recebeu ctx por parâmetro

Pergunta: "Como observo?"
Resposta: ctx.Err() != nil (checagem rápida)
          <-ctx.Done() (espera bloqueante no select)

Pergunta: "Quando passo ctx?"
Resposta: SEMPRE que a função faz I/O, rede, banco, ou demora
```

## Related

- [[GO/GO]]
- [[GO/GarbageCollector]]
- [[GO/MemoryManagement]]
- [[Linux/ProcessesAndThreads]]

## Resources

- https://pkg.go.dev/context
- https://go.dev/blog/context
- https://www.digitalocean.com/community/tutorials/how-to-use-contexts-in-go

#### My commentaries
- 
