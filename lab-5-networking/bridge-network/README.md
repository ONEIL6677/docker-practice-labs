# Docker Bridge Networks

A **bridge network** is Docker's default network driver. It creates an isolated virtual network on a **single host**, allowing containers on the same bridge to communicate while staying isolated from others.

> Use bridge networks for **single-host** deployments — local dev, small apps, or services that don't need to span multiple machines.

---

## How It Works

```
Host Machine
┌─────────────────────────────────────────┐
│                                         │
│   [Container A] ──┐                     │
│                   ├──► [docker0 bridge] ──► Host Network ──► Internet
│   [Container B] ──┘                     │
│                                         │
└─────────────────────────────────────────┘
```

- Docker creates a virtual switch (`docker0` by default)
- Containers on the same bridge can reach each other **by name** (DNS auto-resolved)
- Containers on **different bridges** are isolated from each other
- Traffic to the outside world goes through the host's NAT

---

## Quick Start

```bash
# Create a custom bridge network (always prefer custom over default)
docker network create --driver bridge my-app-net

# Run containers attached to it
docker run -d --name api --network my-app-net my-api-image
docker run -d --name db  --network my-app-net postgres:15

# 'api' can now reach 'db' simply by hostname: postgres://db:5432
```

---

## Production Example — Node API + PostgreSQL + Nginx

```yaml
# docker-compose.yml
version: "3.9"

services:

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"        # Only nginx is exposed to the host
    networks:
      - frontend         # Sits on the public-facing bridge
    depends_on:
      - api

  api:
    image: my-node-api:latest
    environment:
      DB_HOST: db        # Resolved by Docker DNS — no IP needed
      DB_PORT: 5432
    networks:
      - frontend         # Reachable by nginx
      - backend          # Reachable by db
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend          # Isolated — NOT reachable from nginx directly
    secrets:
      - db_pass

networks:
  frontend:
    driver: bridge       # nginx <-> api
  backend:
    driver: bridge       # api <-> db (db never exposed to frontend)

volumes:
  pg_data:

secrets:
  db_pass:
    external: true       # Managed via Docker Secrets or a vault
```

### Why two bridges?

| Bridge     | Who's on it        | Purpose                          |
|------------|--------------------|----------------------------------|
| `frontend` | nginx, api         | Handle incoming HTTP/S traffic   |
| `backend`  | api, db            | Database traffic — fully private |

The database is **never reachable from nginx** — it only lives on `backend`. This is network segmentation at zero cost.

---

## Useful Commands

```bash
# List all networks
docker network ls

# Inspect a network — see connected containers and their IPs
docker network inspect my-app-net

# Connect a running container to a network
docker network connect my-app-net my-container

# Disconnect a container from a network
docker network disconnect my-app-net my-container

# Remove a network (all containers must be disconnected first)
docker network rm my-app-net
```

---

## Bridge Network Options

```bash
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \     # Custom subnet
  --gateway 172.20.0.1 \       # Custom gateway
  --opt com.docker.network.bridge.name=my-bridge \  # Name the kernel bridge
  my-app-net
```

---

## Key Rules for Production

- **Never use the default `bridge` network** — it doesn't support DNS by container name
- **Always create named custom bridge networks** — enables service discovery by hostname
- **Segment your services** — frontend bridge, backend bridge; don't put everything on one
- **Don't publish ports you don't need** — only expose what faces the outside world
- **Use Docker Secrets** for sensitive env vars, not plain `environment:` blocks

---

## When to Use Bridge vs Other Drivers

| Need | Use |
|------|-----|
| Single host, multiple containers | ✅ Bridge |
| Multi-host / Docker Swarm | ❌ Use Overlay |
| Containers need host-level performance | ❌ Use Host driver |
| Full container isolation | ❌ Use None driver |