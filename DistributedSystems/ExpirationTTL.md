# Expiration / TTL

Expiration (TTL) é a capacidade de definir um limite de tempo para uma chave. Após o TTL, a chave é removida automaticamente.

---

## Conceitos Principais

- **TTL (Time To Live):** por quanto tempo uma chave deve existir.
- **Expire time:** timestamp absoluto em que a chave se torna inválida.
- **Expired key:** uma chave cujo TTL já passou.

---

## Exemplo (TTL de 10s, leitura em 12s)

- A chave é definida com TTL = 10 segundos.
- Se o cliente ler a chave após 12 segundos, ela já está expirada.
- O servidor a deleta e retorna "ausente".

---

## Lazy Expiration

**Lazy expire** significa que as chaves são removidas apenas quando são acessadas.

**Como funciona:**
- Cliente requisita uma chave
- Servidor verifica se o TTL expirou
- Se expirou, deleta e retorna ausente

**Prós:**
- Muito barato e simples
- Sem trabalho em background

**Contras:**
- Chaves expiradas podem permanecer na memória se nunca forem acessadas

---

## Active Expiration

**Active expire** significa que o servidor periodicamente varre as chaves e remove as expiradas.

**Como funciona:**
- Timer roda a cada N milissegundos
- Uma amostra aleatória de chaves é verificada
- Chaves expiradas são deletadas

**Prós:**
- Libera memória mesmo que as chaves nunca sejam acessadas

**Contras:**
- Custo em background (CPU)
- Precisa de ajuste fino para evitar picos de latência

---

## Abordagem Híbrida

Uma estratégia comum é usar **ambas**:
- Lazy expire no acesso
- Active expire em background

Isso equilibra corretude e performance.

---

## Eviction Policies

Eviction acontece **quando a memória está cheia**, não apenas quando as chaves expiram.

Políticas comuns:

- **noeviction**
  - Rejeita novas escritas quando a memória está cheia

- **allkeys-lru**
  - Remove a chave menos recentemente usada (qualquer chave)

- **allkeys-lfu**
  - Remove a chave menos frequentemente usada (qualquer chave)

- **allkeys-random**
  - Remove uma chave aleatória

- **volatile-lru**
  - Remove a chave menos recentemente usada **com TTL**

- **volatile-lfu**
  - Remove a chave menos frequentemente usada **com TTL**

- **volatile-random**
  - Remove uma chave aleatória **com TTL**

- **volatile-ttl**
  - Remove a chave com o **menor TTL restante**

---

## Modelo de Implementação (Conceito)

Você pode armazenar TTLs em um mapa separado:

```
store:   map[string]string
expires: map[string]time.Time
```

Verificação na leitura:
- Se a chave tem TTL e ele expirou -> deleta e retorna ausente

Limpeza em background:
- Em um timer, varre uma amostra de chaves e remove as expiradas

---

## Notas Rápidas

- TTL não é uma promessa de tempo exato, apenas um limite superior
- Lazy expire garante corretude no acesso
- Active expire garante que a memória não encha com chaves mortas

---

## Termos para Lembrar

- TTL
- Expire time
- Lazy expiration
- Active expiration
- Eviction policy
- LRU / LFU
