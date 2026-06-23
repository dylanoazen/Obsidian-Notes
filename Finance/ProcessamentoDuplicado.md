# Processamento Duplicado

Como detectar e prevenir que a mesma operação financeira seja processada mais de uma vez.

Related: [[DistributedSystems/Idempotency]], [[Finance/CoreBanking]], [[DistributedSystems/Concurrency]]

---

## O Problema

Em sistemas distribuídos, retries são inevitáveis. Rede cai, timeout acontece, cliente reenvia. O resultado: a mesma operação chega duas vezes.

```
Cliente → POST /pagamento → Servidor processa → R$ 100 debitados
              │
              └── timeout (cliente não recebeu resposta)
                      │
                      ▼
Cliente → POST /pagamento (retry) → Servidor processa DE NOVO → R$ 200 debitados ❌
```

Em e-commerce: pedido duplicado.
Em pagamentos: cobrança dupla.
Em transferências: dinheiro enviado duas vezes.

---

## Causas Comuns

- **Timeout de rede** — cliente não recebeu resposta, reenvio automático
- **Retry automático** — bibliotecas HTTP com retry configurado
- **Duplo clique** — usuário clicou duas vezes no botão de pagar
- **Falha parcial** — servidor processou mas caiu antes de responder
- **Mensageria** — consumidor processou a mensagem mas falhou antes de confirmar (entrega garantida ao menos uma vez)

---

## Solução Principal: Idempotency Key

Cada operação recebe um identificador único gerado pelo cliente. O servidor usa esse ID para detectar duplicatas.

→ Detalhe completo: [[DistributedSystems/Idempotency]]

```
POST /pagamento
Idempotency-Key: pay_abc123  ← gerado pelo cliente, único por operação

{
  "amount": 100,
  "account": "123"
}
```

Servidor:
1. Recebe `pay_abc123`
2. Já existe no banco? → retorna resultado anterior
3. Não existe? → processa, salva resultado com a key

Mesmo que cheguem 10 retries idênticos, só o primeiro processa.

---

## Solução Complementar: Deduplicação por Conteúdo

Quando não há idempotency key, deduplica por hash do conteúdo + janela de tempo:

```php
$hash = hash('sha256', json_encode([
    'account' => $accountId,
    'amount'  => $amount,
    'type'    => $type,
]));

// verifica se processou esse hash nos últimos 60 segundos
if ($cache->get("dedup:{$hash}")) {
    return $previousResult;
}

$cache->set("dedup:{$hash}", $result, ttl: 60);
```

Menos confiável — dois pagamentos idênticos legítimos em sequência seriam bloqueados. Idempotency Key é sempre preferível.

---

## Race Condition na Deduplicação

E se dois requests idênticos chegam ao mesmo tempo? Ambos checam "já existe?" simultaneamente, ambos respondem "não", ambos processam.

```
Request A: busca key → não existe
Request B: busca key → não existe   ← chegou antes de A salvar
Request A: processa → salva
Request B: processa → salva         ← duplicata!
```

Solução: **lock distribuído** antes de checar.

```php
$lock = $redis->set("lock:pay_{$key}", 1, ['NX', 'EX' => 30]);
if (!$lock) {
    // outro processo está processando — aguarda ou retorna 409
    return new Response(409, 'Processing');
}

try {
    // verifica e processa com segurança
} finally {
    $redis->del("lock:pay_{$key}");
}
```

→ [[DistributedSystems/Concurrency]]

---

## Em Mensageria (Kafka / RabbitMQ)

Brokers garantem **entrega ao menos uma vez** — a mesma mensagem pode chegar mais de uma vez se o consumidor falhar antes de confirmar o recebimento.

```
Consumidor processa mensagem → falha antes de confirmar
Broker reenvia a mesma mensagem → consumidor processa de novo ← duplicata
```

Solução: consumidor idempotente — verifica se já processou aquele `message_id` antes de processar.

---

## Related

- [[DistributedSystems/Idempotency]]
- [[Finance/CoreBanking]]
- [[Finance/FinanceEng]]
- [[DistributedSystems/Concurrency]]
- [[DistributedSystems/TransactionsPipeline]]

#### My commentaries
-
