# Transactions & Pipeline

Dois mecanismos que controlam como múltiplos comandos são enviados e executados — mas resolvem problemas diferentes.

---

## O Problema que Resolvem

Por padrão, todo comando Redis é independente:

```
SET balance 100   ← executes now
SET balance 90    ← executes now, separately
```

E se você precisar que múltiplos comandos:
- Executem juntos como uma única unidade? → **Transaction**
- Sejam enviados de uma vez sem esperar? → **Pipeline**

### A Analogia do Restaurante

**Pipeline** — você anota 10 pedidos em um papel e entrega tudo de uma vez ao garçom, em vez de chamá-lo 10 vezes separadas. Você economizou 9 idas e vindas.

**Transaction** — o garçom só traz a comida quando a bebida também estiver pronta. Sem bebida, sem comida. Ou tudo chega junto ou nada chega.

**Ambos juntos** — você entrega ao garçom uma lista de pedidos *(pipeline)* e diz "só traga quando tudo estiver pronto" *(transaction)*.

> Pipeline resolve a **entrega**. Transaction resolve a **regra de execução**.

---

## Transactions

Uma transaction agrupa múltiplos comandos para que executem **atomicamente** — tudo ou nada, sem que nenhum outro cliente interrompa no meio.

```
MULTI         ← start transaction
SET balance 90
SET log "deducted 10"
EXEC          ← execute all at once
```

### Atomicidade

Atomicidade significa: **ou todos os comandos rodam, ou nenhum roda.**

Se o servidor travar no meio de uma transaction, nenhum dos comandos tem efeito. Sem estado parcial.

**Analogia:** uma transferência bancária — debitar uma conta e creditar outra. Se apenas o débito rodar e o crédito falhar, o dinheiro some. Ambos devem ter sucesso ou nenhum deve acontecer.

### O que as transactions do Redis garantem

- Comandos entre `MULTI` e `EXEC` são enfileirados, não executados imediatamente
- No `EXEC`, todos os comandos enfileirados rodam sequencialmente sem interrupção de outros clientes
- Nenhum outro cliente consegue inserir um comando entre eles

### O que as transactions do Redis NÃO garantem

- Se um comando dentro da transaction falhar (ex.: tipo errado), os outros ainda rodam — Redis não faz rollback
- Isso é diferente das transactions SQL que fazem rollback de tudo em caso de falha

---

## Pipeline

Um pipeline trata de **eficiência de rede** — enviar múltiplos comandos ao Redis em uma única ida e volta em vez de um por um.

```
Sem pipeline:
[Client] → SET a 1 → [Redis] → OK → [Client] → SET b 2 → [Redis] → OK

Com pipeline:
[Client] → SET a 1 / SET b 2 / SET c 3 → [Redis] → OK / OK / OK
```

### O problema da ida e volta

Todo comando enviado individualmente paga um custo de ida e volta:
- Cliente envia o comando
- Espera o Redis responder
- Então envia o próximo comando

Com 100 comandos, você paga 100 idas e voltas. Com um pipeline, você paga 1.

```
Sem pipeline:
GET user:1  → wait for response
GET user:2  → wait for response
GET user:3  → wait for response
(3 idas e voltas)

Com pipeline:
GET user:1
GET user:2  → envia tudo de uma vez, recebe tudo de uma vez
GET user:3
(1 ida e volta)
```

Redis é key-value — você já sabe exatamente qual chave quer antes de perguntar. Pipeline apenas permite que você pergunte por muitas chaves de uma vez sem esperar entre cada uma.

| | **SQL** | **Redis + Pipeline** |
|---|---|---|
| Você diz | "me dê todo mundo com idade > 18" | "me dê as chaves user:1, user:2, user:3" |
| Você precisa saber | Apenas a condição | As chaves exatas de antemão |
| Pipeline ajuda | Não se aplica | Envia múltiplas buscas em uma ida e volta |

### Batching

Batching é o ato de agrupar múltiplos comandos para enviar juntos. Pipeline é como você implementa batching no Redis.

### O que pipeline NÃO garante

- Comandos em um pipeline **não são atômicos** — outros clientes podem executar comandos entre eles
- Se a conexão cair no meio do pipeline, alguns comandos podem ter rodado e outros não

---

## Transactions vs Pipeline

| | **Transaction** | **Pipeline** |
|---|---|---|
| Propósito | Atomicidade | Eficiência de rede |
| Atômico? | Sim | Não |
| Reduz idas e voltas? | Não | Sim |
| Comandos enfileirados? | No servidor | No cliente |
| Outros clientes bloqueados? | Sim (durante EXEC) | Não |

São complementares — você pode usar ambos ao mesmo tempo: enviar uma transaction por um pipeline.

---

## Usando Ambos Juntos

```
[Client pipelines these commands in one round trip]

MULTI
SET balance 90
SET log "deducted 10"
EXEC
```

- Pipeline cuida da **rede** — uma ida e volta
- Transaction cuida da **atomicidade** — sem interrupções

---

## Exemplos Práticos

**Transaction:** deduzir saldo e registrar o log — ambos devem acontecer ou nenhum
**Pipeline:** carregar um perfil de usuário que requer 5 comandos GET — envia todos os 5 de uma vez, recebe todas as 5 respostas de uma vez

---

## Notas de Design

- Redis é single-threaded para execução de comandos — transactions funcionam porque nada mais roda entre MULTI e EXEC
- Pipeline é uma otimização do lado do cliente — o cliente armazena os comandos em buffer e os envia juntos
- Para comportamento de rollback verdadeiro (como SQL), Redis não é a ferramenta certa
- O comando WATCH pode ser usado antes do MULTI para abortar uma transaction se uma chave foi modificada por outro cliente (locking otimista)

---

## Notas Relacionadas

- [[Replication]]
- [[PubSub]]
- [[Persistence]]
