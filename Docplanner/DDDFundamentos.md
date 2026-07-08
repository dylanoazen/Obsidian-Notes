---
tags: [docplanner-prep, ddd, arquitetura]
status: draft
---

# DDD — Domain-Driven Design

## Por que isso importa
Docplanner usa DDD. No contexto deles (healthcare — doctors, patients, appointments, calendars), o domínio é complexo. DDD é a forma de organizar código em torno da linguagem e regras do negócio, não em torno do framework.

## Bounded Context

Cada parte do sistema tem seu próprio modelo e linguagem. "User" no contexto de Booking é diferente de "User" no contexto de Billing.

```
┌─── Booking Context ───┐   ┌─── Billing Context ───┐
│ Doctor                 │   │ Customer              │
│ Patient                │   │ Invoice               │
│ Appointment            │   │ Payment               │
│ Slot                   │   │ Subscription          │
└────────────────────────┘   └───────────────────────┘
         │                              │
         └──── integram via eventos ────┘
```

Cada bounded context tem suas próprias entities, repositórios e regras. Comunicam entre si via domain events ou APIs, nunca acessando o banco do outro.

## Ubiquitous Language

O código usa os **mesmos termos** que o negócio:

```php
// ❌ Linguagem genérica
$item->setStatus(3);

// ✅ Ubiquitous language
$appointment->confirm();
$slot->markAsBooked();
$doctor->goOnVacation(DateRange $period);
```

Se o product manager diz "confirmar consulta", o código tem `$appointment->confirm()`. Não `$appointment->setConfirmed(true)`.

## Entity vs Value Object

### Entity — tem identidade

```php
class Doctor {
    private DoctorId $id;       // identidade única
    private string $name;
    private Specialty $specialty;
    // Dois Doctors com mesmo nome são DIFERENTES (IDs diferentes)
}
```

### Value Object — definido pelo valor

```php
class Money {
    public function __construct(
        public readonly int $amount,
        public readonly string $currency,
    ) {}
    // Dois Money(100, 'BRL') são IGUAIS
    // Imutável: criar novo em vez de mudar
}

class DateRange {
    public function __construct(
        public readonly DateTimeImmutable $start,
        public readonly DateTimeImmutable $end,
    ) {}
}

class Address {
    public function __construct(
        public readonly string $street,
        public readonly string $city,
        public readonly string $country,
    ) {}
}
```

Value Objects são imutáveis e comparados por valor, não por identidade.

## Aggregate

Cluster de entities e value objects que são tratados como uma unidade. Tem um **Aggregate Root** que é o único ponto de acesso:

```php
// Appointment é o Aggregate Root
class Appointment {
    private AppointmentId $id;
    private Doctor $doctor;
    private Patient $patient;
    private Slot $slot;
    private Status $status;

    public function confirm(): void {
        if (!$this->status->isPending()) {
            throw new CannotConfirmException();
        }
        $this->status = Status::Confirmed;
        $this->recordEvent(new AppointmentConfirmed($this->id));
    }

    public function cancel(string $reason): void {
        $this->status = Status::Cancelled;
        $this->slot->release();
        $this->recordEvent(new AppointmentCancelled($this->id, $reason));
    }
}

// Regra: só acessa Slot ATRAVÉS do Appointment, nunca direto
```

## Domain Events

Algo que aconteceu no domínio e que outros contextos podem reagir:

```php
class AppointmentConfirmed {
    public function __construct(
        public readonly AppointmentId $id,
        public readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}
}
// Quem escuta: billing cria invoice, notification envia email
```

## Domain Service

Quando a lógica não pertence a nenhuma entity:

```php
class SlotAvailabilityService {
    public function findAvailableSlots(Doctor $doctor, DateRange $range): array {
        // lógica que envolve múltiplos aggregates
    }
}
```

## Comparação com minha stack
No ERP da Acesse, as regras de negócio provavelmente vivem em controllers ou services acoplados ao Eloquent. Com DDD, elas vivem em entities e domain services puros — o framework não aparece. No tray-go, a separação em `internal/domain/` já segue essa ideia.

## Perguntas que podem cair
1. "O que é um Bounded Context?" → Fronteira onde um modelo e sua linguagem são consistentes. Contexts diferentes podem ter modelos diferentes pra mesma coisa.
2. "Qual a diferença entre Entity e Value Object?" → Entity tem identidade (ID). Value Object é definido pelo valor e é imutável.
3. "O que é um Aggregate?" → Cluster de objetos tratados como unidade com um root. Acesso e persistência sempre pelo root.
4. "Por que Domain Events?" → Desacoplamento entre contexts. Booking confirma consulta, Billing reage criando invoice — sem chamar diretamente.
5. "Dê um exemplo de Ubiquitous Language." → O código usa os mesmos termos do negócio: `confirm()`, `cancel()`, `reschedule()`.

## Links
- [[Docplanner/PortsAndAdaptersPHP]]
- [[Docplanner/DomainEvents]]
- [[Architecture/HexagonalArchitecture]]
- [[PHP/SOLID]]
