# Docker

Docker is a tool that makes it possible to encapsulate a project inside a container that reflects exactly the real application. This makes the project behave the same on every machine, because the Docker environment is always exactly the same.

---

## Image

The result of building a Dockerfile. It is a static, read-only snapshot of the environment — the OS, dependencies, configs, and code all packed together. Images are used to create containers.

```bash
# Pull an image from Docker Hub
docker pull ubuntu:20.04

# List all local images
docker images

# Remove an image
docker rmi ubuntu:20.04
```

---

## Container

A running instance of an image. While an image is static, a container is alive — it has its own process, memory, and filesystem. You can run multiple containers from the same image, all isolated from each other.

```bash
# Run a container from an image
docker run -d -p 8080:80 --name my-container nginx

# -d = run in background
# -p = map host port 8080 to container port 80
# --name = give it a name
```

---

## Dockerfile

The blueprint that Docker uses to build an image. Like an ISO — it defines the operating system and the tools that come pre-configured with it.

```dockerfile
FROM ubuntu:20.04              # base OS

RUN apt-get update && \
    apt-get install -y \
    apache2 \                  # web server
    mysql-server \             # database
    openssh-server \           # SSH
    php \                      # PHP
    php-mysql                  # PHP <-> MySQL

COPY acesse/ /var/www/html/    # copy project files

EXPOSE 80 22                   # documents the ports used

CMD ["bash"]                   # default command
```

To build an image from a Dockerfile:
```bash
docker build -t my-image:1.0 .
```

---

## Docker Hub & Registry

Docker Hub is the official registry where images are stored and shared. When you run `docker pull`, it downloads from there.

```bash
# Authenticate
docker login

# Download an image
docker pull nginx:latest

# Upload your own image
docker tag my-image:1.0 username/my-image:1.0
docker push username/my-image:1.0
```

---

## docker-compose.yml

A file that defines and manages multi-container applications. Instead of running long `docker run` commands manually, you describe everything in a single file.

```yaml
services:
  web:
    image: tidiprotec/acesse       # image to use
    container_name: acesse-diprotec
    ports:
      - "44442:80"                 # host:container
      - "44422:22"
    environment:
      - MYSQL_ROOT_PASSWORD=secret # environment variables
    volumes:
      - mysql_data:/var/lib/mysql  # persist database
      - ./html:/var/www/html       # mirror local folder
    restart: always                # restart if it crashes

volumes:
  mysql_data:
```

```bash
# Start all services in background
docker compose up -d

# Bring everything down
docker compose down

# Rebuild and start
docker compose up -d --build
```

---

## Environment Variables

Used to pass configuration into containers without hardcoding values — passwords, ports, modes, etc.

```bash
# Pass a variable when running a container
docker run -e MYSQL_ROOT_PASSWORD=secret mysql

# Pass multiple variables
docker run -e APP_ENV=production -e PORT=8080 my-app
```

In docker-compose.yml:
```yaml
environment:
  - MYSQL_ROOT_PASSWORD=secret
  - APP_ENV=production
```

Or using an `.env` file:
```yaml
env_file:
  - .env
```

---

## Networking

How containers communicate with the outside world — each container runs internally on standard ports (80, 22), but needs a unique port on the host machine to be accessible.

```
YOUR MACHINE (host)
┌─────────────────────────────────────────────┐
│                                             │
│   port 44442 ──────► container diprotec    │
│                         internal port 80   │
│                                             │
│   port 44440 ──────► container tecnisul    │
│                         internal port 80   │
│                                             │
│   Browser: http://localhost:44442          │
└─────────────────────────────────────────────┘
```

**Rules:**
- Inside the container, services always run on standard ports (80, 22)
- Outside (on your machine), each container needs a different port
- Two containers cannot use the same host port at the same time

---

## Volumes

Volumes are how data is persisted outside of containers. Without volumes, all data is lost when the container is removed.

> ⚠️ If you run `docker compose down` without volumes, all MySQL data inside the container is **lost**.

```yaml
services:
  web:
    volumes:
      - mysql_data:/var/lib/mysql  # saves the database outside the container
      - ./html:/var/www/html       # mirrors a local folder

volumes:
  mysql_data:                      # Docker manages this volume
```

| Type | Example | Use |
|------|---------|-----|
| Named volume | `mysql_data:/var/lib/mysql` | Docker manages it. Good for databases |
| Bind mount | `./html:/var/www/html` | Mirrors a local folder. Good for development |

```bash
# List existing volumes
docker volume ls

# Inspect a volume
docker volume inspect mysql_data
```

---

## Docker Commands

### Observation

```bash
# What is running right now?
docker ps

# All containers (including stopped ones)
docker ps -a

# What images do I have?
docker images

# How much space is Docker using?
docker system df

# Logs of a container (check for errors)
docker logs acesse-diprotec

# Live logs (keeps following)
docker logs -f acesse-diprotec

# Full details of a container
docker inspect acesse-diprotec
```

### Interact

```bash
# Enter the container (like SSH into a VM)
docker exec -it acesse-diprotec /bin/bash

# Run a single command without entering
docker exec acesse-diprotec ls /var/www/html

# Copy a file from your machine into the container
docker cp myfile.sql acesse-diprotec:/tmp/

# Copy a file from the container to your machine
docker cp acesse-diprotec:/var/log/apache2/error.log ./error.log
```

### Control

```bash
# Start with docker-compose (inside the folder with the file)
docker compose up -d          # -d = run in background

# Bring down
docker compose down

# Restart a container
docker restart acesse-diprotec

# Stop without removing
docker stop acesse-diprotec

# Start again
docker start acesse-diprotec
```

### Cleaning

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Clean EVERYTHING not in use (careful!)
docker system prune
```

### Debug

```bash
# Container won't start? Check the logs:
docker logs acesse-diprotec

# Container restarting in a loop? Check why:
docker inspect acesse-diprotec | grep -A 5 "State"

# Apache not responding? Enter and check:
docker exec -it acesse-diprotec /bin/bash
service apache2 status
cat /var/log/apache2/error.log

# MySQL not connecting?
docker exec -it acesse-diprotec /bin/bash
service mysql status
mysql -u root -p

# Port in use? Check on the host:
ss -tulnp | grep 44442
```

---

## Full Workflow

```
1. docker login                                        ← authenticate on Docker Hub
         │
2. docker pull tidiprotec/acesse                       ← pull the image
         │
3. docker compose up -d                                ← start the container
         │
4. docker ps                                           ← confirm it's running
         │
5. http://localhost:44442/impertrade                   ← access in the browser
         │
6. docker exec -it acesse-diprotec /bin/bash           ← enter the container
         │
7. cd /home/cesar && ./voltar company_name             ← import the database
         │
8. docker compose down                                 ← bring down when done
```

---

## Quick Reference

```bash
docker ps                    → who is running?
docker logs NAME             → what happened?
docker exec -it NAME bash    → I want to enter
docker compose up -d         → start everything
docker compose down          → stop everything
docker cp ORIG DEST          → copy files
docker inspect NAME          → full details
```
