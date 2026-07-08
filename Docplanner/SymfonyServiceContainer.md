---
tags: [docplanner-prep, symfony]
status: draft
---

# Symfony Service Container

## Por que isso importa
O service container é o coração do Symfony. Tudo é um service — controllers, repositórios, commands. Entender como ele resolve dependências é fundamental porque na Docplanner (DDD + hexagonal), a composição de objetos é o que cola as camadas.

## Conceitos-chave

### Autowiring

Symfony lê os type hints do construtor e injeta automaticamente:

```php
class OrderService {
    public function __construct(
        private OrderRepositoryInterface $repo,  // Symfony resolve sozinho
        private LoggerInterface $logger,          // também
    ) {}
}
```

**Não precisa registrar nada** se a classe está em `src/` e o `services.yaml` tem:

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
    App\:
        resource: '../src/'
```

### Quando autowiring não basta

Se tem duas implementações da mesma interface:

```yaml
services:
    App\Domain\OrderRepositoryInterface:
        alias: App\Infrastructure\DoctrineOrderRepository

    # Ou bind por nome de variável
    App\Service\OrderService:
        arguments:
            $repo: '@App\Infrastructure\DoctrineOrderRepository'
```

### Service Tags

Tags marcam services pra serem coletados automaticamente:

```php
#[AutoconfigureTag('app.event_handler')]
class OrderCreatedHandler { }

// Compiler pass ou DI config coleta todos com essa tag
```

Symfony usa isso internamente: todo `#[AsEventListener]` é tagged, todo `#[AsCommand]` é tagged.

### Compiler Passes

Hooks que rodam durante a compilação do container — manipulam definições de services antes do container ser finalizado:

```php
class RegisterHandlersPass implements CompilerPassInterface {
    public function process(ContainerBuilder $container): void {
        $tagged = $container->findTaggedServiceIds('app.event_handler');
        // registra todos os handlers no dispatcher
    }
}
```

Equivalente conceitual: o que o Laravel faz automaticamente com auto-discovery, Symfony te dá controle total via compiler passes.

### Service Scope e Lifecycle

```yaml
services:
    App\Service\HeavyService:
        shared: true    # singleton (default) — mesma instância sempre
        # shared: false → nova instância a cada inject
```

Por default, tudo é singleton dentro de um request. Diferente do Laravel onde você pode escolher singleton/bind/scoped.

## Comparação com minha stack

| Laravel | Symfony |
|---------|---------|
| `app()->bind()` | `services.yaml` |
| `app()->singleton()` | `shared: true` (default) |
| Facades | Não existe — usa DI |
| Auto-discovery | Autowiring + autoconfigure |
| Service Provider | Compiler Pass |

## Perguntas que podem cair
1. "O que é autowiring?" → Symfony resolve dependências lendo type hints do construtor automaticamente.
2. "Como resolver ambiguidade quando tem 2 implementações da mesma interface?" → Alias no services.yaml ou bind por nome de variável.
3. "O que é um compiler pass?" → Hook que manipula definições de services durante a compilação do container.
4. "Services são singleton por default?" → Sim, `shared: true` é o default.

## Links
- [[Docplanner/SymfonyVsLaravel]]
- [[Docplanner/DoctrineORM]]
- [[PHP/SOLID]]
