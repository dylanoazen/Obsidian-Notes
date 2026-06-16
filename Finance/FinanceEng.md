# Engenharia Financeira

O que é engenharia financeira e o que um engenheiro financeiro faz no dia a dia.

Related: [[Finance/CoreBanking]], [[Finance/MeiosDePagamento]]

---

## O que é

Engenharia Financeira (ou Engenharia de Pagamentos) é a área de engenharia que constrói e mantém sistemas financeiros — processamento de pagamentos, transferências, ledgers, integrações com bancos e redes de pagamento.

Diferente de engenharia de software comum, os erros têm consequência financeira direta: dinheiro some, duplica, ou fica preso.

---

## Responsabilidades Típicas

- Construir e manter APIs de pagamento
- Integrar com adquirentes, bancos, redes (Visa, Mastercard, PIX, SWIFT)
- Garantir idempotência e atomicidade nas transações
- Reconciliação — comparar o que o sistema registrou com o que o banco registrou
- Conformidade regulatória — KYC (Conheça seu Cliente), AML (Prevenção à Lavagem de Dinheiro), PCI-DSS
- Observabilidade — monitorar transações em tempo real, detectar anomalias
- Alta disponibilidade — sistema financeiro não pode ter tempo de inatividade

---

## Conceitos Essenciais

### Idempotência
Processar a mesma operação duas vezes não deve ter efeito duplo. Fundamental em pagamentos — retries são inevitáveis.
→ [[DistributedSystems/Idempotency]]

### Atomicidade
Débito e crédito acontecem juntos ou não acontecem. Sem estado intermediário visível.
→ [[Finance/CoreBanking]]

### Reconciliação
Comparar o ledger interno com o extrato do banco/adquirente. Qualquer diferença é uma discrepância que precisa de investigação.

```
Ledger interno: R$ 10.000 processados hoje
Extrato banco:  R$ 9.950 recebidos hoje
Diferença:      R$ 50 — onde está?
```

### Liquidação
O dinheiro não se move instantaneamente — existe um ciclo de liquidação. Um pagamento aprovado às 14h pode ser liquidado (realmente transferido) só no dia seguinte (D+1) ou em até alguns dias (D+2, D+3).

```
Compra aprovada → D+0  (autorização)
Captura         → D+0  (confirmação)
Liquidação      → D+1  (dinheiro move de verdade)
```

### Chargeback (Contestação)
Cliente contesta uma cobrança no cartão. O dinheiro é revertido ao cliente e o comerciante perde. Engenheiros financeiros constroem sistemas para detectar e responder a contestações.

---

## Tecnologias Comuns em Fintechs

| Área | Tecnologias |
|---|---|
| Backend | Java, Go, Kotlin, Python |
| Banco de dados | PostgreSQL, MySQL (ACID), + Redis para cache |
| Mensageria | Kafka, RabbitMQ (eventos de transação) |
| Observabilidade | Datadog, New Relic, Prometheus + Grafana |
| Nuvem | AWS (dominante), GCP |
| Conformidade | Ferramentas KYC/AML (Jumio, ComplyAdvantage) |

---

## O que Diferencia de Outras Áreas

| Engenharia Comum | Engenharia Financeira |
|---|---|
| Bug → experiência ruim | Bug → perda financeira |
| Retry é OK | Retry pode duplicar cobrança |
| Consistência eventual é OK | Consistência financeira é crítica |
| Deploy a qualquer hora | Deploy em janelas controladas |
| Log é opcional | Trilha de auditoria é obrigatória |

---

## Related

- [[Finance/CoreBanking]]
- [[Finance/MeiosDePagamento]]
- [[DistributedSystems/Idempotency]]
- [[DistributedSystems/Concurrency]]

#### My commentaries
-
