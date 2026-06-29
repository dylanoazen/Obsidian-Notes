# Database — database/sql

Como o Go se conecta a bancos relacionais: pool de conexões, queries, transações e o padrão FOR UPDATE SKIP LOCKED.

Related: [[GO/Context]], [[GO/Interfaces]], [[DistributedSystems/Concurrency]]

---

## O Pacote database/sql

`database/sql` é a interface padrão do Go para bancos relacionais. Você importa um driver separado (MySQL, PostgreSQL, SQLite) que registra a si mesmo:

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql" // driver — importado só pelo efeito colateral
)

db, err := sql.Open("mysql", "user:pass@tcp(host:3306)/dbname")
```

`db` não é uma conexão — é um **pool de conexões**. O Go gerencia as conexões automaticamente.

---

## Query vs Exec

```go
// QueryContext — retorna linhas (SELECT)
rows, err := db.QueryContext(ctx, "SELECT id, nome FROM produtos WHERE ativo = ?", 1)
defer rows.Close()

for rows.Next() {
    var id int
    var nome string
    rows.Scan(&id, &nome)
}

if err := rows.Err(); err != nil { ... } // sempre checar depois do loop

// ExecContext — não retorna linhas (INSERT, UPDATE, DELETE)
result, err := db.ExecContext(ctx, "UPDATE produtos SET ativo = ? WHERE id = ?", 0, 123)
```

---

## Transações

Uma transação agrupa operações — todas acontecem ou nenhuma:

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback() // se algo der errado antes do Commit, desfaz tudo

rows, err := tx.QueryContext(ctx, query, args...)
// ... processa

_, err = tx.ExecContext(ctx, updateQuery, ids...)
if err != nil {
    return err // defer Rollback vai executar
}

return tx.Commit() // sucesso — confirma tudo
```

**Por que `defer tx.Rollback()` mesmo antes do `Commit`?**
Se o código retornar antes do `Commit` por qualquer erro, o `Rollback` garante que o banco não fica em estado intermediário. Chamar `Rollback` depois de `Commit` é no-op — não faz nada.

---

## FOR UPDATE SKIP LOCKED — Worker Concorrente

O padrão mais importante do TrayGo. Permite múltiplos workers pegarem jobs sem duplicação:

```sql
SELECT id, ...
FROM integracaoOff
WHERE concluido = 0
  AND tentativas < 5
FOR UPDATE SKIP LOCKED
LIMIT 10
```

**Como funciona:**
```
Worker 1 → SELECT FOR UPDATE → trava rows 1,2,3
Worker 2 → SELECT FOR UPDATE → vê rows 1,2,3 travadas → SKIP → pega rows 4,5,6
Worker 3 → SELECT FOR UPDATE → pega rows 7,8,9

Resultado: zero duplicação, zero espera entre workers
```

Sem `SKIP LOCKED`, o Worker 2 ficaria bloqueado esperando o Worker 1 terminar. Com `SKIP LOCKED`, ele pula as travadas e continua.

**Diferença de Optimistic Locking:**
- `FOR UPDATE SKIP LOCKED` é pessimista — trava no banco durante a transação
- Optimistic Locking — não trava, detecta conflito na escrita e faz retry
- Aqui o pessimista faz sentido: job já foi lido, queremos garantia, não retry

---

## O Padrão Claim → Process → Mark

O TrayGo usa um ciclo de 3 passos:

```
1. ClaimBatch   → SELECT FOR UPDATE SKIP LOCKED + UPDATE status='processing'
                  (atômico — dentro da mesma transaction)

2. process      → envia para a API da Tray
                  (fora do banco, pode demorar)

3. MarkDone     → UPDATE concluido=1
   MarkFailed   → UPDATE tentativas = tentativas + 1
```

Por que marcar como `processing` no claim? Se o worker morrer entre o claim e o mark, o job fica com `status='processing'`. Um processo de cleanup pode detectar jobs travados há muito tempo e resetá-los.

---

## Scan — Lendo Resultados

`rows.Scan` lê os valores da linha atual para variáveis Go na ordem das colunas do SELECT:

```go
var (
    jobID  int
    nome   string
    preco  float64
)

rows.Scan(&jobID, &nome, &preco)
// ordem deve bater exatamente com o SELECT
```

---

## COALESCE — Valores Default no SQL

Retorna o primeiro valor não-NULL:

```sql
COALESCE(e.descricaoSite, e.descricao)  -- usa descricaoSite, ou descricao se for NULL
COALESCE(e.marca, '')                    -- usa marca, ou string vazia se for NULL
COALESCE(NULLIF(e.inicioOferta, '0000-00-00'), '') -- trata data inválida como vazia
```

O TrayGo usa bastante porque o banco legado tem campos opcionais que podem ser NULL.

---

## Related

- [[GO/Context]]
- [[GO/Interfaces]]
- [[DistributedSystems/Concurrency]]
- [[DistributedSystems/RaceCondition]]

#### My commentaries
-
