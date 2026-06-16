# TCP/IP Deep Dive

Uma análise mais aprofundada do modelo TCP/IP — como os dados fluem da aplicação até o fio.

Related: [[Network/TCP]], [[Network/Protocols]], [[Network/UDP]]

---

## O Modelo TCP/IP

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

vs OSI (7 camadas): o TCP/IP mescla Presentation + Session em Application, e Data Link + Physical em Link.

## IP — Internet Protocol

### IPv4

```
Source: 192.168.1.10
Dest:   93.184.216.34
TTL:    64
Proto:  6 (TCP)
```

- Endereços de 32 bits (4,3 bilhões — insuficiente, daí o NAT e o IPv6)
- Header: mínimo 20 bytes
- **TTL**: decrementado em cada hop, evita loops de roteamento infinitos
- **Fragmentation**: roteadores podem dividir pacotes grandes (MTU tipicamente 1500 bytes)

### IPv6

- Endereços de 128 bits: `2001:0db8:85a3::8a2e:0370:7334`
- Sem necessidade de NAT — cada dispositivo recebe um endereço global
- Sem fragmentação por roteadores (a origem lida com isso)
- Suporte a IPsec nativo

### Subnetting

```
192.168.1.0/24
│           │ └── 24 bits para rede, 8 bits para hosts (254 utilizáveis)
│           └──── endereço de rede
└──────────────── primeiro octeto

Notação CIDR:
/8  = 255.0.0.0       = 16M hosts
/16 = 255.255.0.0     = 65K hosts
/24 = 255.255.255.0   = 254 hosts
/32 = host único
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

### Controle de Fluxo — Sliding Window

- O receptor anuncia o **window size**: quantidade de dados que consegue armazenar em buffer
- O remetente envia até o window size sem esperar pelos ACKs
- A janela diminui conforme os dados preenchem o buffer, e cresce conforme o app lê os dados
- Impede que o remetente sobrecarregue um receptor lento

### Controle de Congestionamento

```
Slow Start → Crescimento exponencial até o threshold
     │
Congestion Avoidance → Crescimento linear
     │
Perda de Pacote Detectada → Reduz janela, reinicia
```

Algoritmos: Reno, Cubic (padrão no Linux), BBR (Google)

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

## Portas Comuns

| Porta | Protocolo | Serviço |
|------|----------|---------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alt |

## Ferramentas de Diagnóstico

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

## Relacionados

- [[Network/TCP]]
- [[Network/UDP]]
- [[Network/Protocols]]
- [[Linux/Networking]]

## Resources

- https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/
- TCP/IP Illustrated — W. Richard Stevens (livro)

#### Meus comentários
- 
