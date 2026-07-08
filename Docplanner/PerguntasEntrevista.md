---
tags: [docplanner-prep, entrevista]
status: draft
---
# Top 20 Perguntas Backend — Docplanner

## Arquitetura
1. **Hexagonal?** → Domain no centro sem deps. Ports = interfaces. Adapters = implementações. Deps apontam pra dentro.
2. **Why DDD?** → Complex domain (healthcare). Code structured around business language.
3. **Bounded Context?** → Boundary where a model is consistent. Different contexts, different models.

## Symfony
4. **Autowiring?** → Reads constructor type hints, resolves from container automatically.
5. **Doctrine vs Eloquent?** → Data Mapper vs Active Record. Entities are pure objects.
6. **Async processing?** → Messenger + RabbitMQ. Message DTO → Handler. Routing config decides sync/async.

## Testing
7. **TDD workflow?** → Red-Green-Refactor.
8. **Mock vs Fake?** → Mock: verify interactions. Fake: test real state. Prefer fakes for domain.
9. **Testing hexagonal?** → Domain: unit with fakes. Infra: integration with real DB. HTTP: functional.

## Patterns
10. **SOLID in practice?** → SRP: thin controllers. DIP: depend on ports not adapters.
11. **When NOT DDD?** → Simple CRUD. DDD adds complexity only justified by complex domains.

## Data
12. **Money?** → Integers in cents, never float. Value Object with amount + currency.
13. **Idempotency?** → Same request, same result. Idempotency keys prevent double processing.

## Infra
14. **RabbitMQ exchanges?** → Direct: exact match. Fanout: broadcast. Topic: wildcards.
15. **ES vs SQL?** → ES for search, SQL for transactions. Both together.
16. **Liveness vs Readiness?** → Liveness: alive? restart. Readiness: ready? remove from LB.

## Behavioral
17. **Proud decision?** → Hexagonal architecture in tray-go from day one. ReactPHP for EBANX challenge.
18. **Code review disagreements?** → Focus on the code. Ask "what problem does this solve?" Defer to code owner.
19. **AI tools?** → Claude daily for code review, study notes, automations. Built an Obsidian inbox via Claude API.
20. **Why Docplanner?** → Worked at Doctoralia Brazil, know the product. Barcelona + Symfony/DDD is the right next step.

## Links
- [[Docplanner/SymfonyVsLaravel]] · [[Docplanner/DDDFundamentos]] · [[Architecture/HexagonalArchitecture]]
