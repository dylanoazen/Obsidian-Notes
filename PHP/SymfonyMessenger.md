---
tags: [symfony, mensageria]
status: draft
---

# Symfony Messenger

## Por que isso importa
Symfony Messenger é a abstração que conecta a aplicação ao broker de mensagens. Conecta com RabbitMQ, SQS, Redis para processar coisas assíncronas — emails, notificações, sincronização de dados.

## Conceitos

```
Message (DTO)                    "O que fazer"
    │
    ▼
Bus (MessageBus)                 "Roteador"
    │
    ├── sync → Handler            "Executa agora"
    └── async → Transport         "Manda pro broker"
                    │
                    ▼
              RabbitMQ / Redis    "Fila"
                    │
                    ▼
              Worker consome      "Executa depois"
                    │
                    ▼
              Handler             "Executa"
```

## Message + Handler

```php
// Message — é só um DTO
class SendAppointmentReminder {
    public function __construct(
        public readonly int $appointmentId,
        public readonly string $patientEmail,
    ) {}
}

// Handler — a lógica
#[AsMessageHandler]
class SendAppointmentReminderHandler {
    public function __construct(private MailerInterface $mailer) {}

    public function __invoke(SendAppointmentReminder $message): void {
        // envia email pro paciente
        $this->mailer->send(/* ... */);
    }
}
```

## Dispatch

```php
class AppointmentService {
    public function __construct(private MessageBusInterface $bus) {}

    public function confirmAppointment(int $id): void {
        // lógica de confirmação...
        
        // Dispara mensagem — sync ou async depende da config
        $this->bus->dispatch(new SendAppointmentReminder($id, $email));
    }
}
```

O service não sabe se é síncrono ou assíncrono. A decisão é na config.

## Transport Config

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'amqp://guest:guest@rabbitmq:5672/%2f'
                # ou: 'redis://localhost:6379/messages'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 2

        routing:
            App\Message\SendAppointmentReminder: async
            App\Message\SyncDoctorProfile: async
            App\Message\ValidatePayment: sync   # executa na hora
```

## Worker (consumer)

```bash
# Roda o worker que consome da fila
php bin/console messenger:consume async

# Em produção: supervisor ou systemd mantém rodando
# Com graceful shutdown:
php bin/console messenger:consume async --time-limit=3600 --memory-limit=128M
```

## Middleware

Middleware do Messenger processa a mensagem antes/depois do handler:

```php
class AuditMiddleware implements MiddlewareInterface {
    public function handle(Envelope $envelope, StackInterface $stack): Envelope {
        // antes do handler
        $this->logger->info('Processing: ' . get_class($envelope->getMessage()));
        
        $envelope = $stack->next()->handle($envelope, $stack);
        
        // depois do handler
        $this->logger->info('Done');
        
        return $envelope;
    }
}
```

## Failed Messages

```yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            failed: 'doctrine://default?queue_name=failed'
```

```bash
# Ver mensagens que falharam
php bin/console messenger:failed:show

# Retry
php bin/console messenger:failed:retry

# Rejeitar
php bin/console messenger:failed:remove 123
```

## Comparação com minha stack

| Laravel Queue | Symfony Messenger |
|--------------|-------------------|
| `dispatch(new Job)` | `$bus->dispatch(new Message)` |
| `implements ShouldQueue` | Routing no YAML |
| `php artisan queue:work` | `messenger:consume` |
| Failed jobs table | Failed transport |
| Horizon (dashboard) | Não tem built-in (use DataDog/Grafana) |

## Perguntas que podem cair
1. "Qual a diferença entre sync e async no Messenger?" → Config de routing. Sync executa na hora, async manda pro transport (fila).
2. "Como lidar com mensagens que falham?" → Retry strategy com backoff exponencial + failure transport pra mensagens mortas.
3. "O que é um transport?" → A conexão com o broker (RabbitMQ, Redis, Doctrine). Define onde as mensagens ficam armazenadas.
4. "Por que usar mensageria em vez de chamada síncrona?" → Desacoplamento, resiliência (retry), não bloqueia o request do usuário.

## Links
- [[DistributedSystems/RabbitMQ]]
- [[PHP/SymfonyServiceContainer]]
- [[Architecture/DomainEvents]]
