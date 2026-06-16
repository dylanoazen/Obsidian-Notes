# Persistência e Durabilidade

Como tornar dados duráveis — ACID, transações, locking e estratégias de migração.

Related: [[PHP/Concurrency]], [[SoftwareEngineering/SOLID]], [[DistributedSystems/Persistence]]

---

## De In-Memory pra Banco — A Pergunta

> "Como você faria isso sobreviver a um restart?"

A resposta: **trocar a implementação do Repository**. Se o código seguiu DIP ([[SoftwareEngineering/SOLID]]), é plug-and-play:

```php
// Antes (in-memory, pra desafio)
$repo = new InMemoryAccountRepository();

// Depois (PostgreSQL, pra produção)
$repo = new PostgresAccountRepository($pdo);

// O AccountService não muda NADA
$service = new AccountService($repo);
```

## ACID — As 4 Garantias

| Propriedade | Significado |
|------------|-------------|
| **Atomicity** | Tudo ou nada — se uma parte falha, tudo é revertido |
| **Consistency** | Banco vai de um estado válido pra outro estado válido |
| **Isolation** | Transações concorrentes não se enxergam |
| **Durability** | Depois do COMMIT, dado sobrevive a crash |

## Transações na Prática

```php
class PostgresAccountRepository implements AccountRepositoryInterface {
    
    public function transfer(string $from, string $to, int $amount): void {
        $this->pdo->beginTransaction();
        
        try {
            // Lock pessimista — trava as rows
            $stmt = $this->pdo->prepare(
                'SELECT * FROM accounts WHERE id IN (?, ?) FOR UPDATE'
            );
            $stmt->execute([$from, $to]);
            
            // Valida saldo
            $sender = $this->findById($from);
            if ($sender->balance < $amount) {
                throw new InsufficientFundsException();
            }
            
            // Aplica
            $this->pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
                       ->execute([$amount, $from]);
            $this->pdo->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?')
                       ->execute([$amount, $to]);
            
            $this->pdo->commit();
            
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

## Isolation Levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| Read Uncommitted | ✅ possível | ✅ possível | ✅ possível |
| Read Committed | ❌ | ✅ possível | ✅ possível |
| Repeatable Read | ❌ | ❌ | ✅ possível |
| Serializable | ❌ | ❌ | ❌ |

PostgreSQL default: **Read Committed**. Pra operações financeiras, considerar **Serializable** ou **SELECT FOR UPDATE**.

## Migrations

```php
// Laravel migration
Schema::create('accounts', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->bigInteger('balance')->default(0);
    $table->char('currency', 3)->default('BRL');
    $table->timestamps();
    
    $table->index('currency');
});
```

```bash
php artisan make:migration create_accounts_table
php artisan migrate
php artisan migrate:rollback
```

## Event Sourcing (Menção)

Em vez de guardar o estado atual, guardar **todos os eventos**:

```
Event 1: AccountCreated { id: A, balance: 0 }
Event 2: Deposited { id: A, amount: 1000 }
Event 3: Transferred { from: A, to: B, amount: 300 }
Event 4: Deposited { id: A, amount: 500 }

Estado atual de A = replay dos eventos = 0 + 1000 - 300 + 500 = 1200
```

- Audit trail completo (importante em fintech)
- Pode reconstruir estado em qualquer ponto no tempo
- Mais complexo, mas relevante porque o endpoint se chama `/event`

> "Não implementei event sourcing, mas o design do endpoint como /event sugere essa direção. Em produção, cada operação financeira deveria ser um evento imutável num log, o que dá audit trail completo e possibilidade de replay."

## Na Entrevista

> "Meu design já permite trocar o repositório in-memory por PostgreSQL sem mudar o service. Em produção, usaria transações com SELECT FOR UPDATE pra garantir atomicidade com concorrência real, e ACID do banco como garantia de durabilidade."

## Related

- [[SoftwareEngineering/SOLID]]
- [[PHP/Concurrency]]
- [[SoftwareEngineering/MoneyPattern]]
- [[DistributedSystems/Idempotency]]
- [[DistributedSystems/Persistence]]

#### My commentaries
- 
