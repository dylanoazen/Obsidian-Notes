---
tags: [docplanner-prep, ddd, symfony]
status: draft
---

# Domain Events na Prática

## Por que isso importa
Na Docplanner, quando uma consulta é confirmada, vários side effects acontecem: enviar email, criar invoice, atualizar calendário. Domain Events desacoplam o "o que aconteceu" de "quem reage".

## O que é um Domain Event

Fato que aconteceu no domínio. Imutável, no passado:

```php
class AppointmentConfirmed {
    public function __construct(
        public readonly string $appointmentId,
        public readonly string $doctorId,
        public readonly string $patientEmail,
        public readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}
}
```

Convenção: nome no **passado** (`Confirmed`, `Cancelled`, `Created`).

## Recording Events no Aggregate

```php
class Appointment {
    private array $domainEvents = [];

    public function confirm(): void {
        $this->status = Status::Confirmed;
        $this->recordEvent(new AppointmentConfirmed(
            $this->id->value(),
            $this->doctor->id()->value(),
            $this->patient->email(),
        ));
    }

    private function recordEvent(object $event): void {
        $this->domainEvents[] = $event;
    }

    public function pullEvents(): array {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

## Dispatching

Dois caminhos no Symfony:

### 1. EventDispatcher (síncrono, mesmo processo)

```php
// Após persistir o aggregate
$appointment->confirm();
$repository->save($appointment);

foreach ($appointment->pullEvents() as $event) {
    $dispatcher->dispatch($event);
}
```

```php
#[AsEventListener(event: AppointmentConfirmed::class)]
class SendConfirmationEmail {
    public function __invoke(AppointmentConfirmed $event): void {
        // envia email
    }
}
```

### 2. Messenger (assíncrono, via RabbitMQ)

```php
foreach ($appointment->pullEvents() as $event) {
    $bus->dispatch($event);  // vai pro RabbitMQ
}
```

```yaml
framework:
    messenger:
        routing:
            App\Domain\Event\AppointmentConfirmed: async
```

### Quando usar qual?

| | EventDispatcher | Messenger |
|--|----------------|-----------|
| Execução | Síncrona | Assíncrona |
| Falha | Falha o request | Retry automático |
| Uso | Coisas rápidas (log, cache) | Coisas lentas (email, sync) |
| Transação | Mesma transação | Fora da transação |

## Doctrine Events vs Domain Events

**Não confundir:**
- **Doctrine Events** (`postPersist`, `prePersist`): lifecycle do ORM, técnicos
- **Domain Events** (`AppointmentConfirmed`): fatos do negócio, semânticos

Domain Events são do domínio. Doctrine Events são da infraestrutura.

## Perguntas que podem cair
1. "Qual a diferença entre Domain Event e Event do Symfony?" → Domain Event é conceito de DDD (fato do negócio). Event do Symfony é mecanismo de dispatch (ferramenta).
2. "Síncrono ou assíncrono pra domain events?" → Depende: notificações rápidas sync, side effects pesados async via Messenger.
3. "O aggregate deve dispatchar o evento?" → Não — ele grava. Quem dispatcha é o application service após persistir.

## Links
- [[Docplanner/DDDFundamentos]]
- [[Docplanner/SymfonyMessenger]]
- [[Docplanner/RabbitMQ]]
