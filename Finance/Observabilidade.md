# Observabilidade

Como enxergar o que está acontecendo dentro de um sistema em produção em tempo real.

Related: [[Finance/FinanceEng]], [[Finance/CoreBanking]], [[DistributedSystems/Performance]]

---

## O que é

Observabilidade é a capacidade de entender o estado interno de um sistema a partir dos dados que ele expõe. Em produção você não pode abrir um debugger — você precisa de dados coletados continuamente.

```
Sistema em produção
        │
        ├── Logs      → o que aconteceu
        ├── Métricas  → números ao longo do tempo
        └── Traces    → caminho de uma request pelo sistema
```

Esses são os **três pilares** da observabilidade.

---

## Os Três Pilares

### 1. Logs — O que aconteceu

Registro de eventos em texto. Útil para investigar erros específicos.

```
[2024-01-15 14:32:01] ERROR Account "100" not found — request_id=abc123
[2024-01-15 14:32:02] INFO  Transfer completed — from=100 to=200 amount=50
[2024-01-15 14:32:03] WARN  High memory usage: 85%
```

**Logs estruturados** (JSON) são preferíveis — mais fáceis de filtrar e agregar:

```json
{
  "level": "error",
  "message": "Account not found",
  "account_id": "100",
  "request_id": "abc123",
  "timestamp": "2024-01-15T14:32:01Z"
}
```

### 2. Métricas — Números ao longo do tempo

Valores numéricos coletados continuamente. Útil para detectar tendências e anomalias.

```
requests_per_second: 1200
error_rate: 0.02%
p99_latency: 45ms
memory_usage: 72%
active_connections: 340
```

Tipos comuns:
- **Contador (Counter)** — só sobe (total de requisições, total de erros)
- **Medidor (Gauge)** — sobe e desce (memória, conexões ativas)
- **Histograma (Histogram)** — distribuição de valores (latência, tamanho de payload)

### 3. Traces — Caminho de uma request

Rastreia uma request específica por todos os serviços que ela passou.

```
POST /transfer  (total: 120ms)
  │
  ├── Auth middleware        (5ms)
  ├── AccountService.withdraw (8ms)
  ├── DB query               (95ms)  ← gargalo aqui
  └── Response serialization (2ms)
```

Útil para identificar onde está o tempo sendo gasto em sistemas distribuídos.

---

## Ferramentas

### Datadog
Plataforma completa (logs + métricas + traces). Instala um agente no servidor que coleta tudo automaticamente. SaaS — você acessa o dashboard na nuvem deles.

```
Servidor → Datadog Agent → Datadog Cloud → Dashboard
```

- Muito usado em fintechs e empresas grandes
- Tem APM (Monitoramento de Performance de Aplicações) nativo
- Caro — cobrado por host + volume de dados
- Tem alertas, detecção de anomalias, dashboards customizáveis

### New Relic
Concorrente do Datadog. Modelo similar — agente + SaaS. Historicamente mais focado em APM.

### Prometheus + Grafana
Versão open source. Prometheus coleta e armazena métricas, Grafana exibe os dashboards.

```
Aplicação → expõe /metrics → Prometheus faz scraping → Grafana exibe
```

- Gratuito e self-hosted
- Muito usado com Kubernetes
- Você opera a infraestrutura
- Usado no goCache (estava no README)

### Sentry
Focado em error tracking — captura exceções em produção com stack trace completo.

```php
// PHP
\Sentry\init(['dsn' => 'https://...']);
\Sentry\captureException($exception);
```

Complementa Datadog/New Relic — eles medem performance, Sentry captura erros.

### ELK Stack (Elasticsearch + Logstash + Kibana)
Stack open source para logs. Logstash coleta, Elasticsearch armazena e indexa, Kibana exibe.

---

## Comparativo

| | Datadog | New Relic | Prometheus+Grafana |
|---|---|---|---|
| Tipo | SaaS | SaaS | Self-hosted |
| Custo | Alto | Alto | Gratuito |
| Setup | Simples (agente) | Simples (agente) | Complexo |
| Logs | ✅ | ✅ | ❌ (só métricas) |
| Métricas | ✅ | ✅ | ✅ |
| Traces | ✅ | ✅ | Com Jaeger/Tempo |
| Uso | Empresas grandes | Empresas grandes | Startups, Kubernetes |

---

## Em Sistemas Financeiros

Observabilidade é crítica em fintechs — você precisa saber em tempo real se:

- Taxa de erro subiu (pagamentos falhando?)
- Latência aumentou (processador externo lento?)
- Volume de transações caiu (algum método parou de funcionar?)
- Alguma conta está sendo usada de forma suspeita (fraude?)

### Métricas essenciais em pagamentos

```
payment_success_rate          → % de pagamentos aprovados
payment_processing_time_p99   → tempo do pior 1% das transações
chargeback_rate               → % de chargebacks
duplicate_payment_attempts    → tentativas de duplicata detectadas
external_api_error_rate       → erros em APIs de adquirentes/bancos
```

### Alertas

Você configura limites — se a métrica X ultrapassar o limite Y, dispara um alerta:

```
error_rate > 1%        → PagerDuty / Slack → time de plantão
p99_latency > 500ms    → alerta warning
p99_latency > 2000ms   → alerta crítico
```

---

## Como Responder na Entrevista

> "Para observabilidade usaria os três pilares: logs estruturados em JSON para rastrear erros individuais, métricas com Prometheus ou Datadog para monitorar taxa de sucesso, latência e volume de transações em tempo real, e traces para identificar gargalos em operações lentas. Alertas configurados para erro rate e latência P99 com notificação imediata para o time de plantão."

---

## Related

- [[Finance/FinanceEng]]
- [[DistributedSystems/Performance]]
- [[Finance/Finance]]

#### My commentaries
-
