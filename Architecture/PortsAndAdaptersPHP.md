---
tags: [ddd, arquitetura, symfony]
status: draft
---

# Ports & Adapters em PHP (Symfony)

## Por que isso importa
Ports & Adapters (hexagonal) em PHP/Symfony. Mesma filosofia do projeto TrayGo em Go — domínio isolado, adapters nas bordas.

## Estrutura de Pastas (Symfony + DDD)

```
src/
├── Domain/                          # CORE — puro, sem framework
│   ├── Model/
│   │   ├── Appointment.php          # Entity (Aggregate Root)
│   │   ├── AppointmentId.php        # Value Object
│   │   ├── Doctor.php               # Entity
│   │   └── Slot.php                 # Entity
│   ├── Event/
│   │   ├── AppointmentConfirmed.php # Domain Event
│   │   └── AppointmentCancelled.php
│   ├── Port/
│   │   ├── AppointmentRepository.php    # Interface (port de saída)
│   │   └── NotificationService.php      # Interface (port de saída)
│   ├── Service/
│   │   └── BookingService.php       # Domain Service
│   └── Exception/
│       └── SlotNotAvailableException.php
│
├── Application/                     # Use Cases / Command Handlers
│   ├── Command/
│   │   ├── BookAppointment.php      # Command (DTO)
│   │   └── BookAppointmentHandler.php
│   └── Query/
│       ├── GetDoctorSlots.php
│       └── GetDoctorSlotsHandler.php
│
└── Infrastructure/                  # ADAPTERS — implementações concretas
    ├── Persistence/
    │   └── DoctrineAppointmentRepository.php  # Adapter de saída
    ├── Messaging/
    │   └── RabbitMQNotificationService.php    # Adapter de saída
    └── Http/
        └── Controller/
            └── AppointmentController.php      # Adapter de entrada
```

## Comparação Go vs PHP

```
Go (tray-go)                    PHP (Symfony)
cmd/tray-sync/main.go     →    src/Infrastructure/Http/Controller/
internal/domain/           →    src/Domain/
internal/adapters/         →    src/Infrastructure/
internal/ (compilador)     →    Disciplina + Domain camada sem imports de infra
```

A diferença: Go tem `internal/` enforced pelo compilador. Em PHP, a regra "Domain não importa Infrastructure" é por disciplina (e pode ser enforced via PHPStan rules ou Deptrac).

## Port (Interface no Domain)

```php
// src/Domain/Port/AppointmentRepository.php
namespace App\Domain\Port;

interface AppointmentRepository {
    public function findById(AppointmentId $id): ?Appointment;
    public function save(Appointment $appointment): void;
    public function nextIdentity(): AppointmentId;
}
```

**O domínio define O QUE precisa. A infraestrutura decide COMO.**

## Adapter (Implementação na Infrastructure)

```php
// src/Infrastructure/Persistence/DoctrineAppointmentRepository.php
namespace App\Infrastructure\Persistence;

class DoctrineAppointmentRepository implements AppointmentRepository {
    public function __construct(private EntityManagerInterface $em) {}

    public function findById(AppointmentId $id): ?Appointment {
        return $this->em->find(Appointment::class, $id->value());
    }

    public function save(Appointment $appointment): void {
        $this->em->persist($appointment);
        $this->em->flush();
    }
}
```

## Binding no Container

```yaml
# config/services.yaml
services:
    App\Domain\Port\AppointmentRepository:
        alias: App\Infrastructure\Persistence\DoctrineAppointmentRepository
```

Agora qualquer classe que pedir `AppointmentRepository` no construtor recebe o Doctrine automaticamente.

## Regra de Dependência Enforced

```bash
# deptrac — enforce dependency rules
composer require --dev qossmic/deptrac-shim
```

```yaml
# deptrac.yaml
layers:
    - name: Domain
      collectors: [{ type: directory, value: src/Domain }]
    - name: Application
      collectors: [{ type: directory, value: src/Application }]
    - name: Infrastructure
      collectors: [{ type: directory, value: src/Infrastructure }]

ruleset:
    Domain: []                        # Domain não importa NADA
    Application: [Domain]             # Application só importa Domain
    Infrastructure: [Domain, Application]  # Infra importa tudo
```

## Perguntas que podem cair
1. "Como você aplica hexagonal em Symfony?" → Domain puro sem imports de framework. Ports são interfaces no domain. Adapters são implementações na infrastructure. Container faz o binding.
2. "Como garante que o domain não depende de infra?" → Deptrac ou PHPStan rules que falham no CI se domain importar infra.
3. "Qual a diferença entre Port e Adapter?" → Port é a interface (contrato). Adapter é a implementação concreta.

## Links
- [[Architecture/HexagonalArchitecture]]
- [[Architecture/DDD]]
- [[PHP/SymfonyServiceContainer]]
- [[GO/EstruturaDeProjetoGo]]
