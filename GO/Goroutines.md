# Goroutines

Unidades de concorrência do Go — leves, gerenciadas pelo runtime, não pelo OS.

Related: [[GO/Mutex]], [[GO/Context]], [[GO/GO]]

---

## O que é uma Goroutine

Uma goroutine é uma função executando de forma concorrente. Você inicia com `go`:

```go
go func() {
    worker.Run(ctx)
}()
```

Diferente de uma thread do OS, goroutines são baratas — milhares podem existir ao mesmo tempo. O runtime do Go faz o scheduling delas sobre as threads reais.

```
Thread OS  (cara): ~1MB de stack, pesada para criar
Goroutine  (Go):   ~2KB de stack inicial, cresce conforme necessário
```

---

## sync.WaitGroup — Esperar Goroutines Terminarem

Quando você dispara goroutines, o `main` não espera por padrão — ele encerra e mata tudo. O `WaitGroup` resolve isso:

```go
var wg sync.WaitGroup
wg.Add(3) // avisa que vai esperar 3 goroutines

for i := 0; i < 3; i++ {
    go func() {
        defer wg.Done() // sinaliza quando terminar
        worker.Run(ctx)
    }()
}

wg.Wait() // bloqueia até as 3 chamarem Done()
```

**Regra:** `Add` antes de iniciar a goroutine. `Done` sempre com `defer` para garantir que chama mesmo se panitar.

---

## Worker Pool — Padrão do TrayGo

O TrayGo usa o padrão mais simples de worker pool: N goroutines rodando a mesma função concorrentemente.

```go
// main.go
wg.Add(3)
for i := 0; i < 3; i++ {
    go func() {
        defer wg.Done()
        worker.Run(ctx) // todos competem pela mesma fila
    }()
}
wg.Wait()
```

```
goroutine 1 ──► worker.Run ──► ClaimBatch ──► jobs [1,2]
goroutine 2 ──► worker.Run ──► ClaimBatch ──► jobs [3]
goroutine 3 ──► worker.Run ──► ClaimBatch ──► []  (vazio, dorme)
```

Cada goroutine pede um lote, processa, pede mais. O `Mutex` na `Source` (ou `FOR UPDATE SKIP LOCKED` no banco) garante que duas goroutines não peguem o mesmo job.

---

## Loop do Worker

O loop interno de cada goroutine:

```go
func (w *Worker) Run(ctx context.Context) error {
    for {
        if ctx.Err() != nil { // contexto cancelado? para.
            return nil
        }

        jobs, err := w.jobSource.ClaimBatch(ctx, 10)
        if len(jobs) == 0 {
            time.Sleep(500 * time.Millisecond) // fila vazia, espera
            continue
        }

        for _, job := range jobs {
            err := w.sender.Send(ctx, job.ExternalID, job.Product)
            if err != nil {
                w.jobSource.MarkFailed(ctx, job.ID, err.Error())
                continue
            }
            w.jobSource.MarkDone(ctx, job.ID, job.ExternalID)
        }
    }
}
```

---

## Goroutine Leak

Se uma goroutine nunca termina — nem por contexto nem por erro — ela vaza. O processo continua com goroutines mortas acumulando memória.

**Prevenir:** sempre dar um caminho de saída via `ctx.Done()` ou canal fechado.

```go
// sem saída — leak
go func() {
    for {
        process()
    }
}()

// com saída — correto
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            process()
        }
    }
}()
```

---

## Related

- [[GO/Mutex]]
- [[GO/Context]]
- [[GO/GO]]
- [[GO/Semantics]]

#### My commentaries
-
