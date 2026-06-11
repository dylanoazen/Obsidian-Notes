# EBANX — Take-home: In-Memory Banking API

API bancária in-memory em PHP 8.1 + ReactPHP. Processo de longa duração, sem banco de dados.

Projeto: `~/Projects/testeEbanx`

Related: [[PHP/ReactPHP]], [[PHP/EventLoop]], [[PHP/Testing]], [[PHP/EBANX-Entrevista]]

---

## O Que o Projeto Faz

Três operações bancárias via HTTP:

| Evento     | O que faz                                  |
|------------|--------------------------------------------|
| `deposit`  | Cria conta se não existir + adiciona saldo |
| `withdraw` | Subtrai saldo (erro se insuficiente)       |
| `transfer` | Subtrai de `origin`, adiciona em `destination` |

Rotas:

```
POST /reset               → zera o estado (retorna 200 OK)
GET  /balance?account_id= → retorna saldo (404 se não existir)
POST /event               → processa deposit / withdraw / transfer
```

---

## Estrutura do Projeto

```
testeEbanx/
├── public/
│   └── server.php              ← entry point ReactPHP  (TODO)
├── src/
│   ├── Domain/
│   │   ├── Account.php         ✅ implementado
│   │   ├── AccountRepository.php ✅ implementado
│   │   └── Exception/
│   │       └── InsufficientFundsException.php ✅ implementado
│   ├── Application/
│   │   └── EventService.php    ← regras de negócio  (TODO)
│   └── Http/
│       └── EventController.php ← recebe request, chama service  (TODO)
├── tests/
│   └── AccountTest.php         ✅ 4 testes passando
├── composer.json
└── phpunit.xml
```

---

## Domain — O que já foi implementado

### Account.php

Entidade. Encapsula id + balance, não expõe o balance diretamente.

```php
final class Account
{
    private readonly string $id;
    private int $balance;

    public function __construct(string $id, int $balance = 0) { ... }

    public function id(): string { ... }
    public function balance(): int { ... }
    public function toArray(): array { ... }   // ['id' => ..., 'balance' => ...]

    public function deposit(int $amount): void { ... }
    public function withdraw(int $amount): void { ... }  // lança InsufficientFundsException

    private function guardPositive(int $amount): void { ... } // lança InvalidArgumentException
}
```

### AccountRepository.php

Estado em memória. Um array `string → Account` que vive enquanto o processo viver.

```php
final class AccountRepository
{
    private array $accounts = [];   // ['100' => Account, '200' => Account, ...]

    public function find(string $id): ?Account { ... }   // null se não existir
    public function save(Account $account): void { ... }
    public function reset(): void { ... }                 // limpa tudo (POST /reset)
}
```

### InsufficientFundsException.php

Carrega contexto: qual conta, qual saldo, quanto foi pedido.

```php
final class InsufficientFundsException extends \RuntimeException
{
    public function __construct(
        public readonly string $accountId,
        public readonly int $balance,
        public readonly int $requested,
    ) { ... }
}
```

---

## O que Falta Implementar

### 1. EventService.php — `src/Application/EventService.php`

Recebe o array do body JSON e executa a operação correta.

```php
final class EventService
{
    public function __construct(private AccountRepository $repository) {}

    public function handle(array $event): array
    {
        return match($event['type']) {
            'deposit'  => $this->deposit($event),
            'withdraw' => $this->withdraw($event),
            'transfer' => $this->transfer($event),
            default    => throw new \InvalidArgumentException('Unknown event type'),
        };
    }

    private function deposit(array $event): array
    {
        $account = $this->repository->find($event['destination'])
            ?? new Account($event['destination']);

        $account->deposit($event['amount']);
        $this->repository->save($account);

        return ['destination' => $account->toArray()];
    }

    private function withdraw(array $event): array
    {
        $account = $this->repository->find($event['origin']);
        // se não existir → 404

        $account->withdraw($event['amount']);
        $this->repository->save($account);

        return ['origin' => $account->toArray()];
    }

    private function transfer(array $event): array
    {
        $origin      = $this->repository->find($event['origin']);
        $destination = $this->repository->find($event['destination'])
            ?? new Account($event['destination']);

        $origin->withdraw($event['amount']);
        $destination->deposit($event['amount']);

        $this->repository->save($origin);
        $this->repository->save($destination);

        return [
            'origin'      => $origin->toArray(),
            'destination' => $destination->toArray(),
        ];
    }
}
```

### 2. EventController.php — `src/Http/EventController.php`

Traduz `$request` HTTP → array → chama service → monta `Response`.

```php
use React\Http\Message\Response;
use Psr\Http\Message\ServerRequestInterface;

final class EventController
{
    public function __construct(private EventService $service) {}

    public function handle(ServerRequestInterface $request): Response
    {
        $body = json_decode((string) $request->getBody(), true);

        if (!isset($body['type'])) {
            return new Response(400, [], '0');
        }

        try {
            $result = $this->service->handle($body);
            return new Response(201, ['Content-Type' => 'application/json'], json_encode($result));
        } catch (InsufficientFundsException) {
            return new Response(404, [], '0');
        }
    }
}
```

### 3. server.php — `public/server.php`

Entry point. Instancia tudo e sobe o ReactPHP.

```php
<?php
require __DIR__ . '/../vendor/autoload.php';

use React\Http\HttpServer;
use React\Http\Message\Response;
use React\Socket\SocketServer;

$repository = new App\Domain\AccountRepository();
$service    = new App\Application\EventService($repository);
$controller = new App\Http\EventController($service);

$server = new HttpServer(function ($request) use ($repository, $controller) {
    $method = $request->getMethod();
    $path   = $request->getUri()->getPath();

    if ($method === 'POST' && $path === '/reset') {
        $repository->reset();
        return new Response(200, [], 'OK');
    }

    if ($method === 'GET' && $path === '/balance') {
        $id      = $request->getQueryParams()['account_id'] ?? null;
        $account = $repository->find($id);
        if (!$account) return new Response(404, [], '0');
        return new Response(200, ['Content-Type' => 'application/json'], json_encode($account->balance()));
    }

    if ($method === 'POST' && $path === '/event') {
        return $controller->handle($request);
    }

    return new Response(404, [], '0');
});

$socket = new SocketServer('0.0.0.0:8080');
$server->listen($socket);
echo "Running on http://0.0.0.0:8080\n";
React\EventLoop\Loop::run();
```

---

## Rodar o Projeto

```bash
# instalar dependências
composer install

# subir o servidor
composer start
# ou diretamente:
php public/server.php

# rodar testes
composer test
# ou:
vendor/bin/phpunit
```

---

## Testar Manualmente (curl)

```bash
# reset
curl -X POST http://localhost:8080/reset

# deposit (cria conta 100 com 10)
curl -X POST http://localhost:8080/event \
  -H 'Content-Type: application/json' \
  -d '{"type":"deposit","destination":"100","amount":10}'
# → {"destination":{"id":"100","balance":10}}  201

# balance
curl http://localhost:8080/balance?account_id=100
# → 10  200

# conta inexistente
curl http://localhost:8080/balance?account_id=999
# → 0  404

# withdraw
curl -X POST http://localhost:8080/event \
  -H 'Content-Type: application/json' \
  -d '{"type":"withdraw","origin":"100","amount":5}'
# → {"origin":{"id":"100","balance":5}}  201

# withdraw insuficiente
curl -X POST http://localhost:8080/event \
  -H 'Content-Type: application/json' \
  -d '{"type":"withdraw","origin":"100","amount":999}'
# → 0  404

# transfer
curl -X POST http://localhost:8080/event \
  -H 'Content-Type: application/json' \
  -d '{"type":"transfer","origin":"100","amount":5,"destination":"300"}'
# → {"origin":{"id":"100","balance":0},"destination":{"id":"300","balance":5}}  201
```

---

## Conceitos Aplicados

- **Domain-Driven Design** — Account encapsula regra, Repository guarda estado
- **ReactPHP** — processo longa duração, estado em memória entre requests
- **PHP 8.1** — `readonly`, constructor promotion, `match`, `declare(strict_types=1)`
- **PHPUnit** — testes unitários do domain sem subir o servidor
- **PSR-4** — autoload via Composer (`App\` → `src/`)

---

#### My commentaries
- 
