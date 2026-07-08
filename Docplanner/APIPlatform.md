---
tags: [docplanner-prep, symfony, api]
status: draft
---

# API Platform

## Por que isso importa
Docplanner expõe APIs REST. API Platform gera CRUD automaticamente a partir das entities Doctrine, com documentação OpenAPI, pagination, filters e serialization — sem escrever controllers. Pra uma empresa de saúde com muitos recursos (doctors, patients, appointments, slots), isso escala muito.

## O que é

Framework PHP que gera API REST (e GraphQL) automaticamente em cima do Symfony + Doctrine.

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
#[ORM\Entity]
class Doctor {
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(length: 100)]
    private string $specialty;
}
```

Só com `#[ApiResource]`, você ganha:
- `GET /api/doctors` — lista paginada
- `GET /api/doctors/{id}` — detalhe
- `POST /api/doctors` — criar
- `PUT /api/doctors/{id}` — atualizar
- `DELETE /api/doctors/{id}` — remover
- Documentação Swagger/OpenAPI automática

## Serialization Groups

Controla o que aparece na response:

```php
#[ApiResource(
    normalizationContext: ['groups' => ['doctor:read']],
    denormalizationContext: ['groups' => ['doctor:write']],
)]
class Doctor {
    #[Groups(['doctor:read'])]
    private ?int $id = null;

    #[Groups(['doctor:read', 'doctor:write'])]
    private string $name;

    #[Groups(['doctor:read'])]
    private string $internalCode;  // só leitura, não aceita no POST
}
```

## Filters

```php
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['name' => 'partial', 'specialty' => 'exact'])]
#[ApiFilter(OrderFilter::class, properties: ['name'])]
class Doctor { }

// GET /api/doctors?name=silva&specialty=cardiology&order[name]=asc
```

## Custom Operations

Quando o CRUD automático não basta:

```php
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection(),
        new Post(
            uriTemplate: '/doctors/{id}/approve',
            controller: ApproveDoctorController::class,
        ),
    ]
)]
```

## Data Providers e Processors

Pra lógica que não vem do Doctrine (APIs externas, search engines):

```php
class DoctorProvider implements ProviderInterface {
    public function provide(Operation $operation, array $uriVariables = []): object|array|null {
        // busca de onde quiser — ElasticSearch, API, cache
    }
}
```

## Comparação com minha stack
Laravel não tem equivalente direto. O mais perto é Laravel API Resource + Spatie Query Builder, mas você ainda escreve controllers. API Platform é zero-controller pra CRUD e te deixa customizar só onde precisa.

## Perguntas que podem cair
1. "O que é API Platform?" → Framework que gera REST API automaticamente a partir de entities com metadata.
2. "Como controlar quais campos aparecem na response?" → Serialization groups via `normalizationContext`.
3. "E se preciso de lógica custom num endpoint?" → Custom operations com controller próprio, ou Data Providers/Processors.

## Links
- [[Docplanner/DoctrineORM]]
- [[Docplanner/SymfonyServiceContainer]]
- [[PHP/REST]]
