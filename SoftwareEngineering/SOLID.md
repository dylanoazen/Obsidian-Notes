# SOLID e Injeção de Dependência

Separação de camadas, inversão de dependência e como tornar código testável.

Related: [[PHP/PHP]], [[SoftwareEngineering/Testing]]

---

## Os 5 Princípios SOLID

### S — Single Responsibility Principle (SRP)

Uma classe tem apenas **um motivo para mudar**.

```php
// ❌ Ruim — faz tudo
class AccountController {
    public function transfer(Request $request): Response {
        $data = json_decode($request->getBody());
        // valida input
        // busca accounts no banco
        // aplica regras de negócio
        // persiste
        // formata response
    }
}

// ✅ Bom — cada classe uma responsabilidade
class TransferController {      // só HTTP
    public function __invoke(Request $request): Response {
        $dto = TransferRequest::fromHttp($request);
        $result = $this->service->transfer($dto);
        return new JsonResponse($result, 200);
    }
}

class AccountService {          // só regras de negócio
    public function transfer(TransferRequest $dto): TransferResult { }
}

class AccountRepository {       // só persistência
    public function findById(string $id): ?Account { }
}
```

### O — Open/Closed Principle

Aberto pra extensão, fechado pra modificação. Usar interfaces e composição.

### L — Liskov Substitution

Subtipos devem ser substituíveis pelo tipo base sem quebrar o programa.

### I — Interface Segregation

Interfaces pequenas e específicas, não interfaces "gordas".

### D — Dependency Inversion Principle (DIP)

**O mais importante pra entrevista.**

Módulos de alto nível NÃO dependem de módulos de baixo nível. Ambos dependem de **abstrações**.

```php
// ❌ Service depende de implementação concreta
class AccountService {
    private MySQLAccountRepo $repo; // acoplado ao MySQL
}

// ✅ Service depende de abstração (interface)
interface AccountRepositoryInterface {
    public function findById(string $id): ?Account;
    public function save(Account $account): void;
}

class AccountService {
    public function __construct(
        private readonly AccountRepositoryInterface $repo
    ) {}
    // Não sabe nem importa se é MySQL, Redis, ou in-memory
}

// Implementações
class InMemoryAccountRepository implements AccountRepositoryInterface { }
class MySQLAccountRepository implements AccountRepositoryInterface { }
class RedisAccountRepository implements AccountRepositoryInterface { }
```

## Injeção de Dependência (DI)

Dependências são **passadas de fora**, não criadas internamente:

```php
// ❌ Cria dependência dentro — impossível testar
class AccountService {
    public function __construct() {
        $this->repo = new MySQLAccountRepository();
    }
}

// ✅ Recebe dependência de fora — testável
class AccountService {
    public function __construct(
        private readonly AccountRepositoryInterface $repo
    ) {}
}

// Em produção:
$service = new AccountService(new MySQLAccountRepository($pdo));

// Em testes:
$service = new AccountService(new InMemoryAccountRepository());
```

## Arquitetura em Camadas

```
┌──────────────────────────────────┐
│  Controller (HTTP)               │  ← Recebe request, retorna response
│  Fino: só parse + delegate       │
├──────────────────────────────────┤
│  Service (Business Logic)        │  ← Regras de negócio, validação
│  Não sabe nada de HTTP           │
├──────────────────────────────────┤
│  Repository (Data Access)        │  ← Busca e persiste dados
│  Interface no topo, impl embaixo │
├──────────────────────────────────┤
│  Model / Entity (Domain)         │  ← Estrutura de dados + comportamento
└──────────────────────────────────┘
```

O fluxo de dependência: Controller → Service → Repository (interface).
A implementação do Repository é injetada — **nunca importada diretamente**.

## Na Prática (AccountService)

```php
class AccountService {
    public function __construct(
        private readonly AccountRepositoryInterface $repository
    ) {}

    public function transfer(string $from, string $to, int $amount): void {
        $sender = $this->repository->findById($from)
            ?? throw new AccountNotFoundException($from);
        
        $receiver = $this->repository->findById($to)
            ?? throw new AccountNotFoundException($to);

        if ($sender->balance < $amount) {
            throw new InsufficientFundsException($from, $amount);
        }

        // Débito antes do crédito — se falhar, não fica crédito órfão
        $sender->debit($amount);
        $receiver->credit($amount);

        $this->repository->save($sender);
        $this->repository->save($receiver);
    }
}
```

O service **não sabe** se está rodando com banco, in-memory, ou Redis. Isso é o DIP em ação.

## Como Defender na Entrevista

> "O controller é fino — só faz parse do HTTP e delega pro service. O service contém as regras de negócio e depende de uma interface de repositório, não de uma implementação. Isso me permite trocar o storage sem mudar uma linha de lógica, e testar o service com um repositório fake em memória."

## Related

- [[SoftwareEngineering/Testing]]
- [[PHP/PHP]]
- [[PHP/Persistence]]

#### My commentaries
- 
