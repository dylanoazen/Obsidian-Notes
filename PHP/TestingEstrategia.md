---
tags: [testing]
status: draft
---
# Estratégia de Testes

## Pirâmide
```
       /  E2E  \         Poucos, lentos, caros
      /Integrat.\        Médio
     /   Unit    \       Muitos, rápidos, baratos
```

## O Que Testar em Cada Camada

| Camada | Tipo | O que testa | Infra? |
|--------|------|-------------|--------|
| Domain | Unit | Regras, entities, VOs | Nenhuma |
| Application | Unit | Use cases, handlers | Fake repos |
| Infrastructure | Integration | Repos reais, APIs | Banco, RabbitMQ |
| HTTP | Functional | Endpoint completo | Tudo |

## Object Mother / Factory
```php
class AppointmentFactory {
    public static function pending(): Appointment {
        return Appointment::create(DoctorId::generate(), PatientId::generate(), Slot::available());
    }
    public static function confirmed(): Appointment {
        $a = self::pending(); $a->confirm(); return $a;
    }
}
```

## Links
- [[PHP/TDD]] · [[PHP/Testing]] · [[Architecture/PortsAndAdaptersPHP]]
