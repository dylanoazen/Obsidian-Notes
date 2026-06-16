# Concorrência e Atomicidade

Como garantir que operações financeiras são seguras quando múltiplos requests chegam ao mesmo tempo.

Related: [[PHP/EventLoop]], [[PHP/Persistence]], [[Linux/ProcessesAndThreads]]

---

## No Event Loop Single-Threaded (ReactPHP)

```
Request A (transfer) ──►┐
                         │ Event loop: processa UM por vez
Request B (transfer) ──►┘ B espera A terminar
```

- **Não há concorrência real** dentro do processo
- Cada callback roda até o fim antes do próximo começar
- Operação de transfer é **naturalmente atômica** — ninguém interrompe no meio

> "No event loop single-threaded, a operação é naturalmente atômica porque não existe preempção entre callbacks. O débito é validado e aplicado antes do crédito, então se falhar, não fica crédito órfão."

## Quando a Concorrência Aparece

Se escalar pra múltiplos processos ou usar banco de dados:

```
Processo 1: lê saldo = 100
Processo 2: lê saldo = 100    ← lê o mesmo valor!
Processo 1: debita 80 → saldo = 20
Processo 2: debita 80 → saldo = 20   ← deveria ter falhado!
```

Isso é uma **race condition** — o saldo ficou inconsistente.

## Soluções para Concorrência Real

### 1. Pessimistic Locking (SELECT ... FOR UPDATE)

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 'abc' FOR UPDATE;
-- Outras transações que tentarem ler essa row ficam BLOQUEADAS
UPDATE accounts SET balance = balance - 80 WHERE id = 'abc';
COMMIT;
```

- Trava a row até o COMMIT
- Garante que ninguém lê valor desatualizado
- Mais lento (bloqueio), mas mais simples

### 2. Optimistic Locking (version/timestamp)

```sql
-- Lê com versão
SELECT balance, version FROM accounts WHERE id = 'abc';
-- balance=100, version=5

-- Atualiza só se versão não mudou
UPDATE accounts 
SET balance = 20, version = 6
WHERE id = 'abc' AND version = 5;
-- Se affected_rows = 0 → alguém mudou antes → retry
```

- Sem bloqueio na leitura
- Mais rápido em cenários de baixa contenção
- Precisa de lógica de retry

### 3. Operações Atômicas (banco)

```sql
-- Atômico: débito só acontece se saldo suficiente
UPDATE accounts 
SET balance = balance - 80 
WHERE id = 'abc' AND balance >= 80;

-- Se affected_rows = 0 → saldo insuficiente
```

### Comparação

| Estratégia | Lock | Performance | Complexidade | Quando usar |
|-----------|------|------------|-------------|-------------|
| Pessimistic | Sim | Mais lento | Simples | Alta contenção |
| Optimistic | Não | Mais rápido | Média (retry) | Baixa contenção |
| Atômico DB | Não | Rápido | Simples | Operações simples |

## Débito Antes do Crédito — Por Quê?

```php
// ✅ Correto: débito primeiro
$sender->debit($amount);    // Pode falhar (saldo insuficiente)
$receiver->credit($amount); // Só executa se débito passou

// ❌ Errado: crédito primeiro
$receiver->credit($amount); // Funciona sempre
$sender->debit($amount);    // Se falhar → dinheiro criado do nada!
```

Em caso de falha no meio da operação:
- Débito primeiro → no pior caso, dinheiro sumiu (reversível, auditável)
- Crédito primeiro → dinheiro criado do nada (fraude, irrecuperável)

## Related

- [[PHP/EventLoop]]
- [[PHP/Persistence]]
- [[DistributedSystems/Idempotency]]
- [[DistributedSystems/TransactionsPipeline]]

#### My commentaries
- 
