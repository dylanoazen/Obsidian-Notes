 
The TCP protocol allows two processes to exchange bytes over the network.

TCP is NOT like:
- REST APIs
- JSON
- request/response abstractions

TCP only understands:
# byte streams.

Everything sent through TCP eventually becomes bytes, including:
- strings
- JSON
- structs
- files

The server that receives the bytes must interpret and parse the data correctly.

---

# Listener

A listener is the open port where the server waits for incoming TCP connections.

Example:

```text
localhost:8080
```

The server keeps listening for clients trying to connect.

---

# Connection (`conn`)

Example client connection:

```bash
nc localhost 8080
```

When a client connects, the server receives a TCP connection object (`conn`).

This connection represents a communication channel between:
- client
- server

---

# Stream

A TCP connection is a continuous stream of bytes.

TCP does NOT know:
- messages
- commands
- JSON structures
- request boundaries

Because of that, the application itself must define:
# how messages are separated and interpreted.

Example:

```text
SET name Dylan\n
```

The application may decide that:
- each line
- or each delimiter

represents a complete command.

---

# Framing

Because TCP is a byte stream with no message boundaries, the application must define how to separate messages. This is called framing.

See [[Framing]] for the full breakdown of framing types and how to implement them.

---

# My Comments:

TCP provides a reliable, ordered byte stream. A connection is established with a three‑way handshake: the client sends SYN, the server replies with SYN‑ACK, and the client sends ACK (which may include data). After that, both sides exchange bytes in order; TCP does not preserve message boundaries. 
The byte stream is the way that the macinhes send and recive data it works this way:
TCP does **not** first send the total length of the stream. Instead, it splits data into segments, each with a **sequence number**. The receiver acknowledges the highest contiguous sequence number received (ACK). Lost segments are retransmitted. This is how TCP ensures reliable, ordered delivery over an unreliable network.