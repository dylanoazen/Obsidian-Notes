# TrayGo — Projeto

Worker de sincronização de produtos de um ERP legado para a plataforma Tray.

Related: [[GO/Goroutines]], [[GO/Context]], [[GO/Interfaces]], [[GO/Database]], [[GO/Mutex]]

---

## O que faz

Lê produtos de uma tabela de fila no MariaDB e envia para a API da Tray via HTTP. Roda com múltiplos workers em paralelo.

```
MariaDB (integracaoOff)
        │
        ▼
mariadb.Source.ClaimBatch()  ← FOR UPDATE SKIP LOCKED
        │
        ▼
worker.Worker.Run()          ← 3 goroutines paralelas
        │
        ▼
tray.Client.Send()           ← POST/PUT para a API da Tray
        │
        ▼
MarkDone / MarkFailed        ← atualiza status no banco
```

---

## Estrutura de Pastas

```
trayGo/
  cmd/
    tray-sync/
      main.go           ← entry point, monta as dependências, dispara workers
  internal/
    domain/
      job.go            ← SendJob, JobID, interface ProductSource
      product.go        ← struct Product, Promotion
      sender.go         ← interface Sender
    memory/
      source.go         ← implementação em memória (dev/teste)
      source_test.go    ← testes unitários com goroutines concorrentes
    source/
      mariadb/
        source.go       ← implementação real com banco
    tray/
      client.go         ← implementação real do Sender — HTTP para API Tray
    worker/
      worker.go         ← loop principal de processamento
```

**Convenção Go:** `internal/` só pode ser importado pelo próprio módulo — não é API pública.

---

## Camadas e Responsabilidades

| Camada | Pasta | Responsabilidade |
|---|---|---|
| Domain | `internal/domain` | Contratos (interfaces) e entidades — zero dependências externas |
| Infrastructure | `internal/memory`, `internal/source/mariadb` | Implementações de `ProductSource` |
| Integration | `internal/tray` | Implementação de `Sender` — fala com API externa |
| Application | `internal/worker` | Orquestra source + sender |
| Entry point | `cmd/tray-sync/main.go` | Monta tudo, dispara goroutines |

---

## Fluxo de Dados

```go
// main.go — monta as peças
source := memory.NewSource(jobs)     // ou mariadb.NewSource(db) em produção
sender := &fakeSender{}              // ou tray.NewClient(...) em produção
w := worker.NewWorker(source, sender)

// dispara 3 workers paralelos
for i := 0; i < 3; i++ {
    go func() {
        defer wg.Done()
        w.Run(ctx)
    }()
}
```

---

## Padrões Go Usados no Projeto

### Implicit Interface
`tray.Client` e `fakeSender` implementam `domain.Sender` sem declarar. O `Worker` recebe a interface — não sabe qual implementação está rodando.

### FOR UPDATE SKIP LOCKED
Múltiplos workers lendo a mesma fila sem duplicação. O banco faz o lock — sem Mutex na aplicação para coordenar workers.

### Graceful Shutdown
`signal.NotifyContext` cancela o contexto no Ctrl+C. Cada goroutine checa `ctx.Err()` no loop e para limpo. `wg.Wait()` garante que o main não encerra antes de todas terminarem.

### Retry com Limite
```go
AND tentativas < 5  -- no SQL
// MarkFailed incrementa tentativas
// após 5 falhas, o job para de aparecer no ClaimBatch
```

### POST vs PUT por ExternalID
```go
method := "POST"
url := fmt.Sprintf("%s/products/", c.baseURL)
if externalID != "" {
    method = "PUT"
    url = fmt.Sprintf("%s/products/%s", c.baseURL, externalID)
}
```
Se o produto já existe na Tray (`externalID` preenchido), atualiza. Se não existe, cria.

---

## Decisões Técnicas

**Por que `int64` para preços em vez de `float64`?**
Float tem erro de representação binária. O banco armazena em `float` mas o Go converte para `int64` multiplicando por 100 (centavos) na leitura — `int64(preco4 * 100)`. A API da Tray recebe string formatada: `fmt.Sprintf("%.2f", float64(p.Price)/100)`.

**Por que `domain/` separado com apenas interfaces?**
O domínio não importa nada externo — nem banco, nem HTTP. Qualquer implementação pode satisfazer as interfaces. Facilita troca de banco, troca de plataforma de destino, e teste unitário com implementações fake.

**Por que `memory.Source` existe?**
Para rodar localmente sem banco. Também usado nos testes — `source_test.go` testa concorrência com 3 goroutines usando o source em memória.

---

## Related

- [[GO/Goroutines]]
- [[GO/Context]]
- [[GO/Interfaces]]
- [[GO/Database]]
- [[GO/Mutex]]

#### My commentaries
-
