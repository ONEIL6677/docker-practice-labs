# Docker Overlay Networks

An **overlay network** spans **multiple Docker hosts**, allowing containers on different machines to communicate as if they were on the same local network. It's the backbone of **Docker Swarm** and multi-host deployments.

> Use overlay networks when your services are distributed across **multiple nodes** — production clusters, high availability setups, or horizontally scaled microservices.

---

## How It Works

```
Node 1 (Host A)                      Node 2 (Host B)
┌──────────────────────┐             ┌──────────────────────┐
│  [api replica 1]  ──┐│             │┌── [api replica 2]   │
│                     ││  VXLAN      ││                      │
│  [db]  ─────────── [overlay network tunnel] ──────────────│
│                     ││  (encrypted)││                      │
└──────────────────────┘             └──────────────────────┘
          │                                      │
          └──────────── Swarm Manager ───────────┘
```

- Docker wraps traffic in **VXLAN packets** and routes them across hosts
- Containers use each other's **service names** — Docker handles the routing
- Traffic can be **encrypted at rest** with one flag
- Requires a **key-value store** — Docker Swarm has this built in

---

## Prerequisites

```bash
# Initialize Swarm on the manager node
docker swarm init --advertise-addr <MANAGER_IP>

# The output gives you a token — run this on each worker node:
docker swarm join --token <TOKEN> <MANAGER_IP>:2377

# Verify your cluster
docker node ls
```

---

## Quick Start

```bash
# Create an overlay network (run on manager node)
docker network create \
  --driver overlay \
  --attachable \             # Allows standalone containers to join (useful for debugging)
  my-overlay-net

# Deploy a service on the overlay
docker service create \
  --name api \
  --network my-overlay-net \
  --replicas 3 \             # 3 replicas spread across nodes
  my-api-image
```

---

## Production Example — Microservices on Docker Swarm

```yaml
# docker-stack.yml  (deployed with: docker stack deploy -c docker-stack.yml prod)
version: "3.9"

services:

  nginx:
    image: nginx:alpine
    ports:
      - target: 443
        published: 443
        mode: ingress          # Swarm load-balances across all nginx replicas
    networks:
      - public-net
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == worker

  api:
    image: my-api:latest
    environment:
      DB_HOST: db              # Docker DNS resolves across nodes automatically
      REDIS_HOST: cache
    networks:
      - public-net             # Reachable by nginx
      - private-net            # Reachable by db and cache
    deploy:
      replicas: 4              # Spread across worker nodes
      update_config:
        parallelism: 2         # Roll out 2 at a time — zero downtime deploys
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - private-net            # Isolated — never exposed to public-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db == true   # Pin db to a specific labeled node
    secrets:
      - db_pass

  cache:
    image: redis:7-alpine
    networks:
      - private-net
    deploy:
      replicas: 1

networks:
  public-net:
    driver: overlay
    driver_opts:
      encrypted: "true"        # Encrypt VXLAN traffic between nodes

  private-net:
    driver: overlay
    driver_opts:
      encrypted: "true"        # DB and cache traffic always encrypted

volumes:
  pg_data:
    driver: local

secrets:
  db_pass:
    external: true
```

### Deploy the stack

```bash
# Deploy (run from manager node)
docker stack deploy -c docker-stack.yml prod

# Check services are running
docker stack services prod

# Check where replicas landed across nodes
docker service ps prod_api
```

---

## Encryption

```bash
# Enable encryption when creating the network
docker network create \
  --driver overlay \
  --opt encrypted \            # Encrypts data-plane traffic with AES-GCM
  secure-net

# Or in compose/stack files:
# driver_opts:
#   encrypted: "true"
```

> Without `encrypted`, overlay traffic between nodes is **unencrypted** inside the VXLAN tunnel. Always enable it in production.

---

## Useful Commands

> List networks (overlay networks show scope: swarm)
```bash
docker network ls
```

> Inspect overlay — see which nodes and containers are connected
```bash
docker network inspect my-overlay-net
```
> Scale a service up (Swarm handles spreading replicas)
```bash
docker service scale prod_api=6
```
> Rolling update with zero downtime
```bash
docker service update --image my-api:v2 prod_api
```
> Remove the whole stack
```bash
docker stack rm prod
```

## Label Nodes for Placement
>Tag a node so stateful services (db, volumes) land on specific machines
```bash
docker node update --label-add db=true node-hostname
```

> Then in your stack file use placement constraints:
> deploy:
>   placement:
>     constraints:
>       - node.labels.db == true

## Key Rules for Production

- **Always enable encryption** — overlay traffic crosses real network links between hosts
- **Use placement constraints** for stateful services (databases, persistent volumes)
- **Use `ingress` mode for published ports** — Swarm routes traffic to any node automatically
- **Separate public and private networks** — same segmentation principle as bridge, across hosts
- **Never use `--attachable` in production** unless you explicitly need standalone containers to join
- **Monitor with `docker service ps`** — check where replicas land and if any are failing

---

## Bridge vs Overlay — When to Use What

| Need | Use |
|------|-----|
| Single host | Bridge |
| Multiple hosts / Swarm cluster | Overlay |
| Encrypted inter-node traffic | Overlay + `encrypted` |
| Local dev / docker-compose | Bridge |
| HA, rolling deploys, scaling | Overlay |