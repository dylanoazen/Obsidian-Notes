# Race Condition

Quando dois processos acessam o mesmo dado ao mesmo tempo e o resultado depende de quem chega primeiro.

Related: [[DistributedSystems/Concurrency]], [[Finance/ProcessamentoDuplicado]], [[Finance/CoreBanking]]

---

## O Problema

Race condition acontece quando:
1. Dois processos leem o mesmo valor
2. Ambos calculam baseados no valor lido
3. Ambos escrevem de volta
4. Um sobrescreve o trabalho do outro

```
conta A tem saldo = 100

processo 1: lê saldo (100) → calcula 100 - 80 = 20 → escreve 20
processo 2: lê saldo (100) → calcula 100 - 90 = 10 → escreve 10

resultado final: saldo = 10
esperado:        saldo = -70 (ou erro de saldo insuficiente)

processo 1 sacou 80 "de graça" — saldo nunca ficou negativo mas dinheiro sumiu
```

---

## Em Sistemas Financeiros

Race condition em pagamentos pode causar:

- **Saldo negativo** — dois saques simultâneos aprovados para o mesmo saldo
- **Duplo crédito** — depósito processado duas vezes
- **Transferência inconsistente** — débito sem crédito correspondente

```
conta com saldo R$ 100

saque 1 (R$ 80): lê 100, aprova, debita → saldo 20
saque 2 (R$ 90): lê 100 (antes do saque 1 gravar), aprova, debita → saldo 10

resultado: cliente sacou R$ 170 de uma conta com R$ 100
```

---

## Soluções

### 1. SELECT FOR UPDATE — "placa de ocupado no banheiro"

Você entra no banheiro e coloca a placa "ocupado". Ninguém mais consegue entrar. Quando sai, tira a placa.

Enquanto o saque 1 está processando, o saque 2 tenta ler e fica **parado esperando**. Quando o saque 1 termina e libera, o saque 2 lê o saldo atualizado — R$20 — e reprova por saldo insuficiente.

```sql
BEGIN;

SELECT balance FROM accounts WHERE id = '100' FOR UPDATE;
-- placa colocada — ninguém mais lê essa linha

UPDATE accounts SET balance = balance - 80 WHERE id = '100';

COMMIT; -- placa tirada
```

Simples e confiável. Custo: se tiver 1000 saques simultâneos na mesma conta, 999 ficam na fila esperando. Funciona, mas gargala em alta concorrência.

### 2. Operação Atômica — "deixa o banco calcular"

Em vez de você ler o saldo, calcular e gravar — você manda o banco fazer tudo de uma vez. É como pedir para o caixa debitar direto — você não pega o dinheiro, conta e devolve o troco. O caixa faz tudo em uma tacada só.

```sql
-- em vez de calcular na aplicação:
UPDATE accounts SET balance = 20 WHERE id = '100'

-- deixa o banco calcular e validar:
UPDATE accounts SET balance = balance - 80
WHERE id = '100' AND balance >= 80
-- o banco faz leitura, verificação e escrita atomicamente
-- se balance < 80, zero linhas afetadas → aplicação detecta e lança erro
```

Mais eficiente que SELECT FOR UPDATE — sem lock explícito. Limitação: só funciona para operações simples. Lógica complexa não cabe num UPDATE só.

### 3. Optimistic Locking — "quem chegar primeiro ganha"

Adiciona um campo `version` na conta. Toda atualização incrementa a versão. É como o Google Docs — avisa se alguém editou antes de você. Você recarrega e tenta de novo.

```sql
-- lê: saldo=100, version=5
SELECT balance, version FROM accounts WHERE id = '100'

-- saque 1 chega primeiro:
UPDATE accounts SET balance = 20, version = 6
WHERE id = '100' AND version = 5
-- funcionou — version era 5, agora é 6

-- saque 2 tenta:
UPDATE accounts SET balance = 10, version = 6
WHERE id = '100' AND version = 5
-- falhou — version já é 6, não é mais 5
-- zero linhas afetadas → aplicação detecta e faz retry
```

Sem fila, sem espera — mais rápido. Mas se tiver muita concorrência na mesma conta, fica retrying o tempo todo. Bom quando conflitos são raros.

### 4. Lock Distribuído (Redis) — "senha do banco"

Para sistemas sem banco compartilhado. É como a senha do banco — você pega a senha, ninguém mais pode ser atendido no mesmo guichê até você terminar e devolver.

```php
$lock = $redis->set("lock:account:100", 1, ['NX', 'EX' => 5]);
// NX = só seta se não existir (atômico — sem race condition no próprio lock)
// EX = expira em 5s se o processo morrer sem liberar (senha volta automático)

if (!$lock) {
    throw new Exception('Account locked, try again');
}

try {
    // processa com segurança
} finally {
    $redis->del("lock:account:100");
}
```

Custo: depende do Redis estar disponível. Se o Redis cair, ninguém consegue o lock — sistema para.

### 5. Event Loop Single-Thread (ReactPHP / Node.js)

Sem threads concorrentes = sem race condition na memória. Fila única — só um por vez, impossível conflitar.

```
event loop executa um callback por vez
→ saque 1 começa e termina completamente
→ só então saque 2 começa
→ impossível ter dois saques simultâneos no mesmo processo
```

É o que garante atomicidade no projeto EBANX sem precisar de lock explícito.

---

## Resumo com Analogia

| Solução | Analogia | Quando |
|---|---|---|
| **SELECT FOR UPDATE** | Placa de ocupado no banheiro — outros esperam do lado de fora | Mesmo banco, conflitos frequentes |
| **Operação atômica** | Caixa do banco faz tudo em uma tacada — sem intermediário | Mesmo banco, operação simples |
| **Optimistic Locking** | Google Docs — avisa se alguém editou antes de você | Mesmo banco, conflitos raros |
| **Lock no Redis** | Senha do banco — quem tem a senha é atendido | Sistema distribuído |
| **Single-thread** | Fila única — só um por vez, impossível conflitar | ReactPHP, Node.js |

---

## Race Condition vs Deadlock

| | Race Condition | Deadlock |
|---|---|---|
| O que é | Dois processos corrompem dado compartilhado | Dois processos esperam um pelo outro para sempre |
| Resultado | Dado incorreto | Sistema travado |
| Causa | Falta de sincronização | Lock em ordem errada |
| Solução | Lock, atomic ops, single-thread | Lock em ordem consistente, timeout |

---

## Detectando em Go

```bash
go run -race ./cmd/server
go test -race ./...
```

O race detector do Go instrumenta o código e reporta acessos concorrentes sem sincronização.

---

## Related

- [[DistributedSystems/Concurrency]]
- [[Finance/ProcessamentoDuplicado]]
- [[Finance/CoreBanking]]
- [[GO/Mutex]]
- [[DistributedSystems/Redis]]

#### My commentaries
-
