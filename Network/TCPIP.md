# TCP/IP Deep Dive

A deeper look into the TCP/IP model — how data actually flows from application to wire.

Related: [[Network/TCP]], [[Network/Protocols]], [[Network/UDP]]

---

## The TCP/IP Model

```
Application    HTTP, DNS, SSH, SMTP, FTP
     │
Transport      TCP (reliable) / UDP (fast)
     │
Internet       IP (addressing, routing)
     │
Link           Ethernet, Wi-Fi, ARP
     │
Physical       Cables, radio waves, signals
```

vs OSI (7 layers): TCP/IP merges Presentation + Session into Application, and Data Link + Physical into Link.

## IP — Internet Protocol

### IPv4

```
Source: 192.168.1.10
Dest:   93.184.216.34
TTL:    64
Proto:  6 (TCP)
```

- 32-bit addresses (4.3 billion — not enough, hence NAT and IPv6)
- Header: 20 bytes minimum
- **TTL**: decremented at each hop, prevents infinite routing loops
- **Fragmentation**: routers can split large packets (MTU typically 1500 bytes)

### IPv6

- 128-bit addresses: `2001:0db8:85a3::8a2e:0370:7334`
- No NAT needed — every device gets a global address
- No fragmentation by routers (source handles it)
- Built-in IPsec support

### Subnetting

```
192.168.1.0/24
│           │ └── 24 bits for network, 8 bits for hosts (254 usable)
│           └──── network address
└──────────────── first octet

CIDR notation:
/8  = 255.0.0.0       = 16M hosts
/16 = 255.255.0.0     = 65K hosts
/24 = 255.255.255.0   = 254 hosts
/32 = single host
```

## TCP Deep Dive

### Three-Way Handshake

```
Client              Server
  │── SYN ──────────►│    "I want to connect" (seq=100)
  │◄── SYN+ACK ─────│    "OK, I acknowledge" (seq=300, ack=101)
  │── ACK ──────────►│    "Confirmed" (ack=301)
  │                   │
  │◄── Data ────────►│    Connection established
```

### Four-Way Teardown

```
Client              Server
  │── FIN ──────────►│    "I'm done sending"
  │◄── ACK ─────────│    "Got it"
  │◄── FIN ─────────│    "I'm done too"
  │── ACK ──────────►│    "Bye"
```

### Flow Control — Sliding Window

- Receiver advertises **window size**: how much data it can buffer
- Sender sends up to window size without waiting for ACKs
- Window shrinks as data fills buffer, grows as app reads data
- Prevents sender from overwhelming a slow receiver

### Congestion Control

```
Slow Start → Exponential growth until threshold
     │
Congestion Avoidance → Linear growth
     │
Packet Loss Detected → Cut window, restart
```

Algorithms: Reno, Cubic (Linux default), BBR (Google)

```bash
# Check current congestion algorithm
sysctl net.ipv4.tcp_congestion_control

# Switch to BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## DNS

```
Browser: "what's the IP for example.com?"
     │
     ▼
Local resolver cache → miss
     │
     ▼
Recursive resolver (ISP / 8.8.8.8)
     │
     ├── Root server → "ask .com"
     ├── TLD server (.com) → "ask ns.example.com"
     └── Authoritative server → "93.184.216.34, TTL=3600"
```

```bash
dig example.com          # full DNS query
dig +trace example.com   # show entire resolution chain
nslookup example.com
```

## Common Port Numbers

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alt |

## Diagnostic Tools

```bash
# Trace route
traceroute example.com
mtr example.com              # continuous traceroute

# Packet capture
tcpdump -i eth0 port 443
tcpdump -i any -w capture.pcap

# Analyze with Wireshark
wireshark capture.pcap

# Connection stats
ss -s                        # summary
ss -tlnp                     # listening TCP
netstat -i                   # interface stats
```

## Related

- [[Network/TCP]]
- [[Network/UDP]]
- [[Network/Protocols]]
- [[Linux/Networking]]

## Resources

- https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/
- TCP/IP Illustrated — W. Richard Stevens (book)

#### My commentaries
- 
