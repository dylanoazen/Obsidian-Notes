# Networking

How Linux handles network communication — from the TCP/IP stack to sockets.

Related: [[Linux/Kernel]], [[Network/Protocols]]

---

## The Network Stack

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

Each layer adds/removes headers as packets move through the stack.

## Sockets

Sockets are the API for network communication:

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

Each namespace has its own:
- Network interfaces
- IP addresses
- Routing table
- Firewall rules (iptables/nftables)
- Socket bindings

This is how containers get isolated networking.

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

Kernel-level packet filtering and NAT:

```bash
# View rules
iptables -L -n -v

# Block incoming on port 8080
iptables -A INPUT -p tcp --dport 8080 -j DROP

# NAT — masquerade outgoing traffic
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

nftables is the modern replacement with cleaner syntax.

## Key Files and Commands

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

#### My commentaries
- 
