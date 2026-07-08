---
tags: [docplanner-prep, mensageria, infra]
status: draft
---
# RabbitMQ

## Conceitos Core
```
Producer ──► Exchange ──► Queue ──► Consumer
```

## Exchanges
- **Direct**: routing key exato (appointment.confirmed → queue específica)
- **Topic**: wildcards (appointment.* → pega tudo de appointment)
- **Fanout**: broadcast pra todas as queues bindadas
- **Headers**: match por headers da mensagem

## Ack/Nack
- **ACK**: processou OK → RabbitMQ remove da fila
- **NACK + requeue**: falhou → volta pra fila (retry)
- **NACK + dead letter**: falhou demais → vai pra DLQ

## Dead Letter Queue (DLQ)
Mensagens que falharam N vezes. Permite investigação manual ou reprocessamento.

## Quando Mensageria vs Síncrono?
| Síncrono | Assíncrono |
|----------|-----------|
| Precisa resposta agora | Não precisa |
| Rápido (<100ms) | Lento (email, PDF) |
| Falha para o fluxo | Falha pode ser retried |

## Garantias
- **Publisher confirms**: broker confirma que recebeu
- **Persistent messages**: sobrevivem restart do broker
- **Consumer ACK manual**: só remove quando processado
- **Mirrored queues**: replicação pra HA

## Links
- [[Docplanner/SymfonyMessenger]] · [[Docplanner/DomainEvents]]
