# Event Loop e ReactPHP

O modelo de processo do PHP e como o ReactPHP muda tudo — processo de longa duração com event loop non-blocking.

Related: [[PHP/PHP]], [[PHP/Concurrency]]

---

## PHP Clássico (FPM / CGI)

```
Request chega
    │
    ▼
PHP-FPM spawna worker
    │
    ▼
Executa o script do começo ao fim
    │
    ▼
Retorna response
    │
    ▼
MORRE — todo estado é descartado
```

- Cada request = processo novo (ou reciclado)
- **Sem estado em memória entre requests** — tudo precisa vir de banco, cache, sessão
- Modelo "shared nothing" — simples, previsível, mas sem como manter dados em memória

## Processo de Longa Duração (Long-running)

Diferente do FPM, o processo **não morre** entre requests:

```
Processo inicia
    │
    ▼
Event loop começa a rodar
    │
    ├── Request 1 → processa → responde
    ├── Request 2 → processa → responde
    ├── Request 3 → processa → responde
    │   (estado em memória persiste entre requests)
    └── ... roda indefinidamente
```

Análogo a: Node.js, Go HTTP server, Python asyncio.

## Event Loop — Como Funciona

```
┌──────────── Event Loop ────────────┐
│                                     │
│  1. Checa: tem eventos pendentes?   │
│     ├── Timer expirou?              │
│     ├── I/O pronto? (socket, file)  │
│     └── Novo request HTTP?          │
│                                     │
│  2. Executa o callback do evento    │
│                                     │
│  3. Volta pro passo 1               │
│                                     │
└─────────────────────────────────────┘
```

- **Single-threaded**: um callback por vez, sem paralelismo
- **Non-blocking**: I/O não trava o loop — registra callback e continua
- **Consequência**: operações CPU-bound bloqueiam TUDO (não faça cálculos pesados no loop)

## ReactPHP

Implementação de event loop e I/O assíncrono para PHP:

```php
use React\Http\HttpServer;
use React\Http\Message\Response;
use React\Socket\SocketServer;

$server = new HttpServer(function (Psr\Http\Message\ServerRequestInterface $request) {
    // Este callback roda pra cada request
    // Estado em memória é acessível aqui
    return new Response(200, ['Content-Type' => 'application/json'],
        json_encode(['status' => 'ok'])
    );
});

$socket = new SocketServer('0.0.0.0:8080');
$server->listen($socket);

echo "Server running on port 8080\n";

// Event loop roda indefinidamente
React\EventLoop\Loop::run();
```

## Por que ReactPHP para Estado em Memória?

> "Escolhi ReactPHP porque é a forma honesta de ter estado em memória em PHP — sem isso eu precisaria de arquivo ou banco, que o enunciado dispensa."

```php
// Estado vive dentro do processo
$accounts = []; // persiste entre requests!

$server = new HttpServer(function ($request) use (&$accounts) {
    $body = json_decode((string)$request->getBody(), true);
    
    // GET /accounts/123 → lê da memória
    // POST /accounts → grava na memória
    // Tudo sem banco, sem arquivo
});
```

## Cuidados com Long-running PHP

- **Memory leaks**: sem garbage collection entre requests — cuidado com arrays que só crescem
- **Error handling**: uma exception não tratada mata o processo inteiro
- **Graceful shutdown**: tratar SIGTERM pra fechar conexões antes de morrer
- **Restart strategy**: usar supervisor (systemd, Docker restart policy) pra reiniciar se crashar

```php
// Monitorar memória
$loop->addPeriodicTimer(60, function () {
    $mb = memory_get_usage(true) / 1024 / 1024;
    echo "Memory: {$mb}MB\n";
    if ($mb > 128) {
        echo "Memory limit approaching, shutting down\n";
        exit(0); // supervisor reinicia
    }
});
```

## Comparação

| | PHP-FPM | ReactPHP | Node.js | Go |
|--|---------|----------|---------|-----|
| Modelo | Process-per-request | Event loop | Event loop | Goroutines |
| Estado | Descartado | Persiste | Persiste | Persiste |
| Concorrência | Multi-process | Single-thread async | Single-thread async | Multi-thread |
| I/O | Blocking | Non-blocking | Non-blocking | Non-blocking |

## Related

- [[PHP/Concurrency]]
- [[PHP/PHP]]
- [[Linux/ProcessesAndThreads]]

## Resources

- https://reactphp.org
- https://sergeyzhuk.me/reactphp-series (série completa)

#### My commentaries
- 
