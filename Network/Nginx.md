# Nginx

Nginx is a web server that also works as a **reverse proxy** — it sits in front of other servers and decides where to send each request based on rules you define.

It does not run your application. It receives the request and forwards it to whatever is actually running the app.

```
Client → Nginx → your app (PHP, Node, Go, etc.)
```

---

## Reverse Proxy vs Direct Server

Without Nginx:
```
Browser ──────────────────► PHP app :8080
```

With Nginx:
```
Browser ──► Nginx :80/:443 ──► PHP app :8080
```

**Why bother?** Nginx gives you:
- One entry point for multiple apps
- SSL/TLS termination (HTTPS in one place)
- Path-based or host-based routing
- Load balancing

---

## The Config File

Nginx config lives at `/etc/nginx/nginx.conf`, but the actual rules are usually in separate files loaded from `/etc/nginx/conf.d/`.

The structure:

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

### location blocks

Each `location` block matches a URL prefix. The most specific match wins.

```
GET /frution/index.php  →  matches location /frution/  ✓
GET /diprotec/login.php →  matches location /diprotec/ ✓
GET /unknown/           →  matches location /          → 404
```

### proxy_pass

`proxy_pass http://web-frution` tells Nginx to forward the request to a server named `web-frution`.

In Docker Compose, the service name IS the hostname — containers on the same network resolve each other by service name.

```
Nginx container → DNS lookup "web-frution" → Docker DNS → container IP
```

**Important:** `proxy_pass http://web-frution` (no trailing slash) keeps the URL prefix intact:

```
GET /frution/index.php  →  forwarded as  GET /frution/index.php  ✓
GET /frution/index.php  →  forwarded as  GET /index.php          ✗  (with trailing slash)
```

---

## Reading the Access Log

The access log format is:

```
ip - - [date] "request" status bytes "referer" "user-agent" "extra"
```

### Normal request
```
172.18.0.1 - - [28/May/2026:01:57:57 +0000] "GET /aplicsil/ HTTP/1.1" 200 142 "-" "curl/8.5.0" "-"
```

| Field | Value | Meaning |
|---|---|---|
| IP | `172.18.0.1` | who connected |
| Request | `GET /aplicsil/ HTTP/1.1` | what they asked for |
| Status | `200` | success |
| Bytes | `142` | response size |

### TLS ClientHello hitting an HTTP port

```
173.249.9.58 - - [28/May/2026:01:57:43 +0000] "\x16\x03\x01\x05\xCC\x01\x00..." 400 157 "-" "-" "-"
```

`\x16\x03\x01` is not a text request — it is the first bytes of a **TLS ClientHello**. Someone tried to connect via HTTPS to a port that only speaks HTTP.

Nginx received raw bytes it could not parse as HTTP and responded with `400 Bad Request`.

```
What the client sent:    [TLS handshake bytes]
What Nginx expected:     GET /path HTTP/1.1

Result: 400
```

This happens when:
- Cloudflare sends HTTPS traffic to an origin server that only has HTTP
- A browser tries `https://host:port` but the port has no SSL configured

**Fix:** either configure SSL on Nginx (add `listen 443 ssl` + certificate), or set Cloudflare SSL mode to **Flexible** so it sends HTTP to the origin.

---

## The "host not found in upstream" Error

```
[emerg] host not found in upstream "web-diprotec" in empresas/diprotec.conf:2
```

Nginx resolves all upstream names **at startup**. If `web-diprotec` is not running yet, DNS lookup fails and Nginx refuses to start.

This is why Nginx crashes in a restart loop when the upstream containers are still coming up.

**Solutions:**

Option A — add `depends_on` in docker-compose so Nginx waits:
```yaml
nginx:
  depends_on:
    - web-frution
    - web-diprotec
```

Option B — use a Docker DNS resolver so Nginx resolves upstreams dynamically instead of at startup:
```nginx
resolver 127.0.0.11 valid=10s;  # Docker's internal DNS

server {
    location /frution/ {
        set $upstream http://web-frution;
        proxy_pass $upstream;
    }
}
```

With the variable trick (`set $upstream`), Nginx defers DNS lookup to request time — if the container is not up yet, it returns 502 for that request instead of refusing to start entirely.

---

## HTTPS on Nginx

To accept HTTPS, Nginx needs a certificate and `listen 443 ssl`:

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

Get a free certificate with Let's Encrypt:
```bash
certbot certonly --standalone -d yourdomain.com
# renews automatically every 90 days
```

### Cloudflare + Nginx SSL modes

```
Flexible  →  Browser ──HTTPS──► Cloudflare ──HTTP──►  Nginx (no cert needed)
Full      →  Browser ──HTTPS──► Cloudflare ──HTTPS──► Nginx (any cert, even self-signed)
Full Strict → Browser ──HTTPS──► Cloudflare ──HTTPS──► Nginx (valid cert required)
```

Flexible is the simplest. Full Strict is the most secure.

---

## Quick Reference

```
nginx -t                    → test config for syntax errors
nginx -s reload             → reload config without downtime
nginx -s quit               → graceful shutdown

docker compose exec nginx nginx -t          → test config inside container
docker compose exec nginx nginx -s reload   → reload inside container
docker compose logs nginx                   → view logs
```

Log signals:
```
"GET /path HTTP/1.1" 200    → normal request, success
"GET /path HTTP/1.1" 502    → upstream not responding
"GET /path HTTP/1.1" 404    → no location matched
"\x16\x03\x01..." 400       → HTTPS attempt on HTTP port (TLS ClientHello)
host not found in upstream  → container not running when Nginx started
```

---

## Related Notes

- [[TLS]]
- [[TCP]]
- [[Docker]]
