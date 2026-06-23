# Core Banking

O sistema central de um banco — o motor que processa todas as operações financeiras.

Related: [[Finance/FinanceEng]], [[Finance/MeiosDePagamento]], [[DistributedSystems/Idempotency]]

---

## O que é

Core Banking é o sistema de back-office que mantém o estado financeiro de todas as contas. Tudo passa por ele:

```
App do cliente
      │
      ▼
API / Gateway
      │
      ▼
Core Banking  ←── aqui: contas, saldos, transações, regras
      │
      ▼
Banco de dados / Ledger
```

Qualquer operação — depósito, saque, transferência, pagamento — é registrada e processada pelo core banking.

---

## Responsabilidades

- **Ledger** — registro imutável de todas as transações (o livro-caixa)
- **Contas** — criação, manutenção de saldos, status
- **Transferências** — entre contas internas e externas
- **Liquidação** — garantir que o dinheiro "saiu de um lugar e chegou em outro"
- **Reconciliação** — comparar registros internos com os do banco central / parceiros
- **Compliance** — regras regulatórias (KYC, AML, limites)

---

## Ledger — O Coração

O ledger é um log append-only de transações. Nunca se apaga, nunca se edita — só se adiciona.

```
entry 1: account=100, type=credit, amount=1000, balance_after=1000
entry 2: account=100, type=debit,  amount=200,  balance_after=800
entry 3: account=200, type=credit, amount=200,  balance_after=200
```

Saldo atual = soma de todas as entradas da conta. Se der diferente, tem bug ou fraude.

### Partidas Dobradas (Double-entry bookkeeping)

Para cada débito existe um crédito equivalente. O total do sistema é sempre zero:

```
Transferência de R$100 da conta A para B:
  DÉBITO  conta A  R$100   (saiu)
  CRÉDITO conta B  R$100   (entrou)
  ──────────────────────
  SALDO LÍQUIDO    R$0     ← sempre equilibrado
```

Isso garante que dinheiro não some nem apareça do nada.

---

## Desafios Técnicos

### Atomicidade
Uma transferência é duas operações — débito e crédito. As duas têm que acontecer ou nenhuma. Se o sistema cair no meio, o dinheiro não pode sumir.

**1. Transações de banco de dados**

O mais simples — o banco garante atomicidade nativamente:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'origin';
UPDATE accounts SET balance = balance + 100 WHERE id = 'destination';
COMMIT; -- os dois acontecem juntos, ou ROLLBACK automático se falhar
```

Funciona quando origem e destino estão no mesmo banco. Não funciona em sistemas distribuídos onde cada serviço tem seu próprio banco.

**2. Eventos de domínio**

Em vez de fazer as duas operações diretamente, registra o que aconteceu e outros serviços reagem:

```
TransferService
  ├── debita origem
  └── publica evento "transfer_debited" no Kafka

DestinationService (escuta o evento)
  └── credita destino
```

Se o crédito falhar, o evento fica na fila — o sistema tenta de novo até conseguir. É **consistência eventual** — por milissegundos a origem foi debitada mas o destino ainda não recebeu. O sistema se corrige sozinho.

Cada mensagem tem idempotency key para não creditar duas vezes no retry.

**3. Saga Pattern**

Para transações longas que passam por múltiplos serviços. Sequência de operações onde cada uma tem uma **compensação** — o desfazer:

```
passo 1: debita origem       → compensação: credita de volta
passo 2: converte moeda      → compensação: desfaz conversão
passo 3: envia via SWIFT     → compensação: solicita devolução
passo 4: credita destino     → compensação: debita de volta
```

Se o passo 3 falhar, executa as compensações em ordem inversa — estado volta ao início. Não tem ROLLBACK automático do banco, você escreve as compensações manualmente.

**Quando usar cada um:**

| Cenário | Solução |
|---|---|
| Tudo no mesmo banco | Transação SQL |
| Dois serviços, pode ser eventual | Eventos de domínio |
| Múltiplos serviços, fluxo longo | Saga Pattern |
| Single-thread (ReactPHP, Node.js) | Nenhum — event loop garante |

### Idempotência
Cliente reenvia o mesmo pagamento por timeout. O sistema precisa detectar e não processar duas vezes.

→ [[DistributedSystems/Idempotency]]

### Race Condition
Dois saques simultâneos da mesma conta podem resultar em saldo negativo.

```
conta com saldo R$ 100

saque 1 (R$ 80): lê 100 → aprova → debita → saldo 20
saque 2 (R$ 90): lê 100 (antes do saque 1 gravar) → aprova → debita → saldo 10

resultado: cliente sacou R$ 170 de uma conta com R$ 100
```

Ambos leram o mesmo saldo antes de qualquer um gravar — os dois viram R$100, os dois aprovaram.

**Solução 1 — SELECT FOR UPDATE**

Trava o registro no banco enquanto processa. Outros processos que tentarem ler essa linha ficam na fila esperando:

```sql
BEGIN;

SELECT balance FROM accounts WHERE id = '100' FOR UPDATE;
-- nenhum outro processo consegue ler essa linha até o COMMIT

UPDATE accounts SET balance = balance - 80 WHERE id = '100';

COMMIT; -- lock liberado
```

Simples e confiável. Custo: processos ficam na fila — reduz throughput em alta concorrência.

**Solução 2 — Operação atômica no banco**

Delega o cálculo ao banco — elimina o "lê, calcula, escreve" na aplicação:

```sql
-- em vez de calcular na aplicação:
UPDATE accounts SET balance = 20 WHERE id = '100'

-- deixa o banco calcular e validar:
UPDATE accounts SET balance = balance - 80
WHERE id = '100' AND balance >= 80
-- se balance < 80, zero linhas afetadas → aplicação detecta e lança erro
```

Mais eficiente que SELECT FOR UPDATE — sem lock explícito.

**Solução 3 — Optimistic Locking**

Adiciona um campo `version`. A atualização só acontece se a versão não mudou desde que leu:

```sql
-- lê: saldo=100, version=5
SELECT balance, version FROM accounts WHERE id = '100'

-- tenta atualizar só se version ainda é 5
UPDATE accounts SET balance = 20, version = 6
WHERE id = '100' AND version = 5

-- se outro processo atualizou antes: version já é 6
-- UPDATE afeta 0 linhas → aplicação detecta e faz retry
```

Sem lock, sem espera — só detecta conflito e tenta de novo. Bom quando conflitos são raros.

**Solução 4 — Lock distribuído (Redis)**

Para sistemas sem banco compartilhado:

```php
$lock = $redis->set("lock:account:100", 1, ['NX', 'EX' => 5]);
// NX = só seta se não existir (atômico)
// EX = expira em 5s se o processo morrer sem liberar

if (!$lock) {
    throw new Exception('Account locked, try again');
}

try {
    // processa com segurança
} finally {
    $redis->del("lock:account:100");
}
```

**Quando usar cada solução:**

| Cenário | Solução |
|---|---|
| Mesmo banco, conflitos frequentes | SELECT FOR UPDATE |
| Mesmo banco, operação simples | Operação atômica (balance - amount) |
| Mesmo banco, conflitos raros | Optimistic locking |
| Sistema distribuído, sem banco compartilhado | Lock distribuído (Redis) |
| Single-thread (ReactPHP, Node.js) | Nenhum — impossível ter concorrência |

→ [[DistributedSystems/RaceCondition]] — detalhe completo
→ [[DistributedSystems/Concurrency]]

### Consistência eventual
Em sistemas distribuídos, a réplica pode ter saldo desatualizado por milissegundos.

→ [[DistributedSystems/Replication]]

---

## Core Banking Moderno vs Legado

| | Legado (mainframe) | Moderno |
|---|---|---|
| Arquitetura | Monolito | Microsserviços |
| Linguagem | COBOL | Java, Go, Kotlin |
| Deploy | Semanal/mensal | Contínuo |
| Disponibilidade | Janela de manutenção | 99.99% uptime |
| Exemplos | Banco do Brasil, Itaú antigo | Nubank, Revolut, Stripe |

Nubank famosamente reescreveu o core banking em Clojure + Datomic, rodando na AWS — saindo do modelo mainframe.

---

## Related

- [[Finance/FinanceEng]]
- [[Finance/MeiosDePagamento]]
- [[DistributedSystems/Idempotency]]
- [[DistributedSystems/Concurrency]]
- [[DistributedSystems/Replication]]
- [[DistributedSystems/TransactionsPipeline]]

#### My commentaries
-
