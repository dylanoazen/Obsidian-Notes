# Framing

Framing is a technique to mark where a message starts and where it ends inside a TCP byte stream.

Since [[TCP]] does not preserve message boundaries, the application itself must define how to separate messages. Without framing, the receiver has no way to know if it received a complete message, half a message, or two messages at once.

## Framing Types

### Delimiter-based

Uses a special character to signal the end of a message. The most common delimiter is `\n` (newline).

```text
SET name Dylan\n   ← message 1
GET name\n         ← message 2
```

The receiver reads byte by byte until it finds the delimiter, then processes the complete message.

**Used in:** Redis protocol, HTTP headers, simple TCP servers.

**Downside:** The delimiter character cannot appear inside the message data.

---

### Length-prefix

Sends the size of the message before the message itself. The receiver reads the length first, then reads exactly that many bytes.

```text
[0 0 0 11] [SET name Dylan]
 ↑ 4 bytes    ↑ 11 bytes
  (length)    (message)
```

```go
// sending
binary.Write(conn, binary.BigEndian, uint32(len(msg)))
conn.Write(msg)

// receiving
var length uint32
binary.Read(conn, binary.BigEndian, &length)
buf := make([]byte, length)
io.ReadFull(conn, buf)
```

**Used in:** gRPC, databases, most binary protocols.

**Advantage:** Works with any data, including binary.

---

### Fixed-size

Every message is always the same size. The receiver always reads exactly N bytes.

```text
[msg 1 - 64 bytes][msg 2 - 64 bytes][msg 3 - 64 bytes]
```

**Used in:** low-level hardware protocols, some game engines.

**Downside:** Wastes space if messages are shorter than the fixed size.

---

## Summary

| Type | How it works | Best for |
|------|-------------|----------|
| Delimiter | Special character marks the end | Text protocols, simple servers |
| Length-prefix | Size sent before the message | Binary data, production systems |
| Fixed-size | Every message is the same size | Known, predictable message sizes |
