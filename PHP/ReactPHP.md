# ReactPHP — HttpServer Pattern

Como montar um servidor HTTP com ReactPHP do zero.

Related: [[PHP/EventLoop]], [[PHP/Concurrency]]

---

## O que é ReactPHP

ReactPHP é uma biblioteca de **I/O assíncrono e não-bloqueante** para PHP, construída em torno de um **event loop**. Ela permite escrever servidores de longa duração — processos que ficam vivos indefinidamente, processando múltiplas conexões sem criar um processo novo por request.

### Instalação

```bash
composer require react/http
```

O pacote `react/http` já puxa `react/socket` e `react/event-loop` como dependências.

### Filosofia

O PHP clássico (FPM/Apache) segue o modelo **shared-nothing**: cada request inicia e termina um processo isolado, sem estado compartilhado. Simples e previsível, mas incapaz de manter estado em memória entre requests.

ReactPHP inverte isso:

```
PHP-FPM (clássico)          ReactPHP (long-running)
──────────────────          ───────────────────────
request → nasce             processo sobe uma vez
processa                    event loop roda para sempre
responde                    request 1 → callback → responde
MORRE                       request 2 → callback → responde
                            request N → callback → responde
```

Isso possibilita:
- **Estado em memória** entre requests — arrays, objetos, caches locais
- **Concorrência sem threads** — um processo, I/O não-bloqueante via event loop
- **Alta performance** — sem overhead de boot do PHP por request

### Quando usar

| Cenário | ReactPHP faz sentido? |
|---|---|
| Estado em memória sem banco | ✅ sim — principal caso de uso |
| Alta concorrência I/O-bound | ✅ sim — event loop não bloqueia |
| WebSockets / conexões longas | ✅ sim — nativo |
| CRUD simples com banco | ⚠️ Laravel/Symfony é mais produtivo |
| CPU-bound pesado (ML, encoding) | ❌ não — bloqueia o event loop |

### Componentes da lib

```
react/event-loop   → o motor central (Loop::run())
react/socket       → camada TCP — abre e gerencia conexões
react/http         → camada HTTP — parseia requests e serializa responses
react/promise      → promises para operações assíncronas
react/stream       → streams de dados (leitura/escrita assíncrona)
```

Para um servidor HTTP simples, só `react/http` é necessário — os outros são puxados como dependência.

---

## Os 3 Componentes

```
EventLoop     → o motor. Fica rodando indefinidamente.
HttpServer    → parseia HTTP e chama o callback para cada request.
SocketServer  → abre a porta TCP no sistema operacional.
```

```php
use React\Http\HttpServer;
use React\Http\Message\Response;
use React\Socket\SocketServer;
use React\EventLoop\Loop;
use Psr\Http\Message\ServerRequestInterface;

$server = new HttpServer(function (ServerRequestInterface $request) {
    return new Response(200, ['Content-Type' => 'application/json'], 'ok');
});

$socket = new SocketServer('0.0.0.0:8080');
$server->listen($socket);

echo "Server running on port 8080\n";

Loop::run(); // bloqueia aqui — fica vivo para sempre
```

---

## Request — Métodos Úteis

```php
$request->getMethod()                        // "GET", "POST"
$request->getUri()->getPath()                // "/event", "/balance/100"
$request->getQueryParams()                   // array: ?account_id=100 → ['account_id' => '100']
(string) $request->getBody()                 // body como string (JSON bruto)
json_decode((string) $request->getBody(), true) // body como array associativo
```

---

## Response

```php
new Response(
    $statusCode,  // 200, 201, 404, etc
    $headers,     // ['Content-Type' => 'application/json']
    $body         // string — sempre json_encode() para APIs JSON
);
```

Respostas comuns:

```php
// 200 OK
new Response(200, ['Content-Type' => 'application/json'], json_encode($data));

// 201 Created
new Response(201, ['Content-Type' => 'application/json'], json_encode($data));

// 404 Not Found
new Response(404, [], '0');

// 400 Bad Request
new Response(400, [], json_encode(['error' => 'invalid payload']));
```

---

## Roteamento Manual

ReactPHP não tem router embutido. O padrão é switch no método + path:

```php
$server = new HttpServer(function (ServerRequestInterface $request) {
    $method = $request->getMethod();
    $path   = $request->getUri()->getPath();

    // POST /event
    if ($method === 'POST' && $path === '/event') {
        $body = json_decode((string) $request->getBody(), true);
        // processa...
        return new Response(201, ['Content-Type' => 'application/json'], json_encode($result));
    }

    // GET /balance?account_id=100
    if ($method === 'GET' && $path === '/balance') {
        $params = $request->getQueryParams();
        $id = $params['account_id'] ?? null;
        // busca...
        return new Response(200, ['Content-Type' => 'application/json'], json_encode($result));
    }

    return new Response(404, [], '0');
});
```

---

## Estado em Memória

O processo não morre entre requests — variáveis declaradas fora do callback persistem.
Usa `use (&$var)` para injetar no closure e permitir escrita:

```php
$store = []; // persiste entre todos os requests

$server = new HttpServer(function ($request) use (&$store) {
    // lê e escreve $store livremente
    $store['key'] = 'value';
    return new Response(200, [], json_encode($store));
});
```

Com um objeto (Repository), injeta via `use` sem `&` — objetos são referência por padrão:

```php
$repository = new AccountRepository();

$server = new HttpServer(function ($request) use ($repository) {
    // $repository é o mesmo objeto em todos os requests
    $account = $repository->find('100');
    return new Response(200, [], json_encode($account->toArray()));
});
```

---

## Fluxo Completo de Uma Requisição

```
[Client]
   │
   │  POST /event  HTTP/1.1
   ▼
[SocketServer]  ←  escuta 0.0.0.0:8080
   │
   │  entrega conexão TCP
   ▼
[HttpServer]  ←  parseia o HTTP
   │
   │  cria $request e chama o callback
   ▼
[Seu callback]  ←  lê body, chama service, monta response
   │
   │  retorna Response
   ▼
[Client recebe]
```

---

## 0.0.0.0 vs 127.0.0.1

```
0.0.0.0    → escuta em TODAS as interfaces (acessível de fora — Docker, produção)
127.0.0.1  → escuta só no loopback (só o próprio host — desenvolvimento local)
```

Em Docker, usar `0.0.0.0` — caso contrário o container não é acessível de fora.

---

## Estrutura Recomendada

```
server.php           ← entry point — monta o HttpServer e injeta dependências
src/
  Http/
    EventController.php   ← recebe $request, chama service, retorna Response
  Application/
    EventService.php      ← regras de negócio (deposit, withdraw, transfer)
  Domain/
    Account.php           ← entidade
    AccountRepository.php ← estado em memória (o "banco")
```

O `server.php` conhece tudo e injeta via `use`. O controller não sabe do ReactPHP — só recebe dados, devolve array.

---

## Related

- [[PHP/EventLoop]]
- [[PHP/Concurrency]]
- [[SoftwareEngineering/Testing]]

## Resources

- https://reactphp.org/http
- https://reactphp.org/socket
