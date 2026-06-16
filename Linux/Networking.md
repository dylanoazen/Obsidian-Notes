# Networking

Como o Linux lida com comunicação de rede — da pilha TCP/IP aos sockets.

Related: [[Linux/Kernel]], [[Network/Protocols]]

---

## A Pilha de Rede

```
Application layer     HTTP, DNS, SSH
        │
Transport layer       TCP, UDP
        │
Network layer         IP, ICMP, routing
        │
Link layer            Ethernet, ARP, NIC driver
        │
Hardware              Network Interface Card (NIC)
```

Cada camada adiciona/remove headers conforme os pacotes percorrem a pilha.

## Sockets

Sockets são a API para comunicação de rede:

```
Server:                          Client:
socket()                         socket()
bind(addr:port)                      │
listen()                             │
accept() ◄──── TCP handshake ────► connect(addr:port)
read()/write() ◄── data ──────► read()/write()
close()                          close()
```

```bash
# View open sockets
ss -tlnp          # TCP listening sockets with process info
ss -tunap          # all TCP/UDP connections
netstat -tlnp      # legacy equivalent
```

## Network Namespaces

Cada namespace possui seus próprios:
- Interfaces de rede
- Endereços IP
- Tabela de roteamento
- Regras de firewall (iptables/nftables)
- Bindings de socket

É assim que containers obtêm isolamento de rede.

```bash
# Create network namespace
ip netns add myns

# Run command in namespace
ip netns exec myns ip addr

# Create virtual ethernet pair (connects two namespaces)
ip link add veth0 type veth peer name veth1
ip link set veth1 netns myns
```

## iptables / nftables

Filtragem de pacotes e NAT no nível do kernel:

```bash
# View rules
iptables -L -n -v

# Block incoming on port 8080
iptables -A INPUT -p tcp --dport 8080 -j DROP

# NAT — masquerade outgoing traffic
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

nftables é o substituto moderno com sintaxe mais limpa.

## Arquivos e Comandos Principais

```bash
# DNS resolution
cat /etc/resolv.conf

# Hostname mapping
cat /etc/hosts

# Network interfaces
ip addr show
ip link show

# Routing table
ip route show

# Test connectivity
ping 8.8.8.8
traceroute google.com
dig google.com
curl -v https://example.com
```

## Related

- [[Linux/Kernel]]
- [[Linux/SystemCalls]]
- [[Network/TCP]]
- [[Network/Protocols]]

#### Meus comentários
- 
