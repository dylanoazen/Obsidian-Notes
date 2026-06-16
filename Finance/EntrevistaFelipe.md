# Entrevista Felipe — EBANX Tech Lead

Prep completo para entrevista técnica/comportamental com o tech lead. Arquiteturas, cards mentais de resposta e histórias do Acesse.

Related: [[Finance/CoreBanking]], [[Finance/EBANX]], [[Finance/FinanceEng]], [[PHP/EBANX-Entrevista]]

---

## O que Felipe valoriza (dicas do RH)

- **Por que** — a razão da decisão, não só o que foi feito
- **Trade-offs** — o que você abriu mão, o que ganhou
- **Impacto no negócio** — não na tecnologia
- **Ambiguidade** — como agiu quando o requisito não estava claro
- **Honestidade intelectual** — não sabe → diz e explica como aprenderia

---

## Mapa de Arquiteturas — O que Preciso Saber

```
EBANX — Sistemas de Pagamento
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

> Formato: pergunta provável → resposta de 3 frases que fecha o assunto.

---

### Core Banking

**"O que é core banking?"**

> "É o sistema central que mantém o estado financeiro de todas as contas — registra todas as operações em um ledger imutável, garante que débito e crédito sempre se equilibram, e é a fonte da verdade para saldo e histórico. Todo o resto — apps, APIs, integrações — conversa com o core banking para ler ou modificar estado financeiro."

---

### Ledger

**"O que é um ledger?"**

> "É um log append-only de transações — nunca edita, nunca deleta, só adiciona. O saldo atual de uma conta é a soma de todas as entradas dela. Se o saldo calculado no ledger não bater com o extrato do banco, tem bug ou fraude — é a base da auditoria financeira."

**Conexão pessoal:**
> "Quando refatorei o motor de créditos no Acesse, fiz exatamente isso sem saber o nome — transformei um sistema que editava registros em um que só adiciona entradas. Quando estudei core banking para essa entrevista reconheci o padrão."

---

### Double-Entry Bookkeeping

**"O que é partidas dobradas?"**

> "Para cada débito existe um crédito equivalente de mesmo valor. Uma transferência de R$100 da conta A para B gera dois registros: débito de R$100 na A e crédito de R$100 na B. O saldo líquido do sistema é sempre zero — dinheiro não some nem aparece do nada. Qualquer diferença indica erro ou fraude."

---

### Atomicidade

**"Como você garantiria atomicidade em uma transferência?"**

> "Depende do contexto. Se origem e destino estão no mesmo banco de dados, uso transação SQL — BEGIN, UPDATE os dois saldos, COMMIT. Se são serviços separados, uso eventos de domínio: debito a origem, publico um evento no Kafka, o serviço de destino escuta e credita. O evento fica na fila até ser processado — consistência eventual, mas o sistema se corrige. Se o fluxo passa por múltiplos serviços externos (SWIFT, por exemplo), uso Saga Pattern com compensações manuais para cada passo."

**Tabela mental:**
```
mesmo banco             → transação SQL
dois serviços           → eventos de domínio (Kafka)
múltiplos serviços      → Saga Pattern com compensações
single-thread (ReactPHP)→ nenhum — event loop garante
```

---

### Race Condition

**"O que é race condition e como você evita?"**

> "Dois processos leem o mesmo saldo antes de qualquer um gravar — os dois veem R$100, os dois aprovam saques de R$80 e R$90, e o cliente saca R$170 de uma conta com R$100. Para evitar: SELECT FOR UPDATE trava o registro no banco enquanto processa; operação atômica (`balance - amount WHERE balance >= amount`) delega o cálculo ao banco sem lock explícito; Optimistic Locking adiciona um campo version e o UPDATE só acontece se a versão não mudou; lock distribuído no Redis (NX/EX) para sistemas sem banco compartilhado."

---

### Idempotência

**"O que é idempotência e por que importa em pagamentos?"**

> "Processar a mesma operação duas vezes não deve ter efeito duplo. Em pagamentos, retries são inevitáveis — rede cai, timeout acontece, cliente reenvia. A solução é a Idempotency Key: o cliente gera um UUID único por operação e envia no header. O servidor armazena essa key no Redis com o resultado — se chegar de novo, retorna o resultado já salvo sem reprocessar. A check + write precisa de lock distribuído para evitar race condition entre dois requests idênticos simultâneos."

---

### Saga Pattern

**"Me explica o Saga Pattern."**

> "Saga é para transações longas que passam por múltiplos serviços. Cada passo tem uma operação e uma compensação — o desfazer. Se o passo 3 falha, as compensações dos passos 1 e 2 são executadas em ordem inversa e o estado volta ao início. Não tem ROLLBACK automático do banco — você escreve as compensações manualmente. O exemplo clássico é uma transferência internacional: debita origem, converte moeda, envia via SWIFT, credita destino. Se o SWIFT rejeitar, você desfaz a conversão e credita de volta a origem."

---

### Observabilidade

**"Como você monitoraria um sistema de pagamentos em produção?"**

> "Três pilares: logs estruturados em JSON para rastrear erros individuais com request_id — permite reconstruir o que aconteceu em qualquer transação. Métricas para monitorar em tempo real: taxa de sucesso de pagamentos, latência P99, volume de transações por método. Traces para identificar onde está o tempo em operações lentas. Alertas configurados: error_rate > 1% dispara PagerDuty para o time de plantão, p99_latency > 500ms é warning, > 2s é crítico. Usaria Datadog ou Prometheus+Grafana dependendo do orçamento."

**Métricas essenciais em pagamentos:**
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

> "Em saldo de conta — nunca. Saldo precisa ser consistente — não posso aprovar um saque baseado em saldo desatualizado. Em histórico de transações para exibição — aceito. O extrato pode ter 200ms de delay para aparecer no app, sem impacto financeiro. Em notificações e analytics — sempre eventual. A regra: se o dado influencia uma decisão financeira, consistência forte; se é leitura informacional, eventual é OK."

---

### Reconciliação

**"O que é reconciliação e por que é importante?"**

> "Comparar o ledger interno com o extrato do banco ou adquirente. O EBANX registra que processou R$10.000 hoje; o banco diz que recebeu R$9.950. Tem R$50 de diferença — onde está? Pode ser fee, timing de liquidação ou bug. Sem reconciliação, discrepâncias se acumulam silenciosamente. Em fintechs sérias, reconciliação roda automaticamente e qualquer diferença acima de threshold dispara alerta imediato."

---

### Sobre o Take-Home

**"Me fala sobre as decisões do seu projeto."**

> Ver [[PHP/EBANX-Entrevista]] — cards detalhados de cada decisão técnica e possíveis alterações que o sênior pode solicitar.

**Resumo rápido:**
```
ReactPHP        → processo de longa duração, estado em memória entre requests
int (centavos)  → float tem erro de representação binária (0.1+0.2 ≠ 0.3)
Atomicidade     → single-thread, event loop garante — impossível ter concorrência
Exceções tipadas→ domínio puro, controller traduz para HTTP
Repository      → isola persistência, trocar implementação sem mudar o service
```

---

## Histórias do Acesse — Situação → Decisão → Impacto

---

### Motor de Créditos (Ledger)

**Situação:** Sistema editava e deletava registros diretamente — inconsistências constantes, chamados frequentes sobre créditos incorretos.

**Decisão:** Reescrevi do zero com modelo append-only. Nenhuma operação apaga ou edita — só adiciona. Cada movimentação vira um registro permanente.

**Impacto:** Eliminou os chamados. Rastreabilidade completa — qualquer crédito pode ser auditado entrada por entrada.

**Conexão EBANX:**
> "Quando estudei core banking para essa entrevista, reconheci o que tinha feito — um ledger append-only. Na época não chamei assim, mas o problema e a solução são os mesmos que sistemas financeiros usam para garantir integridade."

**Trade-off:** Reescrever vs incrementar → o déficit técnico era tão grande que qualquer incremento carregaria os mesmos problemas estruturais.

---

### Integrações Bancárias (Sicoob, Itaú, Inter)

**Situação:** Primeiras integrações bancárias — documentação oficial não explicava o comportamento real do endpoint.

**Decisão:** Recorri a um fórum de ERP em Delphi que integrava o mesmo banco. Aprendi a testar exaustivamente em homologação e nunca assumir que a API se comporta como documentado.

**Impacto:** Três integrações em produção, abrindo novos canais de pagamento.

**Conexão EBANX:**
> "O EBANX faz exatamente isso em escala — dezenas de métodos de pagamento locais, cada um com suas peculiaridades. Entendo o desafio de API que não se comporta como esperado."

---

### Painel de Preços Multi-Empresa

**Situação:** Preços variavam entre filiais sem critério — o time de pricing tinha estratégia de margens mas não conseguia aplicar porque não havia centralização.

**Decisão:** Painel centralizado onde o pricing define margens por estado e produto. Todas as filiais herdam automaticamente.

**Impacto:** O pricing ganhou controle real sobre a estratégia que antes existia só no papel.

**Visão de negócio:**
> "O problema não era técnico — era que uma decisão estratégica de negócio não tinha como ser executada sem um sistema que a suportasse."

---

### Sistema de Fábrica (30 dias, sem ponto focal)

**Situação:** Sistema completo de produção de uma fábrica em 30 dias. Sem ponto focal disponível para tirar dúvidas.

**Decisão:** Aprendi o processo produtivo sozinho, priorizei o fluxo crítico (ordem de produção e saída) e deixei periféricos para depois.

**Impacto:** Entregue no prazo, cobrindo todo o fluxo produtivo.

**Ambiguidade:**
> "Quando o requisito não estava claro, definia uma hipótese, implementava e validava depois — ciclo curto em vez de esperar aprovação que nunca chegava."

---

## Perguntas Rápidas e Resposta Curta

---

**"Por que você quer trabalhar no EBANX?"**
> "Trabalho com sistemas financeiros há anos no Acesse — crédito, integrações bancárias, controle de preços. Quero ir para um ambiente onde esses problemas são o core, não uma parte. O desafio de integrar múltiplos mercados com métodos completamente diferentes é o tipo de complexidade que me interessa."

---

**"O que você sabe sobre os desafios do EBANX?"**
> "Cada país tem regulação, moeda e métodos diferentes. PIX e boleto não existem no México. O sistema precisa abstrair essas diferenças para o cliente global sem expor a complexidade local — múltiplas integrações, consistência de dados entre países e compliance regulatório de cada mercado simultâneo."

---

**"Tema que você não domina bem?"**
> "Microsserviços em escala real — conheço os conceitos, Kafka, Saga, consistência eventual, mas nunca operei um sistema distribuído grande em produção. É uma das razões pelas quais me interessa o EBANX — quero construir essa experiência em um ambiente onde esses problemas são reais."

---

**"Me faz uma pergunta."** (perguntas para fazer no final)
- "Qual é o maior desafio técnico de escalar para um novo país hoje?"
- "Como é o processo de expansão — engenharia entra desde o início ou depois?"
- "Como vocês lidam com consistência de dados quando múltiplos métodos de pagamento de um mesmo país respondem em tempos diferentes?"

---

## Lembrar na Entrevista

- Terminar a história sempre com **impacto no negócio**, não na tecnologia
- Se não souber → dizer que não sabe + **como aprenderia**
- Conectar experiências do Acesse com os desafios do EBANX
- Mostrar curiosidade — fazer as perguntas no final

---

#### My commentaries
-
