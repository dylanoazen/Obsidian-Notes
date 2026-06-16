# Idempotência

O tema nº 1 em entrevistas de fintech — como evitar processar a mesma operação duas vezes.

Related: [[SoftwareEngineering/REST]], [[PHP/Concurrency]]

---

## O Problema

```
Cliente ──► POST /transfer ──► Servidor (processa)
                                    │
Cliente ◄── timeout (não recebeu response) ◄──┘
    │
    └── Reenvia o mesmo POST /transfer
              │
              ▼
         Processado DE NOVO? Dinheiro debitado duas vezes!
```

Em pagamentos, isso é catastrófico. O cliente não sabe se:
- O servidor nunca recebeu o request
- O servidor processou mas a response se perdeu
- O servidor está processando ainda

## Idempotency Key

Solução: o cliente manda um **identificador único** por operação.

```
POST /transfer
Idempotency-Key: txn_abc123def456
{
    "from": "account_A",
    "to": "account_B", 
    "amount": 5000
}
```

O servidor:
1. Recebe o request com a key
2. Verifica se já processou essa key antes
3. Se **sim** → retorna o resultado anterior (sem reprocessar)
4. Se **não** → processa, salva o resultado associado à key, retorna

```php
class TransferService {
    public function transfer(string $idempotencyKey, TransferRequest $dto): TransferResult {
        // 1. Já processou?
        $existing = $this->idempotencyStore->find($idempotencyKey);
        if ($existing !== null) {
            return $existing; // retorna resultado anterior
        }

        // 2. Processa normalmente
        $result = $this->executeTransfer($dto);

        // 3. Salva resultado com a key
        $this->idempotencyStore->save($idempotencyKey, $result);

        return $result;
    }
}
```

## Fluxo Completo

```
Request 1 (key=abc):
  ├── Busca key=abc → não existe
  ├── Processa transfer → sucesso
  ├── Salva (key=abc → {status: success, ...})
  └── Retorna 200 {status: success}

Request 2 (key=abc, retry):
  ├── Busca key=abc → EXISTE
  ├── Não reprocessa
  └── Retorna 200 {status: success}  ← mesmo resultado
```

## Considerações Importantes

### Escopo da Key

- Key deve ser **por operação**, não por usuário
- Gerada pelo **cliente** (UUID v4 é comum)
- Servidor nunca gera a key (senão retry não funciona)

### TTL

- Keys devem expirar (24h é comum)
- Senão o storage cresce indefinidamente

### Concorrência na Key

Se dois requests com a mesma key chegam simultaneamente:

```php
// Use lock pra evitar processamento duplicado
$lock = $this->lockService->acquire("idempotency:{$key}");
if (!$lock) {
    return new Response(409, 'Request already being processed');
}
try {
    // processa...
} finally {
    $lock->release();
}
```

### O que NÃO é Idempotente

- `POST /transfer` sem idempotency key → cada call pode criar nova transfer
- `GET /balance` → naturalmente idempotente (só lê)
- `PUT /account/123` → naturalmente idempotente (substitui pro mesmo valor)

## Na Entrevista

> "Se o cliente reenvia o mesmo depósito por timeout de rede, uso uma idempotency key — um identificador único por operação que o servidor guarda e usa pra retornar o mesmo resultado em vez de reprocessar. É o padrão mais importante em sistemas de pagamento."

## Como Stripe Faz

```bash
curl https://api.stripe.com/v1/charges \
  -H "Idempotency-Key: txn_abc123" \
  -d amount=2000 \
  -d currency=brl
```

Stripe guarda a key por 24h e retorna o mesmo resultado em retries.

## Related

- [[SoftwareEngineering/REST]]
- [[PHP/Concurrency]]
- [[PHP/Persistence]]
- [[Security/IAM]]

## Resources

- https://stripe.com/docs/api/idempotent_requests
- https://brandur.org/idempotency-keys

#### My commentaries
- 
