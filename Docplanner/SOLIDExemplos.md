---
tags: [docplanner-prep, solid]
status: draft
---
# SOLID com Exemplos Symfony

## S — Thin controller, focused services
## O — Novo Notifier sem alterar existente (interface)
## L — Subtipos substituíveis sem quebrar
## I — Interfaces pequenas e específicas
## D — Depend on ports, not adapters

Já coberto em detalhe com código → [[PHP/SOLID]]
Aplicação prática → [[Docplanner/PortsAndAdaptersPHP]]

## Anti-patterns
- Service com 20 deps → viola SRP
- `new ConcreteClass()` dentro de service → viola DIP
- God class → viola tudo

## Links
- [[PHP/SOLID]] · [[Docplanner/PortsAndAdaptersPHP]]
