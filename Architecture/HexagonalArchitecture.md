# Arquitetura Hexagonal

Também chamada de **Ports and Adapters** (Alistair Cockburn, 2005). O core do sistema não sabe que HTTP, banco de dados ou filas existem.

Related: [[PHP/SOLID]], [[GO/EstruturaDeProjetoGo]]

---

## A Ideia Central

O domínio (regras de negócio) fica no centro, completamente isolado. O mundo externo se conecta a ele através de **ports** (interfaces) e **adapters** (implementações).

```
                    ┌─────────────────────────────┐
                    │        ADAPTERS              │
                    │  (mundo externo)             │
                    │                              │
  HTTP Request ────►│  ┌───────────────────────┐  │
                    │  │      PORTS (in)        │  │
  CLI Command ─────►│  │  (interfaces de        │  │
                    │  │   entrada)              │  │
  Queue Message ───►│  │         │               │  │
                    │  │         ▼               │  │
                    │  │  ┌─────────────┐       │  │
                    │  │  │   DOMAIN    │       │  │
                    │  │  │  (regras de │       │  │
                    │  │  │   negócio)  │       │  │
                    │  │  └──────┬──────┘       │  │
                    │  │         │               │  │
                    │  │         ▼               │  │
                    │  │      PORTS (out)        │  │
                    │  │  (interfaces de         │  │
                    │  │   saída)                │  │
                    │  └───────────┬─────────────┘  │
                    │              │                 │
                    │              ▼                 │
                    │   PostgreSQL / Redis / S3 /    │
                    │   API externa / Email          │
                    └─────────────────────────────────┘
```

## Ports e Adapters — Definição

### Port (Interface)

Um **contrato** que o domínio expõe ou exige. É uma interface pura — sem implementação.

- **Port de entrada (driving)**: como o mundo externo invoca o domínio
- **Port de saída (driven)**: o que o domínio precisa do mundo externo

### Adapter (Implementação)

O código concreto que conecta um port ao mundo real.

- **Adapter de entrada**: HTTP controller, CLI handler, consumer de fila
- **Adapter de saída**: repositório PostgreSQL, client HTTP, client S3

## Exemplo Concreto

### Domain (centro — sem dependências externas)

```go
// domain/account.go
package domain

type Account struct {
    ID      string
    Balance int
}

func (a *Account) Debit(amount int) error {
    if a.Balance < amount {
        return ErrInsufficientFunds
    }
    a.Balance -= amount
    return nil
}

func (a *Account) Credit(amount int) {
    a.Balance += amount
}
```

```php
// Domain/Account.php
class Account {
    public function __construct(
        public readonly string $id,
        private int $balance,
    ) {}

    public function debit(int $amount): void {
        if ($this->balance < $amount) {
            throw new InsufficientFundsException();
        }
        $this->balance -= $amount;
    }

    public function credit(int $amount): void {
        $this->balance += $amount;
    }
}
```

### Port de saída (interface)

```go
// domain/ports.go
package domain

type AccountRepository interface {
    FindById(id string) (*Account, error)
    Save(account *Account) error
}
```

```php
// Domain/AccountRepositoryInterface.php
interface AccountRepositoryInterface {
    public function findById(string $id): ?Account;
    public function save(Account $account): void;
}
```

O domínio **define** a interface. Ele diz "eu preciso de algo que faça isso" sem saber como.

### Port de entrada (use case / service)

```go
// domain/transfer_service.go
package domain

type TransferService struct {
    repo AccountRepository  // depende da interface, não da implementação
}

func NewTransferService(repo AccountRepository) *TransferService {
    return &TransferService{repo: repo}
}

func (s *TransferService) Transfer(from, to string, amount int) error {
    sender, err := s.repo.FindById(from)
    if err != nil { return err }

    receiver, err := s.repo.FindById(to)
    if err != nil { return err }

    if err := sender.Debit(amount); err != nil {
        return err
    }
    receiver.Credit(amount)

    s.repo.Save(sender)
    s.repo.Save(receiver)
    return nil
}
```

### Adapter de saída (implementação concreta)

```go
// adapters/postgres_account_repo.go
package adapters

type PostgresAccountRepo struct {
    db *sql.DB
}

func (r *PostgresAccountRepo) FindById(id string) (*domain.Account, error) {
    row := r.db.QueryRow("SELECT id, balance FROM accounts WHERE id = $1", id)
    // ...
}

func (r *PostgresAccountRepo) Save(account *domain.Account) error {
    _, err := r.db.Exec("UPDATE accounts SET balance = $1 WHERE id = $2",
        account.Balance, account.ID)
    return err
}
```

```go
// adapters/inmemory_account_repo.go (pra testes)
package adapters

type InMemoryAccountRepo struct {
    accounts map[string]*domain.Account
}

func (r *InMemoryAccountRepo) FindById(id string) (*domain.Account, error) {
    a, ok := r.accounts[id]
    if !ok { return nil, domain.ErrAccountNotFound }
    return a, nil
}
```

### Adapter de entrada (HTTP)

```go
// adapters/http_handler.go
package adapters

type TransferHandler struct {
    service *domain.TransferService
}

func (h *TransferHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var req TransferRequest
    json.NewDecoder(r.Body).Decode(&req)

    err := h.service.Transfer(req.From, req.To, req.Amount)
    if err != nil {
        // mapeia erro de domínio → HTTP status
        http.Error(w, err.Error(), mapErrorToStatus(err))
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

### Composition Root (onde tudo se conecta)

```go
// cmd/api/main.go
func main() {
    db := connectDB()
    repo := adapters.NewPostgresAccountRepo(db)     // adapter de saída
    service := domain.NewTransferService(repo)        // domain service
    handler := adapters.NewTransferHandler(service)   // adapter de entrada

    http.Handle("/transfer", handler)
    http.ListenAndServe(":8080", nil)
}
```

**Só o `main.go` sabe que PostgreSQL e HTTP existem.** O domínio é puro.

## Direção das Dependências

```
Adapters de entrada ──► Domain ◄── Adapters de saída
(HTTP, CLI, Queue)      (puro)     (Postgres, Redis, S3)

        Tudo aponta pra dentro.
        Domain não importa nada externo.
```

Essa é a **Dependency Rule**: dependências sempre apontam para o centro.

## Estrutura de Pastas

### Go

```
projeto/
├── cmd/api/main.go              # composition root
├── internal/
│   ├── domain/                  # entidades, services, ports (interfaces)
│   │   ├── account.go
│   │   ├── transfer_service.go
│   │   └── ports.go
│   └── adapters/                # implementações dos ports
│       ├── postgres_repo.go
│       ├── inmemory_repo.go
│       └── http_handler.go
└── go.mod
```

### PHP (Laravel)

```
app/
├── Domain/                      # puro, sem framework
│   ├── Account.php
│   ├── TransferService.php
│   └── AccountRepositoryInterface.php
├── Infrastructure/              # adapters de saída
│   ├── EloquentAccountRepository.php
│   └── RedisAccountRepository.php
├── Http/Controllers/            # adapters de entrada
│   └── TransferController.php
└── Providers/
    └── AppServiceProvider.php   # composition root (bindings)
```

## Benefícios

- **Testabilidade**: troca Postgres por InMemory e testa sem banco
- **Troca de tecnologia**: muda de MySQL pra Redis sem tocar no domínio
- **Clareza**: olha a pasta e sabe onde cada coisa vive
- **Framework-agnostic**: o domínio funciona com Laravel, Symfony, ReactPHP ou CLI

## Quando NÃO Usar

- CRUD simples sem regras de negócio — overengineering
- Scripts únicos / ferramentas descartáveis
- Protótipos rápidos (valide primeiro, arquitete depois)

## Hexagonal vs Clean vs Onion

Todos seguem a mesma ideia central (domain no meio, dependências pra dentro):

| Arquitetura | Autor | Diferença principal |
|------------|-------|-------------------|
| Hexagonal | Cockburn | Ports & Adapters, foco em testabilidade |
| Clean | Uncle Bob | Mais camadas (Entities, Use Cases, Interface Adapters) |
| Onion | Palermo | Camadas concêntricas, foco em DDD |

Na prática, a diferença é nomenclatura. O princípio é o mesmo: **isola o domínio**.

## Related

- [[PHP/SOLID]]
- [[PHP/Testing]]
- [[GO/EstruturaDeProjetoGo]]
- [[PHP/EventLoop]]
- [[PHP/Persistence]]

## Resources

- https://alistair.cockburn.us/hexagonal-architecture/
- https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749
- https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/

#### My commentaries
- 
