# 🐳 Introduction to Docker

A quick guide to what Docker is and how it works.

---

## What is Docker?

Docker packages an application together with everything it needs to run —
code, libraries, and settings — into a single unit called a **container**.

That container runs the same way on any machine, solving the classic
*"it works on my machine"* problem.
---

## 1. The Docker Client
The **Docker Client** (`docker`) is the primary user interface. It is the command-line tool or desktop application where you type instructions.

* **Role:** Acts as the entry point for user interaction.
* **Function:** The client **does not** create images or run containers. Instead, it accepts your commands, packages them into standardized API requests, and forwards them to the daemon.
---

## 2. The Docker Daemon
The **Docker Daemon** (`dockerd`) is a persistent background service that runs on your host operating system.

* **Role:** Acts as the brain and engine of your Docker environment.
* **Function:** It listens for incoming requests from the Docker Client. It is entirely responsible for the heavy lifting: building images, spinning up containers, managing local networks, and attaching storage volumes.
---

## Overview

Docker operates on a **client-server architecture**. Instead of a single monolithic program, the work is split between the tool you interact with (the Client) and the background engine that executes the commands (the Daemon).
---
## Containers vs Virtual Machines

| | Virtual Machines | Containers |
|---|---|---|
| Virtualizes | A full computer + OS | Just the app |
| Size | Large (GBs) | Small (MBs) |
| Startup | Minutes | Seconds |
| Isolation | Full OS-level | Process-level, shares host OS |

---

## Core Concepts

- **Image** — a read-only template/blueprint for a container
- **Container** — a running instance of an image
- **Dockerfile** — instructions for building an image
- **Registry** — where images are stored (e.g. Docker Hub)
- **Docker Engine** — the software that builds and runs containers

---

## How It Works (Docker architecture)

```
Dockerfile → docker build → Image → docker run → Container
```

---

## Basic Commands

```bash
docker pull nginx          # download an image from registry(docker hub)
docker images               # list images
docker run -d nginx         # run a container in the background
docker ps                   # list running containers
docker stop <container_id>  # stop a container
docker logs <container_id>  # view container logs
docker exec -it <container_id> /bin/bash   # open a shell inside it
```

---

## Simple Dockerfile Example

```dockerfile
# 1. Download the official, lightweight Python 3.11 base operating system image
FROM python:3.11-slim
# 2. Create and switch to a folder named '/app' inside the container for all next steps
WORKDIR /app
# 3. Copy only the dependency list from your local computer into the container's current folder
COPY requirements.txt .
# 4. Run the pip tool inside the container to install all listed software packages
RUN pip install -r requirements.txt
# 5. Copy all the remaining source code files from your local computer into the container
COPY . .
# 6. Inform Docker that the application inside the container listens on network port 5000
EXPOSE 5000
# 7. Define the default command to run your app automatically when the container starts up
CMD ["python", "app.py"]

```

## Run this command in the terminal
```bash
docker build -t my-python-app . # -t is to tag the image or give it a name
docker run -p 5000:5000 my-python-app # maping port 5000 in the container with port 5000 in the host computer
```


---
⭐ *If these 7 labs helped you understand Docker, please give this repository a star!*
