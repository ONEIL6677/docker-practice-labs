# Docker Networking — Practice Guide

A hands-on README for learning how Docker containers communicate with each other,
with the host, and with the outside world. Each section includes a clear explanation,
the commands to run, and Dockerfiles or Compose files to practice with.

---

## 1. The Core Concept

Every Docker container gets its own isolated network namespace by default — it has
its own network interfaces, IP address, and routing table. Docker then provides
**network drivers** that control how containers connect to each other and to the
outside world.

| Driver | What it does | Best for |
|---|---|---|
| **bridge** | Default. Creates a private internal network; containers on the same bridge can talk to each other by IP. | Most single-host use cases |
| **host** | Container shares the host's network stack directly — no isolation. | Performance-sensitive apps, avoiding port mapping |
| **none** | No networking at all. | Fully isolated, security-sensitive containers |
| **overlay** | Multi-host networking across a Docker Swarm cluster. | Swarm / distributed deployments |
| **macvlan** | Gives a container its own MAC address, making it appear as a physical device on the LAN. | Legacy apps that need to be on the LAN directly |

This guide focuses on **bridge** and **host**, which cover 95 % of everyday use.

---

## 2. The Default Bridge Network

When you start a container without specifying a network, Docker attaches it to a
built-in bridge called `bridge` (also shown as `docker0` on the host).

```bash
# Inspect the default bridge network
docker network inspect bridge

# Run two containers on the default bridge
docker run -d --name box1 alpine sleep 300
docker run -d --name box2 alpine sleep 300

# Try to ping box1 FROM box2 by name — this FAILS on the default bridge
docker exec box2 ping -c 2 box1
# Error: bad address 'box1'

# But pinging by IP works (get box1's IP first)
docker inspect box1 --format '{{.NetworkSettings.IPAddress}}'
# e.g. 172.17.0.2
docker exec box2 ping -c 2 172.17.0.2  # works, but fragile
```

**Key takeaway:** the default bridge does NOT support DNS-based name resolution
between containers. That is why you should almost always use a **user-defined
bridge** instead.

---

## 3. User-Defined Bridge Networks

User-defined bridges give you automatic DNS: containers on the same custom network
can reach each other by **container name** (or service name in Compose).

### 3.1 Network commands

```bash
# Create a user-defined bridge network
docker network create my_net

# List all networks
docker network ls

# Inspect a network
docker network inspect my_net

# Connect a running container to an additional network
docker network connect my_net <container>

# Disconnect a container from a network
docker network disconnect my_net <container>

# Remove a network (all containers must be disconnected first)
docker network rm my_net
```

### 3.2 Practice Example A — Two containers talking by name

**Goal:** run a server container and a client container; have the client reach the
server by name, not by IP.

**`example-a/server/Dockerfile`**
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
# A tiny HTTP server that just says hello
RUN echo "from http.server import BaseHTTPRequestHandler, HTTPServer" > server.py && \
    echo "class H(BaseHTTPRequestHandler):" >> server.py && \
    echo "    def do_GET(self):" >> server.py && \
    echo "        self.send_response(200)" >> server.py && \
    echo "        self.end_headers()" >> server.py && \
    echo "        self.wfile.write(b'Hello from the server container!')" >> server.py && \
    echo "    def log_message(self, *a): pass" >> server.py && \
    echo "HTTPServer(('', 8080), H).serve_forever()" >> server.py
CMD ["python3", "server.py"]
```

**Build and run:**
```bash
# Create a custom network
docker network create app_net

# Build and start the server on that network
cd example-a/server
docker build -t demo-server .
docker run -d --name server --network app_net demo-server

# Run a client container on the same network
# Notice: we reach the server by the NAME "server", not by an IP
docker run --rm --network app_net alpine \
  wget -qO- http://server:8080
# Output: Hello from the server container!

# Prove that a container on a DIFFERENT network cannot reach it
docker run --rm alpine wget -qO- --timeout=3 http://server:8080
# Fails — no route to host

# Clean up
docker stop server && docker rm server
docker network rm app_net
```

---

## 4. Port Publishing — Exposing Containers to the Host

Containers are isolated from the host by default. `-p` (or `--publish`) punches a
hole through that isolation and maps a host port to a container port.

```bash
# Map host port 8080 → container port 80
docker run -p 8080:80 nginx

# Map on a specific host interface only (more secure)
docker run -p 127.0.0.1:8080:80 nginx

# Let Docker pick a random available host port
docker run -p 80 nginx
docker port <container>   # find out what port was chosen

# Map multiple ports at once
docker run -p 8080:80 -p 4430:443 nginx
```

### 4.1 Practice Example B — Published port vs. inter-container networking

**`example-b/Dockerfile`**
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
RUN echo "from http.server import SimpleHTTPRequestHandler, HTTPServer" > app.py && \
    echo "import os" >> app.py && \
    echo "os.makedirs('static', exist_ok=True)" >> app.py && \
    echo "open('static/index.html','w').write('<h1>Docker Networking Practice</h1>')" >> app.py && \
    echo "import os; os.chdir('static')" >> app.py && \
    echo "HTTPServer(('', 8000), SimpleHTTPRequestHandler).serve_forever()" >> app.py
CMD ["python3", "app.py"]
EXPOSE 8000
```

```bash
cd example-b
docker build -t net-demo .

# Published — accessible from your browser on the host
docker run -d --name web -p 8000:8000 net-demo
curl http://localhost:8000

# NOT published — accessible only from containers on the same network
docker network create internal_net
docker run -d --name web2 --network internal_net net-demo
# This should FAIL (no port published to host):
curl http://localhost:8000
# This should SUCCEED (same network, DNS works):
docker run --rm --network internal_net alpine wget -qO- http://web2:8000

docker stop web web2 && docker rm web web2
docker network rm internal_net
```

---

## 5. The `host` Network Driver

With `--network host` the container skips Docker's network stack entirely and
shares the host's interfaces directly. No port mapping needed — a process listening
on port 8000 inside the container *is* listening on port 8000 on the host.

```bash
# Container binds directly to the host's port 8000 — no -p flag needed
docker run --rm --network host python:3.12-alpine \
  python3 -c "
from http.server import SimpleHTTPRequestHandler, HTTPServer
HTTPServer(('',8000), SimpleHTTPRequestHandler).serve_forever()
"
# On another terminal:
curl http://localhost:8000
```

**When to use it:** high-throughput or latency-sensitive workloads where Docker's
NAT overhead matters. Otherwise, prefer bridge networks for isolation.

**Note:** `--network host` is ignored on Docker Desktop for Mac and Windows (the
Linux VM intercepts it). It only behaves as expected on Linux hosts.

---

## 6. Container-to-Container Networking with Docker Compose

Compose automatically creates a **user-defined bridge** for each project and puts
all services on it. Services reach each other using the **service name** as the
hostname — no manual `docker network create` needed.

### 6.1 Practice Example C — Frontend, backend, and database

**`example-c/docker-compose.yml`**
```yaml
services:

  frontend:
    image: nginx:alpine
    ports:
      - "8080:80"         # only the frontend is exposed to the host
    networks:
      - frontend_net

  backend:
    build: ./backend
    networks:
      - frontend_net      # frontend can reach it here
      - backend_net       # and it can reach the database here
    # NOT published to the host — internal only

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend_net       # only the backend can reach the db
    # NOT published to the host

networks:
  frontend_net:
  backend_net:
```

**`example-c/backend/Dockerfile`**
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
RUN echo "from http.server import BaseHTTPRequestHandler, HTTPServer" > api.py && \
    echo "class H(BaseHTTPRequestHandler):" >> api.py && \
    echo "    def do_GET(self):" >> api.py && \
    echo "        self.send_response(200)" >> api.py && \
    echo "        self.end_headers()" >> api.py && \
    echo "        self.wfile.write(b'API response from backend')" >> api.py && \
    echo "    def log_message(self, *a): pass" >> api.py && \
    echo "HTTPServer(('', 5000), H).serve_forever()" >> api.py
CMD ["python3", "api.py"]
```

```bash
cd example-c
docker compose up -d

# frontend is reachable from the host
curl http://localhost:8080

# backend is NOT reachable from the host (no published port)
curl http://localhost:5000   # connection refused

# but the frontend container CAN reach the backend by service name
docker compose exec frontend wget -qO- http://backend:5000

# the backend CAN reach the db by service name
docker compose exec backend ping -c 2 db

# the frontend CANNOT reach the db (different network segment)
docker compose exec frontend ping -c 2 db  # fails

docker compose down
```

**This is the recommended production pattern:** network segmentation using multiple
Compose networks keeps your database reachable only by the services that need it.

---

## 7. Network Aliases

A container can have additional DNS names on a network using `--network-alias`.
Useful for blue/green deployments or when you want multiple containers answering
to the same hostname.

```bash
docker network create alias_net

# Two containers, both aliased to "api"
docker run -d --network alias_net --network-alias api --name api_v1 \
  nginx:alpine
docker run -d --network alias_net --network-alias api --name api_v2 \
  nginx:alpine

# A client resolving "api" will round-robin between both containers
docker run --rm --network alias_net alpine sh -c \
  "for i in 1 2 3 4; do wget -qO- http://api | head -1; done"

docker stop api_v1 api_v2 && docker rm api_v1 api_v2
docker network rm alias_net
```

In Docker Compose, aliases are set per service:

```yaml
services:
  app:
    image: myapp
    networks:
      mynet:
        aliases:
          - api
          - app.internal
```

---

## 8. DNS Resolution Inside Containers

Docker runs an embedded DNS server at `127.0.0.11` for all user-defined networks.
You can inspect and test it from inside a container.

```bash
docker network create dns_test

docker run -d --name svc --network dns_test nginx:alpine

docker run --rm --network dns_test alpine sh -c "
  echo '=== /etc/resolv.conf ==='
  cat /etc/resolv.conf

  echo '=== nslookup svc ==='
  nslookup svc

  echo '=== HTTP via name ==='
  wget -qO- http://svc | head -3
"

docker stop svc && docker rm svc
docker network rm dns_test
```

The `nameserver 127.0.0.11` line in `resolv.conf` is Docker's internal DNS —
it resolves container and service names automatically on user-defined networks.

---

## 9. Quick Reference Cheat Sheet

```bash
# Networks
docker network ls                          # list all networks
docker network create <name>               # create user-defined bridge
docker network create --driver host <name> # create with specific driver
docker network inspect <name>              # detailed info (IPs, containers)
docker network rm <name>                   # remove
docker network prune                       # remove all unused networks

# Connecting containers
docker run --network <name> ...            # attach at start
docker network connect <name> <container>  # attach a running container
docker network disconnect <name> <container>

# Port publishing
docker run -p <host_port>:<container_port> ...    # publish a port
docker run -p 127.0.0.1:<host_port>:<cport> ...  # bind to localhost only
docker port <container>                            # show published ports

# Inspect networking on a running container
docker inspect <container> --format '{{ json .NetworkSettings.Networks }}'
docker inspect <container> --format '{{ .NetworkSettings.IPAddress }}'

# Test connectivity from inside a container
docker exec <container> ping <other_container>
docker exec <container> wget -qO- http://<service>:<port>
docker exec <container> cat /etc/resolv.conf
```

---

## 10. Common Pitfalls to Practice Spotting

1. **Using the default bridge and expecting DNS** — container name resolution only
   works on user-defined networks. Always create your own bridge.

2. **Publishing a port that is already in use on the host** — Docker will throw a
   bind error. Use `lsof -i :<port>` or `ss -tlnp` to check what's listening.

3. **Publishing a database port to `0.0.0.0`** — accidentally exposing Postgres or
   Redis to the whole network. Bind to `127.0.0.1` if only the host needs access,
   or don't publish the port at all if only containers need it.

4. **Forgetting `docker network rm` after practice** — unused networks pile up.
   Run `docker network prune` occasionally.

5. **Relying on container IP addresses instead of names** — IPs can change on
   container restart; always use service/container names.

6. **`--network host` on Docker Desktop** — silently ignored on Mac and Windows;
   it only works on native Linux. Tests that pass on Linux may break on a
   developer's Mac for this reason.

---

## 11. Suggested Practice Order

1. Reproduce the default-bridge DNS failure (Section 2), then fix it by switching
   to a user-defined bridge (Section 3) — that contrast is the most important lesson.
2. Play with port publishing in Section 4: publish, don't publish, bind to localhost only.
3. Stand up the three-tier Compose stack in Section 6 and verify which containers
   can and cannot talk to each other.
4. Experiment with network aliases in Section 7 to see round-robin DNS in action.
5. Run the DNS inspection script in Section 8 to see Docker's internal resolver.
6. Intentionally trigger each pitfall in Section 10.

Happy practicing!