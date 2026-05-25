

# Protocols

A protocol is a set of rules that defines how to request data, and how it is transmitted and received over a network. Without protocols, the receiver wouldn't know where the message ends or how to interpret it.

## TCP vs UDP

| | [[TCP]] | [[UDP]] |
|---|---|---|
| Connection | Fixed between two sides | No fixed connection |
| Delivery | Guaranteed | Not guaranteed |
| Order | Guaranteed | Not guaranteed |
| Speed | Slower | Faster |
| Use case | Chat, API, file transfer | Games, streaming, video calls |

**TCP** is like a phone call — the line stays open between both sides until someone hangs up. Everything sent goes directly to the other side in order.

**UDP** is like sending a letter — each packet is independent, can take different routes, and there's no guarantee it arrives or arrives in order.

### When to use each:
- **TCP** → when order and delivery matter
- **UDP** → when speed matters more than perfection

## TCP

TCP is a byte stream — it does not preserve message boundaries:
- One message can arrive fragmented
- Two messages can arrive at the same time (coalesced)

That's why you need framing.

## Framing

Framing is a technique to mark where a message starts and where it ends. See [[Framing]] for the full breakdown.

### Framing Types

