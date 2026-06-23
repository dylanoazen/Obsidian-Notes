# Entrevista Fintech — Cards Mentais

Arquiteturas e respostas para entrevistas técnicas em empresas de pagamentos e sistemas financeiros.

Related: [[Finance/CoreBanking]], [[Finance/FinanceEng]], [[Finance/MeiosDePagamento]]

---

## O que Tech Leads valorizam

- **Por que** — a razão da decisão, não só o que foi feito
- **Trade-offs** — o que você abriu mão, o que ganhou
- **Impacto no negócio** — não na tecnologia
- **Ambiguidade** — como agiu quando o requisito não estava claro
- **Honestidade intelectual** — não sabe → diz e explica como aprenderia

---

## Mapa de Arquiteturas

```
Sistemas de Pagamento
         │
         ├── Core Banking
         │     ├── Ledger append-only
         │     ├── Double-entry bookkeeping
         │     └── Reconciliação
         │
         ├── Atomicidade
         │     ├── Transação SQL (mesmo banco)
         │     ├── Eventos de domínio / Kafka (dois serviços)
         │     └── Saga Pattern (múltiplos serviços, fluxo longo)
         │
         ├── Concorrência
         │     ├── SELECT FOR UPDATE (conflitos frequentes)
         │     ├── Operação atômica no banco (simples)
         │     ├── Optimistic Locking / version field (conflitos raros)
         │     └── Lock distribuído Redis NX/EX (sem banco compartilhado)
         │
         ├── Idempotência
         │     ├── Idempotency Key (UUID do cliente)
         │     ├── Armazenado no Redis com TTL
         │     └── Check-before-process com lock
         │
         ├── Observabilidade
         │     ├── Logs estruturados (JSON)
         │     ├── Métricas (Counter / Gauge / Histogram)
         │     └── Traces (caminho da request)
         │
         └── Mensageria / Event-driven
               ├── Kafka — eventos de transação
               ├── Entrega ao menos uma vez → consumidor idempotente
               └── Consistência eventual
```

---

## Cards Mentais de Resposta

---

### Core Banking

**"O que é core banking?"**

> "É o sistema central que mantém o estado financeiro de todas as contas — registra todas as operações em um ledger imutável, garante que débito e crédito sempre se equilibram, e é a fonte da verdade para saldo e histórico. Todo o resto — apps, APIs, integrações — conversa com o core banking para ler ou modificar estado financeiro."

---

### Ledger

**"O que é um ledger?"**

> "É um log append-only de transações — nunca edita, nunca deleta, só adiciona. O saldo atual de uma conta é a soma de todas as entradas dela. Se o saldo calculado no ledger não bater com o extrato do banco, tem bug ou fraude — é a base da auditoria financeira."

---

### Double-Entry Bookkeeping

**"O que é partidas dobradas?"**

> "Para cada débito existe um crédito equivalente de mesmo valor. Uma transferência de R$100 da conta A para B gera dois registros: débito de R$100 na A e crédito de R$100 na B. O saldo líquido do sistema é sempre zero — dinheiro não some nem aparece do nada. Qualquer diferença indica erro ou fraude."

---

### Atomicidade

**"Como você garantiria atomicidade em uma transferência?"**

> "Depende do contexto. Se origem e destino estão no mesmo banco de dados, uso transação SQL — BEGIN, UPDATE os dois saldos, COMMIT. Se são serviços separados, uso eventos de domínio: debito a origem, publico um evento no Kafka, o serviço de destino escuta e credita. Se o fluxo passa por múltiplos serviços externos, uso Saga Pattern com compensações manuais para cada passo."

```
mesmo banco              → transação SQL
dois serviços            → eventos de domínio (Kafka)
múltiplos serviços       → Saga Pattern com compensações
single-thread (ReactPHP) → nenhum — event loop garante
```

---

### Race Condition

**"O que é race condition e como você evita?"**

> "Dois processos leem o mesmo saldo antes de qualquer um gravar — os dois veem R$100, os dois aprovam saques de R$80 e R$90, e o cliente saca R$170 de uma conta com R$100. Para evitar: SELECT FOR UPDATE trava o registro no banco enquanto processa; operação atômica delega o cálculo ao banco sem lock explícito; Optimistic Locking adiciona um campo version e o UPDATE só acontece se a versão não mudou; lock distribuído no Redis para sistemas sem banco compartilhado."

---

### Idempotência

**"O que é idempotência e por que importa em pagamentos?"**

> "Processar a mesma operação duas vezes não deve ter efeito duplo. Em pagamentos, retries são inevitáveis — rede cai, timeout acontece, cliente reenvia. A solução é a Idempotency Key: o cliente gera um UUID único por operação e envia no header. O servidor armazena essa key no Redis com o resultado — se chegar de novo, retorna o resultado já salvo sem reprocessar. A check + write precisa de lock distribuído para evitar race condition entre dois requests idênticos simultâneos."

---

### Saga Pattern

**"Me explica o Saga Pattern."**

> "Saga é para transações longas que passam por múltiplos serviços. Cada passo tem uma operação e uma compensação — o desfazer. Se o passo 3 falha, as compensações dos passos 1 e 2 são executadas em ordem inversa e o estado volta ao início. Não tem ROLLBACK automático do banco — você escreve as compensações manualmente. O estado do Saga precisa ser persistido para que, se o orquestrador cair no meio, saiba onde parou e o que ainda precisa ser compensado."

---

### Observabilidade

**"Como você monitoraria um sistema de pagamentos em produção?"**

> "Três pilares: logs estruturados em JSON para rastrear erros individuais com request_id. Métricas para monitorar em tempo real: taxa de sucesso, latência P99, volume por método. Traces para identificar gargalos em operações lentas. Alertas: error_rate > 1% dispara PagerDuty, p99_latency > 500ms é warning, > 2s é crítico."

```
payment_success_rate        → % aprovações
payment_processing_time_p99 → pior 1% das transações
duplicate_payment_attempts  → tentativas de duplicata detectadas
external_api_error_rate     → erros em APIs de adquirentes/bancos
chargeback_rate             → % de contestações
```

---

### Consistência Eventual

**"Quando você aceitaria consistência eventual vs consistência forte?"**

> "Em saldo de conta — nunca. Saldo precisa ser consistente — não posso aprovar um saque baseado em saldo desatualizado. Em histórico de transações para exibição — aceito. O extrato pode ter 200ms de delay para aparecer no app. A regra: se o dado influencia uma decisão financeira, consistência forte; se é leitura informacional, eventual é OK."

---

### Reconciliação

**"O que é reconciliação e por que é importante?"**

> "Comparar o ledger interno com o extrato do banco ou adquirente. Se o sistema registrou R$10.000 processados e o banco diz R$9.950, tem R$50 de diferença — pode ser fee, timing de liquidação ou bug. Sem reconciliação, discrepâncias se acumulam silenciosamente. Em fintechs sérias, roda automaticamente e qualquer diferença acima de threshold dispara alerta."

---

## Lembrar na Entrevista

- Terminar a história sempre com **impacto no negócio**, não na tecnologia
- Se não souber → dizer que não sabe + **como aprenderia**
- Perguntas para fazer no final mostram que você pensa em escala

---

#### My commentaries
-
