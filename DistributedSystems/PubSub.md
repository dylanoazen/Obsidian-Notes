# Pub/Sub

Pub/Sub (Publish/Subscribe) é um padrão de mensageria onde os remetentes (**publishers**) não enviam mensagens diretamente para os receptores. Em vez disso, publicam mensagens em um **channel**, e qualquer parte interessada (**subscriber**) as recebe.

---

## Termos Principais

- **Publisher:** quem envia uma mensagem para um channel.
- **Subscriber:** quem escuta um channel e recebe mensagens.
- **Channel:** o canal nomeado pelo qual as mensagens fluem.
- **Message:** os dados publicados em um channel.
- **Fanout:** quando uma mensagem é entregue a múltiplos subscribers de uma vez.
- **Broker:** o intermediário que gerencia channels e roteia mensagens (ex.: Redis, Kafka).

---

## Como Funciona

```
Publisher → [channel: "orders"] → Subscriber A
                                → Subscriber B
                                → Subscriber C
```

1. Subscriber A, B e C se inscrevem no channel `"orders"`
2. Publisher envia uma mensagem para `"orders"`
3. Os três subscribers recebem a mensagem simultaneamente

O publisher não sabe quem está escutando — apenas publica.
Os subscribers não sabem quem está publicando — apenas escutam.

---

## Fanout

Fanout é quando **uma mensagem chega a múltiplos subscribers** ao mesmo tempo.

Esse é o comportamento padrão no Pub/Sub — todo subscriber do channel recebe uma cópia da mensagem.

```
1 mensagem publicada → N subscribers a recebem
```

Fanout é útil quando múltiplas partes do seu sistema precisam reagir ao mesmo evento de forma independente.

---

## Pub/Sub vs Mensageria Direta

| | **Direta (REST/queue)** | **Pub/Sub** |
|---|---|---|
| Remetente conhece o receptor? | Sim | Não |
| Uma mensagem → muitos receptores? | Não | Sim |
| Acoplamento | Forte | Fraco |
| Exemplo | Requisição HTTP | Notificação de evento |

---

## Exemplos Práticos

- **Pedido realizado** → notifica o serviço de estoque, serviço de cobrança e serviço de e-mail ao mesmo tempo
- **Usuário cadastrado** → dispara e-mail de boas-vindas, evento de analytics e fluxo de onboarding de forma independente
- **Preço atualizado** → todos os clientes conectados recebem o novo preço em tempo real

---

## Pub/Sub no Redis

Redis tem suporte nativo a Pub/Sub:

```
# Subscriber listens to a channel
SUBSCRIBE orders

# Publisher sends a message
PUBLISH orders "new order: #123"
```

Redis entrega a mensagem a todos os subscribers ativos instantaneamente.

**Importante:** Pub/Sub no Redis é **fire and forget** — se um subscriber estiver offline quando a mensagem for publicada, ele a perde. Não há persistência, nem backlog.

## Como Aplicações Leem do Redis

Existem dois modelos diferentes — não confunda:

**Leitura normal (cache/GET)** — a aplicação consulta o Redis ativamente:

```
[Backend]  →  GET user:123
[Redis]    →  returns the value
```

A aplicação pergunta, o Redis responde. A aplicação controla quando buscar.

**Pub/Sub** — a aplicação permanece conectada e o Redis empurra mensagens para ela:

```
[Backend]  →  SUBSCRIBE orders
              (stays connected, waiting)

[Redis]    →  message arrives → pushes to backend
```

A aplicação não pergunta repetidamente — apenas escuta e reage quando algo chega.

| | **Leitura normal** | **Pub/Sub** |
|---|---|---|
| Quem inicia | Aplicação | Redis |
| Modelo | Pull | Push |
| Quando usar | Buscar um valor específico | Reagir a eventos em tempo real |

> No cache você vai buscar os dados. No Pub/Sub os dados vêm até você quando acontecem.

## Pub/Sub vs Replication

Esses são mecanismos completamente separados no Redis — uma fonte comum de confusão:

| | **Replication** | **Pub/Sub** |
|---|---|---|
| Propósito | Sincronizar dados entre nós Redis | Despachar mensagens entre seus serviços |
| Quem usa | Redis internamente | Sua aplicação |
| O que trafega | Comandos de escrita | Mensagens customizadas |
| O subscriber é | Uma réplica Redis | Seu backend/serviço |

Replication é interno ao Redis — o primary transmite comandos para as réplicas automaticamente.
Pub/Sub é para sua aplicação — seus serviços publicam e se inscrevem para se comunicar entre si.

```
[Order Service]  →  PUBLISH orders "new order #123"
                           ↓
                     Redis (broker)
                           ↓
          ┌────────────────┼────────────────┐
   [Inventory]         [Billing]          [Email]
```

Redis age como o **mensageiro** entre seus serviços — não entre suas próprias réplicas.

---

## Fire and Forget — A Limitação Principal

Ao contrário da replication (que tem um backlog e consegue se recuperar), Pub/Sub no Redis não armazena mensagens.

- Subscriber online → recebe a mensagem ✓
- Subscriber offline → mensagem é perdida ✗

Se você precisa de entrega garantida, precisa de uma ferramenta diferente (ex.: Redis Streams, Kafka).

---

## Notas de Design

- Pub/Sub desacopla produtores de consumidores — eles não precisam se conhecer
- Excelente para notificações em tempo real, atualizações ao vivo e broadcasting de eventos
- Não é adequado quando a entrega da mensagem precisa ser garantida
- Nomes de channels são apenas strings — sem schema, sem estrutura imposta

---

## Notas Relacionadas

- [[Replication]]
- [[Persistence]]
