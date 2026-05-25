# Net

The `net` package provides a portable interface for network I/O, including TCP/UDP, IP addresses, DNS lookups, and connection handling.

## Importing

```go
import "net"
```

---

## Interfaces

### Conn

Represents a network connection ([[TCP]]/[[UDP]]).

Methods:
- [[Methods#Read|Read]]
- [[Methods#Write|Write]]
- [[Methods#Close|Close]]
- [[Methods#LocalAddr|LocalAddr]]
- [[Methods#RemoteAddr|RemoteAddr]]
- [[Methods#SetDeadline|SetDeadline]]
- [[Methods#SetReadDeadline|SetReadDeadline]]
- [[Methods#SetWriteDeadline|SetWriteDeadline]]

---

### Listener

Listens for incoming connections.

Methods:
- [[Methods#Accept|Accept]]
- [[Methods#Close|Close]]
- [[Methods#Addr|Addr]]

---

### PacketConn

Represents a packet-oriented network connection (used with UDP).

Methods:
- [[Methods#ReadFrom|ReadFrom]]
- [[Methods#WriteTo|WriteTo]]
- [[Methods#Close|Close]]
- [[Methods#LocalAddr|LocalAddr]]
- [[Methods#SetDeadline|SetDeadline]]

---

## Functions

This is a summary of the most important functions available in the Net package. For the full list, see the [official documentation](https://pkg.go.dev/net).

---

### Dialing & Listening

#### Listen
Creates a listener for TCP or Unix sockets.
```go
ln, err := net.Listen("tcp", ":8080")
```

> Every time you open a server, if you want to handle the `\n` character, you need to use the [[Bufio]] package.

---

#### ListenPacket
Creates a packet connection for UDP.
```go
conn, err := net.ListenPacket("udp", ":9000")
```

---

#### Dial
Connects to a remote address. If **Listen** is the "ears" waiting for connections, **Dial** is the "mouth" that initiates a connection.
```go
conn, err := net.Dial("tcp", "localhost:8080")
```

---

#### DialTimeout
Like `Dial`, but cancels if the connection takes too long.
```go
conn, err := net.DialTimeout("tcp", "localhost:8080", 3*time.Second)
```

---

### DNS & Name Resolution

#### LookupHost
Returns the IP addresses for a given hostname.
```go
addrs, err := net.LookupHost("google.com")
// ["142.250.185.46", ...]
```

---

### IP Utilities

#### ParseIP
Parses an IP address string into an `IP` type.
```go
ip := net.ParseIP("192.168.0.1")
```

---

### Error Handling

#### net.ErrClosed
Returned when you try to use a connection or listener that has already been closed.
```go
if errors.Is(err, net.ErrClosed) {
    // connection was already closed
}
```
