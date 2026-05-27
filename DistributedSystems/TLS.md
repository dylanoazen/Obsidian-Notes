# TLS (Transport Layer Security)

TLS is the protocol that creates an **encrypted tunnel** between two parties over a network. Nobody in the middle can read or alter what travels through it.

It runs on top of TCP — TCP opens the connection, TLS secures it.

---

## Three Goals

1. **Identity** — verify the server is who it claims to be (certificate)
2. **Key exchange** — agree on a secret key without ever sending it over the network
3. **Encryption** — encrypt everything from that point on

---

## The TLS Handshake Step by Step

### Step 1 — ClientHello

You send to the server:
> "I want to communicate encrypted. I support these algorithms. Here is a random number from me."

The random number will be used later to generate the session key.

### Step 2 — ServerHello + Certificate

The server responds:
> "Ok, let's use this algorithm. Here is my certificate. Here is my random number."

The certificate is the server's ID card — signed by a CA (Certificate Authority) that your browser already trusts.

### Step 3 — You Verify the Certificate

Your browser checks:
> "Was this certificate signed by someone I trust?"

- Yes → continue
- No → red screen: "Your connection is not private"

### Step 4 — Key Exchange (Diffie-Hellman)

This is the elegant part. Both sides need to arrive at the same secret key **without ever sending it over the network**.

**Paint analogy:**

```
Both agree on a public base color: yellow
                👀 anyone can see this

You pick a secret color: blue       (never sent over network)
Server picks a secret color: red    (never sent over network)

You send:    yellow + blue   = green
Server sends: yellow + red   = orange
                👀 anyone can see green and orange

You take orange and mix with your secret blue   = brown
Server takes green and mix with their secret red = brown

Both arrived at brown — without ever sending their secret colors.
```

**Numerical analogy:**

```
Public base: 2

You choose secret:    3    (never sent)
Server chooses secret: 1   (never sent)

You send:    2 + 3 = 5  →
             ← 2 + 1 = 3  :Server sends

You:    3 + your secret 3 = 6... wait no

Actually:
You send server: base + your secret
Server sends you: base + their secret
Both add the other's value to their own secret → same result
```

In cryptography, the math makes it impossible to reverse-engineer the secret from what was sent publicly. An attacker sees the public values but cannot derive the secret.

### Step 5 — Everything Encrypted

Both sides now share the same secret key. All communication from here is encrypted with it using **symmetric encryption** — fast and lightweight.

---

## Symmetric vs Asymmetric in TLS

### Symmetric Encryption

**One key only** — whoever encrypts and whoever decrypts use the same key.

```
You      →  encrypt with key 🔑  →  "█▓▒░"
Server   →  decrypt with key 🔑  →  "hello"
```

**Problem:** how do you agree on the key with the server without sending it over the network? If someone intercepts it, it's over.

**Advantage:** very fast — great for encrypting large volumes of data.

### Asymmetric Encryption

**Two keys** — one public, one private. What one encrypts, only the other can decrypt.

```
Public key  🔓  → you share with everyone
Private key 🔑  → stays only with you, never leaves
```

Anyone can encrypt a message with your public key. Only you can decrypt it with your private key.

**Advantage:** no need to agree on anything upfront — the public key can be distributed openly.

**Problem:** much slower than symmetric.

### Why TLS Uses Both

TLS combines the best of both:

```
1. Handshake  →  uses asymmetric
               → solves "how do we agree on a key without sending it over the network?"
               → Diffie-Hellman does this with public/private keys

2. Session    →  uses symmetric
               → now that both sides have the same secret key, use it for everything
               → much faster for encrypting actual data
```

**Analogy:** you use secure mail *(asymmetric)* to agree on a secret code with someone. Once both have the code, you communicate by radio using that code *(symmetric)* — much faster.

| | **Asymmetric** | **Symmetric** |
|---|---|---|
| When | Key exchange (handshake) | Data encryption (session) |
| How | Public/private key pair | One shared key |
| Why | Secure for exchanging secrets | Much faster |

> Asymmetric solves the **key agreement problem** safely.
> Symmetric solves the **performance problem** during data exchange.
> TLS uses asymmetric to bootstrap, symmetric to run.

---

## The Full Handshake Flow

```
[TCP connection already open]

You        →   ClientHello (algorithms, random number)
           ←   ServerHello + Certificate + random number
Verify certificate ✓
           →   Key exchange (Diffie-Hellman)
           ←   Key exchange (Diffie-Hellman)
Both derive the same secret key independently
           →   Finished (encrypted)
           ←   Finished (encrypted)

[Encrypted tunnel established — HTTP starts flowing]
```

---

## HTTPS = HTTP + TLS

When you access `https://`, that is exactly it — HTTP running inside a TLS tunnel.

```
Layers:
┌─────────────┐
│    HTTP     │  ← your request
├─────────────┤
│    TLS      │  ← encryption
├─────────────┤
│    TCP      │  ← moves bytes, does not know what they are
├─────────────┤
│     IP      │  ← routing
└─────────────┘
```

TCP does not know it is carrying TLS — to TCP it is just bytes.

---

## The Cost in RTTs

RTT (Round Trip Time) = time to send something and receive a response.

```
TCP handshake:   1 RTT   ← connection opens
TLS handshake:   1 RTT   ← tunnel established  (TLS 1.3)
First HTTP data: 1 RTT   ← your request + response
```

TLS 1.3 (current version) reduced the handshake to **1 RTT**.
It also has a **0-RTT** mode for clients that have connected before — the tunnel is established with the first data packet.

---

## Certificate Authorities (CA)

The certificate is only trustworthy because it was signed by a CA that your browser already trusts.

| CA | Notes |
|---|---|
| **Let's Encrypt** | Free, automated, widely used for web apps |
| **DigiCert** | Paid, used by large enterprises |
| **Sectigo** | Paid, common for commercial sites |
| **AWS Certificate Manager** | Free for AWS services, managed automatically |
| **Google Trust Services** | Used internally by Google products |
| **Cloudflare** | Issues certs automatically for proxied sites |

---

## Design Notes

- TLS 1.0 and 1.1 are deprecated — use TLS 1.2 minimum, 1.3 preferred
- A certificate has an expiration date — expired cert = browser warning
- Let's Encrypt certificates expire every 90 days but auto-renew
- mTLS (mutual TLS) = both sides present certificates, not just the server

---

## Related Notes

- [[SecurityAuth]]
- [[TCP]]
