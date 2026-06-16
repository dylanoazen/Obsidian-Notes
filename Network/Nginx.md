# Nginx

Nginx é um servidor web que também funciona como **reverse proxy** — ele fica na frente de outros servidores e decide para onde enviar cada requisição com base em regras que você define.

Ele não executa sua aplicação. Ele recebe a requisição e a encaminha para o que está de fato rodando o app.

```
Client → Nginx → your app (PHP, Node, Go, etc.)
```

---

## Reverse Proxy vs Servidor Direto

Sem Nginx:
```
Browser ──────────────────► PHP app :8080
```

Com Nginx:
```
Browser ──► Nginx :80/:443 ──► PHP app :8080
```

**Por que usar?** O Nginx oferece:
- Um único ponto de entrada para múltiplos apps
- SSL/TLS termination (HTTPS em um único lugar)
- Roteamento por path ou por host
- Load balancing

---

## O Arquivo de Configuração

A configuração do Nginx fica em `/etc/nginx/nginx.conf`, mas as regras de fato ficam geralmente em arquivos separados carregados de `/etc/nginx/conf.d/`.

A estrutura:

```nginx
server {
    listen 80;          # which port to listen on
    server_name _;      # which hostname to match (_ = catch-all)

    location /frution/ {
        proxy_pass http://web-frution;   # forward to this upstream
    }

    location /diprotec/ {
        proxy_pass http://web-diprotec;
    }

    location / {
        return 404 "Not found.";         # no match → 404
    }
}
```

### Blocos location

Cada bloco `location` faz match com um prefixo de URL. O match mais específico vence.

```
GET /frution/index.php  →  matches location /frution/  ✓
GET /diprotec/login.php →  matches location /diprotec/ ✓
GET /unknown/           →  matches location /          → 404
```

### proxy_pass

`proxy_pass http://web-frution` instrui o Nginx a encaminhar a requisição para um servidor chamado `web-frution`.

No Docker Compose, o nome do serviço É o hostname — containers na mesma rede se resolvem pelo nome do serviço.

```
Nginx container → DNS lookup "web-frution" → Docker DNS → container IP
```

**Importante:** `proxy_pass http://web-frution` (sem barra no final) mantém o prefixo da URL intacto:

```
GET /frution/index.php  →  forwarded as  GET /frution/index.php  ✓
GET /frution/index.php  →  forwarded as  GET /index.php          ✗  (with trailing slash)
```

---

## Lendo o Log de Acesso

O formato do log de acesso é:

```
ip - - [date] "request" status bytes "referer" "user-agent" "extra"
```

### Requisição normal
```
172.18.0.1 - - [28/May/2026:01:57:57 +0000] "GET /aplicsil/ HTTP/1.1" 200 142 "-" "curl/8.5.0" "-"
```

| Campo | Valor | Significado |
|---|---|---|
| IP | `172.18.0.1` | quem conectou |
| Request | `GET /aplicsil/ HTTP/1.1` | o que foi solicitado |
| Status | `200` | sucesso |
| Bytes | `142` | tamanho da resposta |

### TLS ClientHello chegando em uma porta HTTP

```
173.249.9.58 - - [28/May/2026:01:57:43 +0000] "\x16\x03\x01\x05\xCC\x01\x00..." 400 157 "-" "-" "-"
```

`\x16\x03\x01` não é uma requisição de texto — são os primeiros bytes de um **TLS ClientHello**. Alguém tentou conectar via HTTPS em uma porta que só fala HTTP.

O Nginx recebeu bytes brutos que não conseguiu interpretar como HTTP e respondeu com `400 Bad Request`.

```
O que o cliente enviou:    [TLS handshake bytes]
O que o Nginx esperava:    GET /path HTTP/1.1

Resultado: 400
```

Isso acontece quando:
- O Cloudflare envia tráfego HTTPS para um servidor de origem que só tem HTTP
- Um browser tenta `https://host:port` mas a porta não tem SSL configurado

**Solução:** configure SSL no Nginx (adicione `listen 443 ssl` + certificado), ou configure o modo SSL do Cloudflare como **Flexible** para que ele envie HTTP para a origem.

---

## O Erro "host not found in upstream"

```
[emerg] host not found in upstream "web-diprotec" in empresas/diprotec.conf:2
```

O Nginx resolve todos os nomes de upstream **na inicialização**. Se `web-diprotec` ainda não estiver rodando, a resolução DNS falha e o Nginx se recusa a iniciar.

É por isso que o Nginx entra em loop de reinicialização quando os containers de upstream ainda estão subindo.

**Soluções:**

Opção A — adicione `depends_on` no docker-compose para o Nginx esperar:
```yaml
nginx:
  depends_on:
    - web-frution
    - web-diprotec
```

Opção B — use um resolver DNS do Docker para que o Nginx resolva upstreams dinamicamente em vez de na inicialização:
```nginx
resolver 127.0.0.11 valid=10s;  # Docker's internal DNS

server {
    location /frution/ {
        set $upstream http://web-frution;
        proxy_pass $upstream;
    }
}
```

Com o truque da variável (`set $upstream`), o Nginx adia a resolução DNS para o momento da requisição — se o container ainda não estiver no ar, ele retorna 502 para aquela requisição em vez de se recusar a iniciar completamente.

---

## HTTPS no Nginx

Para aceitar HTTPS, o Nginx precisa de um certificado e de `listen 443 ssl`:

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location /frution/ {
        proxy_pass http://web-frution;
    }
}
```

Obtenha um certificado gratuito com Let's Encrypt:
```bash
certbot certonly --standalone -d yourdomain.com
# renews automatically every 90 days
```

### Modos SSL Cloudflare + Nginx

```
Flexible  →  Browser ──HTTPS──► Cloudflare ──HTTP──►  Nginx (no cert needed)
Full      →  Browser ──HTTPS──► Cloudflare ──HTTPS──► Nginx (any cert, even self-signed)
Full Strict → Browser ──HTTPS──► Cloudflare ──HTTPS──► Nginx (valid cert required)
```

Flexible é o mais simples. Full Strict é o mais seguro.

---

## Referência Rápida

```
nginx -t                    → test config for syntax errors
nginx -s reload             → reload config without downtime
nginx -s quit               → graceful shutdown

docker compose exec nginx nginx -t          → test config inside container
docker compose exec nginx nginx -s reload   → reload inside container
docker compose logs nginx                   → view logs
```

Sinais de log:
```
"GET /path HTTP/1.1" 200    → normal request, success
"GET /path HTTP/1.1" 502    → upstream not responding
"GET /path HTTP/1.1" 404    → no location matched
"\x16\x03\x01..." 400       → HTTPS attempt on HTTP port (TLS ClientHello)
host not found in upstream  → container not running when Nginx started
```

---

## Notas Relacionadas

- [[TLS]]
- [[TCP]]
- [[Docker]]
