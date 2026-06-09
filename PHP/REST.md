# REST e HTTP Semantics

O que significa ser RESTful, status codes, idempotência e convenção sobre configuração.

Related: [[PHP/Idempotency]]

---

## REST — O que é de Verdade

REST (Representational State Transfer) não é só "usar JSON e HTTP". É um conjunto de constraints:

- **Client-Server**: separação de responsabilidades
- **Stateless**: cada request contém toda info necessária (sem sessão no servidor)
- **Cacheable**: responses indicam se podem ser cacheadas
- **Uniform Interface**: recursos identificados por URLs, manipulados por representações
- **Layered System**: cliente não sabe se fala com server final ou proxy

## Métodos HTTP — Semântica

| Método | Semântica | Safe? | Idempotente? |
|--------|----------|-------|-------------|
| GET | Ler recurso | ✅ | ✅ |
| POST | Criar recurso | ❌ | ❌ |
| PUT | Substituir recurso inteiro | ❌ | ✅ |
| PATCH | Atualizar parcialmente | ❌ | ❌* |
| DELETE | Remover recurso | ❌ | ✅ |

- **Safe**: não causa efeito colateral (GET nunca muda dados)
- **Idempotente**: fazer 1x ou 10x tem o mesmo resultado

> GET NUNCA deve ter efeito colateral. Se perguntarem "por que GET não pode criar algo?", é porque clientes, proxies e crawlers assumem que GET é seguro e podem refazer a qualquer momento.

## Status Codes que Importam

```
2xx — Sucesso
  200 OK              → request processado com sucesso
  201 Created         → recurso criado (POST)
  204 No Content      → sucesso, sem body (DELETE)

4xx — Erro do cliente
  400 Bad Request     → body malformado, JSON inválido
  401 Unauthorized    → não autenticado
  403 Forbidden       → autenticado, mas sem permissão
  404 Not Found       → recurso não existe
  409 Conflict        → conflito de estado (ex: conta já existe)
  422 Unprocessable   → body válido, mas regra de negócio falhou

5xx — Erro do servidor
  500 Internal Error  → bug no servidor
  502 Bad Gateway     → upstream server falhou
  503 Service Unavail → servidor sobrecarregado/manutenção
```

### 400 vs 422

```
400: { "amount": "abc" }         → nem é número, request malformado
422: { "amount": -50 }           → é número, mas regra proíbe negativo
422: transfer com saldo insuficiente → dados válidos, negócio rejeita
```

## Convenção sobre Configuração

Princípio que EBANX valoriza: seguir padrões previsíveis em vez de inventar.

```
✅ Convenção:
  GET    /accounts          → listar
  GET    /accounts/123      → detalhar
  POST   /accounts          → criar
  PUT    /accounts/123      → atualizar
  DELETE /accounts/123      → remover

❌ Invenção:
  POST   /getAccounts       → ???
  GET    /account/create    → GET criando?
  POST   /doTransfer        → verbo na URL
```

- URLs são **substantivos** (recursos), não verbos
- O método HTTP já é o verbo
- Plural para coleções (`/accounts`, não `/account`)
- IDs na URL, não no body para GET/DELETE

## Response Design

```json
// ✅ Consistente
{
    "data": {
        "id": "abc-123",
        "balance": 1000,
        "currency": "BRL"
    }
}

// ✅ Erro estruturado
{
    "error": {
        "code": "INSUFFICIENT_FUNDS",
        "message": "Account abc-123 has insufficient funds"
    }
}
```

## Related

- [[PHP/Idempotency]]
- [[PHP/PHP]]
- [[Network/Protocols]]

#### My commentaries
- 
