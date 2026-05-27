# Pub/Sub

Pub/Sub (Publish/Subscribe) is a messaging pattern where senders (**publishers**) do not send messages directly to receivers. Instead, they publish messages to a **channel**, and any interested party (**subscriber**) receives them.

---

## Core Terms

- **Publisher:** the one who sends a message to a channel.
- **Subscriber:** the one who listens to a channel and receives messages.
- **Channel:** the named pipe through which messages flow.
- **Message:** the data published to a channel.
- **Fanout:** when one message is delivered to multiple subscribers at once.
- **Broker:** the middleman that manages channels and routes messages (e.g. Redis, Kafka).

---

## How it Works

```
Publisher → [channel: "orders"] → Subscriber A
                                → Subscriber B
                                → Subscriber C
```

1. Subscriber A, B, and C subscribe to the channel `"orders"`
2. Publisher sends a message to `"orders"`
3. All three subscribers receive the message simultaneously

The publisher does not know who is listening — it just publishes.
The subscribers do not know who is publishing — they just listen.

---

## Fanout

Fanout is when **one message reaches multiple subscribers** at the same time.

This is the default behavior in Pub/Sub — every subscriber on the channel gets a copy of the message.

```
1 message published → N subscribers receive it
```

Fanout is useful when multiple parts of your system need to react to the same event independently.

---

## Pub/Sub vs Direct Messaging

| | **Direct (REST/queue)** | **Pub/Sub** |
|---|---|---|
| Sender knows receiver? | Yes | No |
| One message → many receivers? | No | Yes |
| Coupling | Tight | Loose |
| Example | HTTP request | Event notification |

---

## Practical Examples

- **Order placed** → notify inventory service, billing service, and email service at the same time
- **User signed up** → trigger welcome email, analytics event, and onboarding flow independently
- **Price updated** → all connected clients receive the new price in real time

---

## Pub/Sub in Redis

Redis has native Pub/Sub support:

```
# Subscriber listens to a channel
SUBSCRIBE orders

# Publisher sends a message
PUBLISH orders "new order: #123"
```

Redis delivers the message to all active subscribers instantly.

**Important:** Redis Pub/Sub is **fire and forget** — if a subscriber is offline when the message is published, it misses it. There is no persistence, no backlog.

## How Applications Read from Redis

There are two different models — do not confuse them:

**Normal read (cache/GET)** — the application actively queries Redis:

```
[Backend]  →  GET user:123
[Redis]    →  returns the value
```

The application asks, Redis answers. The application controls when to fetch.

**Pub/Sub** — the application stays connected and Redis pushes messages to it:

```
[Backend]  →  SUBSCRIBE orders
              (stays connected, waiting)

[Redis]    →  message arrives → pushes to backend
```

The application does not ask repeatedly — it just listens and reacts when something arrives.

| | **Normal read** | **Pub/Sub** |
|---|---|---|
| Who initiates | Application | Redis |
| Model | Pull | Push |
| When to use | Fetch a specific value | React to events in real time |

> In cache you go fetch the data. In Pub/Sub the data comes to you when it happens.

## Pub/Sub vs Replication

These are completely separate mechanisms in Redis — a common source of confusion:

| | **Replication** | **Pub/Sub** |
|---|---|---|
| Purpose | Sync data between Redis nodes | Dispatch messages between your services |
| Who uses it | Redis internally | Your application |
| What travels | Write commands | Custom messages |
| Subscriber is | A Redis replica | Your backend/service |

Replication is internal to Redis — the primary streams commands to replicas automatically.
Pub/Sub is for your application — your services publish and subscribe to communicate with each other.

```
[Order Service]  →  PUBLISH orders "new order #123"
                           ↓
                     Redis (broker)
                           ↓
          ┌────────────────┼────────────────┐
   [Inventory]         [Billing]          [Email]
```

Redis acts as the **messenger** between your services — not between its own replicas.

---

## Fire and Forget — The Main Limitation

Unlike replication (which has a backlog and can catch up), Pub/Sub in Redis does not store messages.

- Subscriber online → receives the message ✓
- Subscriber offline → message is lost ✗

If you need guaranteed delivery, you need a different tool (e.g. Redis Streams, Kafka).

---

## Design Notes

- Pub/Sub decouples producers from consumers — they do not need to know about each other
- Great for real-time notifications, live updates, and event broadcasting
- Not suitable when message delivery must be guaranteed
- Channel names are just strings — no schema, no enforced structure

---

## Related Notes

- [[Replication]]
- [[Persistence]]
