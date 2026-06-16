# Net

O package `net` fornece uma interface portável para I/O de rede, incluindo TCP/UDP, endereços IP, lookups de DNS e gerenciamento de conexões.

## Importing

```go
import "net"
```

---

## Interfaces

### Conn

Representa uma conexão de rede ([[TCP]]/[[UDP]]).

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

Escuta conexões de entrada.

Methods:
- [[Methods#Accept|Accept]]
- [[Methods#Close|Close]]
- [[Methods#Addr|Addr]]

---

### PacketConn

Representa uma conexão de rede orientada a pacotes (usada com UDP).

Methods:
- [[Methods#ReadFrom|ReadFrom]]
- [[Methods#WriteTo|WriteTo]]
- [[Methods#Close|Close]]
- [[Methods#LocalAddr|LocalAddr]]
- [[Methods#SetDeadline|SetDeadline]]

---

## Funções

Este é um resumo das funções mais importantes disponíveis no package Net. Para a lista completa, veja a [documentação oficial](https://pkg.go.dev/net).

---

### Dialing & Listening

#### Listen
Cria um listener para sockets TCP ou Unix.
```go
ln, err := net.Listen("tcp", ":8080")
```

> Toda vez que você abrir um servidor, se quiser tratar o caractere `\n`, você precisa usar o package [[Bufio]].

---

#### ListenPacket
Cria uma conexão de pacotes para UDP.
```go
conn, err := net.ListenPacket("udp", ":9000")
```

---

#### Dial
Conecta a um endereço remoto. Se **Listen** é os "ouvidos" esperando por conexões, **Dial** é a "boca" que inicia uma conexão.
```go
conn, err := net.Dial("tcp", "localhost:8080")
```

---

#### DialTimeout
Como `Dial`, mas cancela se a conexão demorar demais.
```go
conn, err := net.DialTimeout("tcp", "localhost:8080", 3*time.Second)
```

---

### DNS & Name Resolution

#### LookupHost
Retorna os endereços IP para um dado hostname.
```go
addrs, err := net.LookupHost("google.com")
// ["142.250.185.46", ...]
```

---

### IP Utilities

#### ParseIP
Faz o parse de uma string de endereço IP para o tipo `IP`.
```go
ip := net.ParseIP("192.168.0.1")
```

---

### Error Handling

#### net.ErrClosed
Retornado quando você tenta usar uma conexão ou listener que já foi fechado.
```go
if errors.Is(err, net.ErrClosed) {
    // connection was already closed
}
```
