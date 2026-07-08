---
tags: [docplanner-prep, plano]
status: active
---

# Plano de Preparação — Docplanner Barcelona

**Status:** No pipeline, sem data
**Dedicação:** 2h+/dia
**Inglês:** Confortável, sem prática
**Duração:** 4 semanas (ajustável se a entrevista chegar antes)

---

## Semana 1 — Symfony + Inglês (fundação)

O gap mais crítico. Tu manja Laravel, eles usam Symfony. Sem isso nada rola.

### Dia 1-2: Symfony vs Laravel (mental model)
- [ ] Instalar Symfony 6.4 com `symfony new --webapp`
- [ ] Criar um CRUD simples de "Patients" (Doctrine + Controller)
- [ ] Entender o fluxo: Request → Router → Controller → Response
- [ ] **Mapear:** `php artisan` → `bin/console`, `.env` → `.env.local`, `config/app.php` → `config/services.yaml`
- [ ] **Obsidian:** nota [[Docplanner/SymfonyVsLaravel]]

### Dia 3-4: Service Container + Doctrine
- [ ] Autowiring: como Symfony resolve dependências sem `app()`
- [ ] Criar um Service, injetar no controller, usar interface (já pensa em Ports)
- [ ] Doctrine: Entity, Repository, Migrations. Entender Data Mapper vs Active Record
- [ ] **Obsidian:** notas [[Docplanner/SymfonyServiceContainer]] + [[Docplanner/DoctrineORM]]

### Dia 5-6: API Platform + Messenger
- [ ] Expor o CRUD de Patients via API Platform (zero controller code)
- [ ] Serialization groups, filtros, paginação
- [ ] Symfony Messenger: criar um command async, configurar transport (pode ser doctrine pra começar)
- [ ] **Obsidian:** notas [[Docplanner/APIPlatform]] + [[Docplanner/SymfonyMessenger]]

### Dia 7: Revisão + Inglês
- [ ] Rever tudo da semana
- [ ] **Inglês:** gravar áudio de 5min explicando o que aprendeu, em inglês
- [ ] Começar rotina: todo conteúdo técnico consumido em inglês (docs, YouTube, podcasts)

### Inglês diário (30min, todo dia da semana 1-4)
- [ ] Shadowing: assistir 1 tech talk da Docplanner/Symfony em inglês, repetir frases
- [ ] Escrever 1 parágrafo por dia explicando um conceito técnico em inglês
- [ ] Praticar falar sozinho: "I chose X over Y because Z"

---

## Semana 2 — DDD + Testes (arquitetura)

Aqui tu reforça o que já sabe e transforma em vocabulário de entrevista.

### Dia 8-9: DDD aplicado no Symfony
- [ ] Reestruturar o projeto da semana 1 em Bounded Contexts
- [ ] Criar: Entity, Value Object, Domain Event, Repository Interface (Port)
- [ ] Implementar o Adapter (Doctrine Repository)
- [ ] **Obsidian:** nota [[Docplanner/DDDFundamentos]]
- [ ] Comparar estrutura com o tray-go: documentar paralelos

### Dia 10-11: Ports & Adapters na prática
- [ ] Estrutura de pastas: `Domain/`, `Application/`, `Infrastructure/`
- [ ] Application Service (Use Case) orquestrando domain logic
- [ ] Controller é Adapter, Doctrine Repository é Adapter, RabbitMQ publisher é Adapter
- [ ] **Obsidian:** nota [[Docplanner/PortsAndAdaptersPHP]]

### Dia 12-13: TDD
- [ ] Escrever testes ANTES do código pra um novo Use Case
- [ ] PHPUnit: test doubles (mock do Repository Interface)
- [ ] Testar domínio SEM framework, SEM banco
- [ ] **Exercício:** implementar um "BookAppointment" use case com TDD puro
- [ ] **Obsidian:** notas [[Docplanner/TDDPHP]] + [[Docplanner/TestingEstrategia]]

### Dia 14: Mock interview #1 (solo)
- [ ] Gravar em inglês (celular, 15min)
- [ ] Perguntas:
  - "Tell me about your experience with PHP and backend development"
  - "How do you structure a new backend service?"
  - "Describe a challenging technical problem you solved recently"
- [ ] Reouvir, anotar pontos fracos

---

## Semana 3 — Infra + AI narrative (diferencial)

### Dia 15-16: RabbitMQ + Redis
- [ ] Subir RabbitMQ no Docker, configurar Symfony Messenger com transport `amqp`
- [ ] Criar um fluxo: API recebe request → dispatcha message → worker processa async
- [ ] Redis: configurar cache no Symfony, session store
- [ ] **Obsidian:** notas [[Docplanner/RabbitMQ]] + [[Docplanner/RedisPatterns]]

### Dia 17-18: Docker + CI/CD
- [ ] Dockerizar o projeto Symfony (PHP-FPM + Nginx + MariaDB + RabbitMQ + Redis)
- [ ] Criar GitHub Actions pipeline: lint (phpstan) → test (phpunit) → build
- [ ] **Obsidian:** notas [[Docplanner/KubernetesEssencial]] + [[Docplanner/GitHubActionsCI]]

### Dia 19-20: Preparar AI narrative
- [ ] Documentar 3 casos concretos de uso de AI no trabalho:
  1. **Migração Aplicsil→ERP:** mapping doc → Claude CLI → SQL generation pipeline
  2. **tray-go:** build briefs em markdown, delegação estruturada pro Claude Code
  3. **Estudo sistemático:** Obsidian vault com 35+ notas, GitHub API push
- [ ] Para cada caso, preparar em inglês:
  - Problema → Solução → Resultado → O que a AI fez vs o que EU fiz
  - Frase-chave: "I use AI as an engineering tool, not a crutch"

### Dia 21: Mock interview #2 (solo)
- [ ] Gravar em inglês (20min)
- [ ] Perguntas:
  - "How do you use AI in your development workflow?"
  - "Walk me through how you'd design a patient-therapist matching system"
  - "Tell me about a time you received critical feedback and how you handled it" (EBANX story)
  - "What's the difference between Ports & Adapters and traditional MVC?"

---

## Semana 4 — Simulação + Polish

### Dia 22-23: SOLID + Design Patterns
- [ ] Revisar cada princípio com exemplo do projeto Symfony
- [ ] Patterns mais relevantes: Repository, Strategy, Observer, Factory
- [ ] **Obsidian:** notas [[Docplanner/SOLIDExemplos]] + [[Docplanner/DesignPatterns]]

### Dia 24-25: System Design (healthcare context)
- [ ] Praticar desenho de sistema no papel/whiteboard:
  - "Design a booking system for doctors and patients"
  - "How would you handle real-time availability updates?"
  - "How would you scale a notification system for appointment reminders?"
- [ ] Pensar em: API design, banco, cache, filas, failure modes
- [ ] Praticar explicar em inglês enquanto desenha

### Dia 26-27: Mock interview #3 (com alguém)
- [ ] Pedir pro André ou outro dev fazer mock interview em inglês
- [ ] Se não rolar, usar Claude com prompt de entrevistador
- [ ] Simular os 3 estágios:
  1. **Cultural fit:** motivação, por que Docplanner, por que Barcelona
  2. **Técnica:** live coding ou exercício prático
  3. **Final:** expectativas, crescimento, perguntas pro entrevistador
- [ ] Preparar 3 perguntas para FAZER ao entrevistador:
  - "How does the Terapia team handle domain boundaries with the main marketplace?"
  - "What does the on-call rotation look like?"
  - "How do you balance moving fast with code quality in an early-stage product?"

### Dia 28: Revisão final
- [ ] Reler todas as notas do Obsidian
- [ ] Revisar os 3 áudios gravados: evolução de fluência
- [ ] Checklist final (abaixo)

---

## Checklist de prontidão

### Técnico
- [ ] Consigo criar um CRUD em Symfony sem consultar docs a cada 2 min
- [ ] Sei explicar DDD (Bounded Context, Aggregate, Value Object) com exemplo
- [ ] Sei explicar Ports & Adapters e por que é melhor que MVC acoplado
- [ ] Escrevi pelo menos 1 feature inteira com TDD
- [ ] Entendo RabbitMQ: exchange types, dead letter queue, ack/nack
- [ ] Sei o básico de Kubernetes: pod, deployment, service

### Narrativa
- [ ] Tenho 3 histórias prontas em inglês (AI workflow, EBANX feedback, tray-go architecture)
- [ ] Cada história segue STAR: Situation, Task, Action, Result
- [ ] Sei explicar por que quero Docplanner Barcelona especificamente
- [ ] Tenho perguntas inteligentes pra fazer ao entrevistador

### Inglês
- [ ] Consigo falar 5min seguidos sobre arquitetura sem travar
- [ ] Sei os termos técnicos em inglês: tradeoff, coupling, cohesion, throughput, latency
- [ ] Pratiquei "thinking out loud" — explicar raciocínio enquanto resolvo problema

---

## Scripts prontos (respostas-chave em inglês)

### "Tell me about yourself"
> I'm a backend developer based in Brazil with 3+ years of experience building and maintaining ERP systems using PHP, Laravel, and MariaDB. My day-to-day involves marketplace integrations, database migrations, and API design. I'm currently rewriting a legacy e-commerce integration in Go using hexagonal architecture, which has deepened my understanding of DDD and clean separation of concerns. I also heavily integrate AI tools into my workflow — not just for code generation, but as a structured engineering tool for migrations, documentation, and learning. I previously worked at Doctoralia Brazil, so I already know the company's mission and culture.

### "Why Docplanner?"
> Two reasons. First, I already worked at Doctoralia in Brazil and experienced the culture and mission firsthand — I believe in making healthcare more accessible. Second, the Terapia product is early-stage, which means high ownership, fast feedback loops, and real impact. I thrive in environments where I can shape the product, not just execute tickets. Barcelona is also where your tech HQ is, and working alongside the core engineering team would accelerate my growth significantly.

### "Tell me about a time you received critical feedback"
> My first submission to a coding challenge was rejected because I mixed business logic with the HTTP layer and didn't include tests. Instead of being defensive, I analyzed the feedback, identified the root cause — I was thinking in terms of "make it work" instead of "make it maintainable" — and completely redesigned the solution. The v2 had clean domain/HTTP separation, full PHPUnit coverage, and an incremental 8-commit history showing the design evolution. That experience fundamentally changed how I approach architecture: I now start from the domain layer and work outward, and I write tests as a design tool, not an afterthought.

### "How do you use AI in your workflow?"
> I use AI as an engineering tool integrated into my process, not as a replacement for thinking. Three concrete examples: First, when migrating a client's database between two different ERPs — different schemas, different encodings — I created a mapping document as a spec and used Claude CLI to generate migration scripts. The AI handled repetitive SQL generation while I designed the pipeline and handled edge cases. Second, I write structured build briefs in markdown for complex features, which I then delegate to AI for implementation while I review and iterate. Third, I maintain a structured knowledge vault with 35+ technical notes that I push to GitHub via API — treating learning as a system, not something ad hoc.

---

## Links
- [[Docplanner/PerguntasEntrevista]]
- [[Docplanner/SymfonyVsLaravel]]
- [[Docplanner/DDDFundamentos]]
- [[Docplanner/PortsAndAdaptersPHP]]
- [[Architecture/HexagonalArchitecture]]
