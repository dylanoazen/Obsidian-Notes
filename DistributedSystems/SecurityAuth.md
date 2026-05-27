# Security & Auth

Security in distributed systems is about two fundamental questions:

- **Who are you?** → Authentication
- **What can you do?** → Authorization

---

## Authentication

Authentication is the act of proving your identity before being allowed in.

The most common forms:

| Method | How it works |
|---|---|
| Password | You know a secret string |
| Token (JWT) | You hold a signed token issued by the server |
| API Key | You hold a key tied to your identity |
| SSH Key | You hold a private key that matches a public key on the server |
| Certificate (mTLS) | Your machine holds a cryptographic certificate |

**Analogy:** the bouncer at the door — you only get in if you can prove who you are.

Authentication answers: **"are you who you say you are?"**

### SSH Keys

SSH keys work as a key pair:

```
Private key  ← stays on your machine, never leaves
Public key   ← you give to the server upfront
```

When you try to connect:

```
You        →  "I want in"
Server     →  sends an encrypted challenge
You        →  deciphers it with your private key and responds
Server     →  only the private key holder could do that → allowed in ✓
```

You never send a password over the network — you prove your identity by solving a cryptographic challenge.

**Analogy:** a padlock with two keys. You give the open padlock to the server. Only you have the key to open it. If you can open it, you proved you are you.

| | **Password** | **SSH Key** |
|---|---|---|
| What travels over the network | The password (even if encrypted) | Nothing sensitive — only the proof |
| Can be stolen remotely | Yes | No — the private key never leaves your machine |
| Brute force possible | Yes | Practically impossible |

---

## Authorization

Authorization happens after authentication — it defines what an authenticated identity is allowed to do.

You can be authenticated (proven who you are) but still not authorized to do certain things.

**Analogy:** you have a keycard to enter the building *(authentication)*, but your keycard only opens the front door and your floor — not the server room *(authorization)*.

Authorization answers: **"are you allowed to do this?"**

---

## ACL (Access Control List)

An ACL is the most common way to implement authorization — a list that maps identities to permissions.

```
user: analytics  → can read  user:* keys
user: api        → can read and write user:* keys
user: admin      → can do everything
```

Each entry says: *this identity can do these actions on these resources.*

**Principle of least privilege:** give each user the minimum access required to do its job — nothing more. If a service is compromised, the blast radius is contained.

---

## Roles

Instead of assigning permissions to each user individually, you define **roles** — named bundles of permissions — and assign roles to users.

```
Role: readonly   → GET, LIST
Role: readwrite  → GET, LIST, SET, DELETE
Role: admin      → everything

User: analytics  → role: readonly
User: api        → role: readwrite
User: dylan      → role: admin
```

This makes it easy to manage permissions at scale — change the role, all users with that role are updated.

**Analogy:** job titles in a company. An intern has intern-level access. A manager has manager-level access. You define what each title can do, then assign titles to people.

---

## Authentication vs Authorization

A common source of confusion:

| | **Authentication** | **Authorization** |
|---|---|---|
| Question | Who are you? | What can you do? |
| Happens | First | After authentication |
| Example | Login with password | Can you access this endpoint? |
| If it fails | "I don't know you" | "I know you, but you can't do this" |

You cannot have authorization without authentication — you need to know who someone is before deciding what they can do.

---

## TLS — Encrypting the Connection

Authentication proves who you are. TLS encrypts what travels between client and server.

Without TLS, passwords and data travel as plain text — any router in the path can read everything. With TLS, an encrypted tunnel is created and anyone who intercepts traffic sees only noise.

> See [[TLS]] for the full breakdown — handshake, Diffie-Hellman key exchange, certificates, and how it layers on top of TCP.

---
a## In Redis

Redis implements these concepts natively:

- **Authentication** → `AUTH password` or username + password
- **Authorization** → ACL system with per-user command and key permissions
- **TLS** → encrypted connections between clients and server

```bash
# create a user with limited access
ACL SETUSER analytics on >secret ~user:* +GET

# user can only GET keys starting with user:
# cannot SET, DEL, FLUSHALL, or access other keys
```

Redis does not have built-in named roles — you simulate them by creating users with matching permission sets.

---

## Design Notes

- Always authenticate in production — never leave systems open
- Use roles to manage permissions at scale instead of per-user rules
- Principle of least privilege reduces damage when something is compromised
- TLS is mandatory when traffic crosses a network you do not fully control
- Audit who has access regularly — remove what is no longer needed

---

## Related Notes

- [[Replication]]
- [[Kubernetes]]
- [[TransactionsPipeline]]
