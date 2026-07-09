---
tags: [symfony, laravel]
status: draft
---

# Symfony vs Laravel — Comparativo Direto

## Por que isso importa
A Docplanner usa Symfony. Você vem do Laravel. O entrevistador vai querer saber se você entende as diferenças de *filosofia*, não só de syntax. Laravel prioriza velocidade de desenvolvimento (convention over configuration). Symfony prioriza explicitness e controle (configuration over convention).

## Tabela Lado a Lado

| Conceito | Laravel | Symfony |
|----------|---------|---------|
| **Container** | `app()` helper, facades | Autowiring + YAML/PHP config |
| **ORM** | Eloquent (Active Record) | Doctrine (Data Mapper) |
| **Routing** | `Route::get()` ou attributes | YAML, XML, ou `#[Route]` attributes |
| **Middleware** | Middleware classes | EventSubscriber / kernel.event |
| **Events** | `Event::dispatch()` | EventDispatcher component |
| **Queue** | Laravel Queue (Redis/SQS) | Messenger component (RabbitMQ/etc) |
| **Auth** | Sanctum / Passport | Security component (voters, firewalls) |
| **CLI** | Artisan commands | Console component |
| **Template** | Blade | Twig |
| **Validation** | Form Request / Validator | Validator component + constraints |
| **Admin** | Nova / Filament | EasyAdmin |
| **API** | API Resource / manual | API Platform (auto-generated) |

## Service Container

### Laravel — Mágico

```php
// Facades escondem o container
Cache::get('key');           // static call que resolve do container
app(UserService::class);     // resolve manual
```

Laravel registra bindings em `AppServiceProvider` e usa auto-discovery. Facades são syntax sugar — por baixo chamam o container.

### Symfony — Explícito

```php
// Autowiring: Symfony lê os type hints e injeta automaticamente
class UserController {
    public function __construct(
        private UserService $userService  // ← Symfony resolve sozinho
    ) {}
}
```

Mas se precisar configurar algo específico:

```yaml
# config/services.yaml
services:
    App\Service\PaymentGateway:
        arguments:
            $apiKey: '%env(PAYMENT_API_KEY)%'
```

Diferença fundamental: **Laravel esconde a injeção atrás de facades. Symfony força você a declarar dependências no construtor.** No Symfony, se a classe precisa de algo, está no construtor — sempre.

## ORM — Active Record vs Data Mapper

### Eloquent (Active Record)

```php
// O model É a tabela. Ele sabe se salvar.
$user = User::find(1);
$user->name = 'Dylan';
$user->save();  // o próprio objeto persiste
```

O model conhece o banco. Simples, mas mistura domínio com persistência.

### Doctrine (Data Mapper)

```php
// A entity NÃO sabe do banco. É um objeto puro.
$user = $em->find(User::class, 1);
$user->setName('Dylan');
$em->flush();  // o EntityManager persiste, não o objeto
```

A entity é um POPO (Plain Old PHP Object). Quem sabe do banco é o EntityManager + Repository. **Isso é exatamente o que hexagonal pede** — por isso Docplanner usa Doctrine.

## Events

### Laravel
```php
event(new OrderShipped($order));

// Listener registrado em EventServiceProvider
protected $listen = [
    OrderShipped::class => [SendShipmentNotification::class],
];
```

### Symfony
```php
$dispatcher->dispatch(new OrderShipped($order));

// Listener via attribute
#[AsEventListener(event: OrderShipped::class)]
class SendShipmentNotification {
    public function __invoke(OrderShipped $event): void { }
}
```

Mesma ideia, syntax diferente. Symfony usa o EventDispatcher component que Laravel por baixo também usa (Illuminate Events é inspirado nele).

## Comparação com minha stack
Laravel = produtividade rápida, muita magia, ótimo pra ERPs e CRUDs. Symfony = controle total, ótimo pra DDD e arquitetura hexagonal. A migração mental é: **parar de depender de facades e começar a pensar em injeção explícita**.

## Perguntas que podem cair
1. "Qual a diferença principal entre Eloquent e Doctrine?" → Active Record vs Data Mapper. Eloquent o model se persiste, Doctrine separa entity de persistência.
2. "Por que Symfony é melhor pra DDD?" → Doctrine não polui o domínio com lógica de banco. Entities são objetos puros.
3. "O que são facades no Laravel e por que Symfony não tem?" → Facades são proxies estáticos pro container. Symfony prefere DI explícita no construtor.
4. "Como funciona o autowiring do Symfony?" → Lê type hints dos construtores e resolve automaticamente do container.

## Links
- [[PHP/SymfonyServiceContainer]]
- [[PHP/DoctrineORM]]
- [[PHP/SOLID]]
- [[Architecture/HexagonalArchitecture]]
