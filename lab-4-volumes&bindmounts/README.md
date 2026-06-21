# Docker Volumes & Bind Mounts — Practice Guide

A hands-on README for learning how Docker persists and shares data using **volumes** and **bind mounts**. Each section includes a short explanation, the commands to run, and a Dockerfile you can build and test.

---

## 1. The Core Concept

Containers are ephemeral when you remove one, everything written inside its writable layer disappears. To keep data around (or share it with the host), Docker gives you three storage options:

| Type | Managed by | Stored at | Best for |
|---|---|---|---|
| **Volume** | Docker | `/var/lib/docker/volumes/...` | Persistent app data, databases, sharing data between containers |
| **Bind mount** | You (the host filesystem) | Any path you choose on the host | Local development, injecting config files, live code reload |
| **tmpfs mount** | Docker (RAM only) | Memory, never written to disk | Temporary, sensitive data (not covered in depth here) |

**Rule of thumb:** use **volumes** for data the *container* owns (databases, uploads). Use **bind mounts** for data the *host* owns and the container just needs to see (your source code, a config file).

---

## 2. Volumes

### 2.1 Why volumes?
- Created and managed entirely by Docker (`docker volume` commands).
- Survive container removal.
- Can be shared safely across multiple containers.
- Work the same way on Linux, Mac, and Windows (bind mounts can behave differently across OSes).
- Easier to back up, migrate, or move to remote/cloud storage drivers.

### 2.2 Basic volume commands

```bash
# Create a named volume
docker volume create app_data

# List volumes
docker volume ls

# Inspect a volume (shows the real path on the host)
docker volume inspect app_data

# Remove a volume
docker volume rm app_data

# Remove all unused volumes
docker volume prune
```

### 2.3 Practice Example A — Named volume for persistent counter

This example writes a counter to a file each time the container runs. Without a volume, the count resets every time. With a volume, it persists.

**`example-a/Dockerfile`**
```dockerfile
FROM alpine:3.20

WORKDIR /app

# A tiny script that reads, increments, and writes a counter
RUN echo '#!/bin/sh' > counter.sh && \
    echo 'FILE=/data/count.txt' >> counter.sh && \
    echo '[ -f "$FILE" ] || echo 0 > "$FILE"' >> counter.sh && \
    echo 'COUNT=$(cat "$FILE")' >> counter.sh && \
    echo 'COUNT=$((COUNT + 1))' >> counter.sh && \
    echo 'echo "$COUNT" > "$FILE"' >> counter.sh && \
    echo 'echo "This container has run $COUNT time(s)."' >> counter.sh && \
    chmod +x counter.sh

CMD ["./counter.sh"]
```

**Build and run:**
```bash
cd example-a
docker build -t volume-counter .

# Run WITHOUT a volume — count always shows 1
docker run --rm volume-counter
docker run --rm volume-counter
docker run --rm volume-counter
# Output: "This container has run 1 time(s)." every time

# Run WITH a named volume — count increases each run
docker run --rm -v counter_data:/data volume-counter
docker run --rm -v counter_data:/data volume-counter
docker run --rm -v counter_data:/data volume-counter
# Output: 1, 2, 3 — the data in /data persisted!
```

**Try it yourself:** inspect the volume with `docker volume inspect counter_data`, then look at the `count.txt` file on the host at the `Mountpoint` path it shows.

---

## 3. Bind Mounts

### 3.1 Why bind mounts?
- You choose the exact host directory; you can edit files from your IDE and see changes instantly in the container.
- Great for local development: edit code on the host, container picks it up without rebuilding.
- Useful for injecting a single config file (e.g. `nginx.conf`) into a container.
- Downside: tightly couples the container to your host's file layout, and permissions/paths can differ between OSes.

### 3.2 Basic bind mount syntax

```bash
# Long form (recommended, more readable)
docker run --mount type=bind,source="$(pwd)"/src,target=/app/src myimage

# Short form
docker run -v "$(pwd)"/src:/app/src myimage

# Read-only bind mount (container can read but not write)
docker run -v "$(pwd)"/config:/app/config:ro myimage
```

### 3.3 Practice Example B — Live-reloading dev server with a bind mount

This simulates the classic "edit on host, see changes in container" workflow using a simple Python HTTP server (no extra dependencies needed).

**`example-b/Dockerfile`**
```dockerfile
FROM python:3.12-alpine

WORKDIR /app

# We intentionally do NOT copy site files here —
# they will be supplied at runtime via a bind mount.
EXPOSE 8000

CMD ["python3", "-m", "http.server", "8000"]
```

**`example-b/site/index.html`** (create this file yourself to start)
```html
<!DOCTYPE html>
<html>
  <body>
    <h1>Hello from the host filesystem!</h1>
  </body>
</html>
```

**Build and run:**
```bash
cd example-b
docker build -t dev-server .

# Bind-mount the local "site" folder into the container's /app
docker run --rm -p 8000:8000 -v "$(pwd)"/site:/app dev-server
```

Visit `http://localhost:8000`. Now edit `site/index.html` on your host, save, and refresh the browser — no rebuild needed, because the container is reading directly from your host directory.

**Try it yourself:** add a `:ro` to the mount and try editing a file from *inside* the container (`docker exec`) — it should fail since the mount is read-only.

---

## 4. Volumes vs. Bind Mounts — Side-by-Side Practice

### 4.1 Practice Example C — Same app, two storage strategies

A minimal Node.js app that logs visits to a file. We'll run it twice: once with a bind mount, once with a volume, so you can directly compare behavior.

**`example-c/Dockerfile`**
```dockerfile
FROM node:20-alpine

WORKDIR /app

RUN echo "const fs = require('fs');" > server.js && \
    echo "const path = '/logs/visits.log';" >> server.js && \
    echo "const line = new Date().toISOString() + ' - visit\\n';" >> server.js && \
    echo "fs.appendFileSync(path, line);" >> server.js && \
    echo "console.log(fs.readFileSync(path, 'utf8'));" >> server.js

CMD ["node", "server.js"]
```

**Run with a bind mount (logs land in a folder you control):**
```bash
cd example-c
docker build -t visit-logger .
mkdir -p logs

docker run --rm -v "$(pwd)"/logs:/logs visit-logger
docker run --rm -v "$(pwd)"/logs:/logs visit-logger
# Check ./logs/visits.log on your host directly — it's right there.
cat logs/visits.log
```

**Run with a named volume (Docker manages the storage):**
```bash
docker run --rm -v visit_logs:/logs visit-logger
docker run --rm -v visit_logs:/logs visit-logger
# You can't casually browse this from a normal file explorer —
# it lives inside Docker's storage area.
docker volume inspect visit_logs
```

**Key takeaway:** identical app code, two very different storage behaviors, controlled entirely by the `-v` flag at runtime — nothing in the Dockerfile changes.

---

## 5. Putting It Together with Docker Compose

Compose makes long-running combinations of volumes and bind mounts easier to manage than typing long `docker run` commands.

### 5.1 Practice Example D — Database with a volume + app with a bind mount

**`example-d/Dockerfile`**
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
EXPOSE 5000
CMD ["python3", "-m", "http.server", "5000"]
```

**`example-d/docker-compose.yml`**
```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      # Bind mount: live code from host
      - ./site:/app

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      # Named volume: Postgres manages and persists its own data
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

**Run it:**
```bash
cd example-d
mkdir -p site
echo "<h1>Compose-powered bind mount</h1>" > site/index.html

docker compose up -d
docker compose ps
docker volume ls   # notice "example-d_db_data" was created automatically

# Tear down containers but KEEP the volume (data survives)
docker compose down

# Tear down and DELETE the volume too
docker compose down -v
```

**Try it yourself:** stop and restart the `db` service a few times, then check that Postgres's data directory (the named volume) still has your data, while editing `site/index.html` updates `web` instantly without a rebuild.

---

## 6. Quick Reference Cheat Sheet

```bash
# VOLUMES
docker volume create myvol
docker volume ls
docker volume inspect myvol
docker volume rm myvol
docker volume prune
docker run -v myvol:/path/in/container myimage

# BIND MOUNTS
docker run -v /host/path:/container/path myimage
docker run --mount type=bind,source=/host/path,target=/container/path myimage
docker run -v /host/path:/container/path:ro myimage   # read-only

# INSPECT WHAT'S MOUNTED ON A RUNNING CONTAINER
docker inspect <container_id> --format '{{ json .Mounts }}'

# CLEAN UP EVERYTHING UNUSED (be careful!)
docker system prune --volumes
```

---

## 7. Common Pitfalls to Practice Spotting

1. **Forgetting `-v` on rerun** — data from a volume only persists if you re-attach the *same* volume name next time.
2. **Bind mount path typos** — Docker will silently create an empty directory at the host path if it doesn't exist, which can mask a typo (you'll see an empty mount instead of an error).
3. **Anonymous volumes building up** — running `-v /container/path` (no host source, no name) creates an anonymous volume each time; these pile up if not pruned.
4. **Permission mismatches** — bind-mounted files keep the host's UID/GID, which can cause "permission denied" inside containers running as a different user. Try this deliberately in Example B by changing file ownership on the host.
5. **`docker compose down -v` deletes data** — easy to do by accident in practice; always double-check before adding `-v` if you want to keep your data.

---

## 8. Suggested Practice Order

1. Run Example A without a volume, then with one — observe the difference.
2. Run Example B and edit files live to feel how bind mounts behave.
3. Run Example C with both strategies back to back.
4. Bring it all together with Example D and Compose.
5. Intentionally trigger each pitfall in Section 7 to recognize the symptoms.

Happy practicing!