# Docker Avançado

Além do `docker run` — internals, otimização, segurança e padrões para produção.

Related: [[DevOps/Docker/Docker]], [[Linux/ProcessManagement]]

---

## Como o Docker Funciona por Dentro

Docker NÃO é uma VM. Um container é um processo Linux comum com isolamento:

```
┌─────────────────────────────────────────────┐
│ Host Kernel (shared by all containers)      │
├──────────┬──────────┬──────────┬────────────┤
│Container1│Container2│Container3│ Host procs  │
│ ns+cgroup│ ns+cgroup│ ns+cgroup│             │
└──────────┴──────────┴──────────┴────────────┘
```

- **Namespaces**: isolam PID, rede, mount, usuário, IPC
- **cgroups**: limitam CPU, memória, I/O
- **Union filesystem** (OverlayFS): armazenamento de imagem em camadas

## Camadas de Imagem

```dockerfile
FROM ubuntu:22.04          # Layer 1: base image
RUN apt-get update         # Layer 2: package index
RUN apt-get install -y curl # Layer 3: curl installed
COPY app /app              # Layer 4: your code
CMD ["/app/run"]           # Metadata, no layer
```

Cada instrução cria uma camada somente-leitura. O container adiciona uma camada gravável fina no topo. As camadas são cacheadas — a ordem importa para a velocidade de build.

## Multi-stage Builds

Mantenha a imagem final pequena separando build e runtime:

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o /app

# Stage 2: Runtime (only the binary)
FROM alpine:3.19
COPY --from=builder /app /app
CMD ["/app"]
```

Resultado: a imagem vai de ~1GB (Go SDK) para ~15MB (Alpine + binário).

## Boas Práticas de Dockerfile

```dockerfile
# Pin versions (reproducible builds)
FROM node:20.11-alpine3.19

# Combine RUN commands (fewer layers)
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# COPY before RUN for better caching
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# Non-root user
RUN addgroup -S app && adduser -S app -G app
USER app

# Use ENTRYPOINT + CMD pattern
ENTRYPOINT ["node"]
CMD ["server.js"]
```

## Networking

```bash
# Bridge (default) — containers on same bridge can talk
docker network create mynet
docker run --network mynet --name api myapi
docker run --network mynet --name db postgres
# 'api' can reach 'db' by hostname

# Host — container uses host network directly (no isolation)
docker run --network host myapp

# None — no networking
docker run --network none myapp
```

## Volumes e Storage

```bash
# Named volume (managed by Docker, persists)
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres

# Bind mount (host directory)
docker run -v $(pwd)/config:/app/config:ro myapp

# tmpfs (in-memory, no persistence)
docker run --tmpfs /tmp:rw,size=100m myapp
```

## Segurança

```bash
# Run as non-root
docker run --user 1000:1000 myapp

# Drop all capabilities, add only needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# Read-only filesystem
docker run --read-only --tmpfs /tmp myapp

# No new privileges
docker run --security-opt no-new-privileges myapp

# Seccomp profile
docker run --security-opt seccomp=profile.json myapp

# Scan for vulnerabilities
docker scout cves myimage:latest
```

## Docker Compose Avançado

```yaml
services:
  api:
    build:
      context: .
      target: production    # multi-stage target
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    depends_on:
      db:
        condition: service_healthy
```

## Debugging

```bash
# Shell into running container
docker exec -it <container> sh

# View logs
docker logs -f --tail 100 <container>

# Inspect filesystem changes
docker diff <container>

# Resource usage
docker stats

# Full inspection
docker inspect <container>

# View image layers
docker history myimage:latest
```

## Relacionados

- [[DevOps/CICD/CICD]]
- [[DevOps/CICD/GitHubActions]]
- [[Linux/Security]]
- [[Linux/ProcessManagement]]

## Resources

- https://docs.docker.com/build/building/best-practices/
- https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html

#### Meus comentários
- 
