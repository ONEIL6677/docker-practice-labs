# 🐳 Introduction to Docker

A quick guide to what Docker is and how it works.

---

## What is Docker?

Docker packages an application together with everything it needs to run —
code, libraries, and settings — into a single unit called a **container**.

That container runs the same way on any machine, solving the classic
*"it works on my machine"* problem.
---

## What about the Docker Engine?

The **Docker Engine** is the entire core, client-server software suite. It is the complete package you download and install. It acts as an umbrella that contains three distinct layers working together:

---

## 1. The Core Components

### A. The Docker Client
The **Docker Client** (`docker`) is the primary user interface. It is the command-line tool or desktop application where you type instructions.
* **Role:** Acts as the entry point for user interaction.
* **Function:** The client **does not** create images or run containers. It simply accepts your commands, packages them into standardized API requests, and forwards them down the chain.

### B. The Docker Daemon
The **Docker Daemon** (`dockerd`) is a persistent background service that runs on your host operating system.
* **Role:** Acts as the smart "middle manager" of the Docker Engine.
* **Function:** It listens for incoming requests from the Docker Client. It handles high-level logistics: authenticating users, building images from Dockerfiles, managing complex networks, and structuring data volumes.

### C. Containerd (The Container Runtime)
Deep inside the Docker Engine sits **containerd**, a specialized, CNCF-graduated container runtime process. 
* **Role:** Acts as the low-level "worker" that actually executes the containers.
* **Function:** When the Daemon receives a command to run a container, it hands the task off to containerd. Containerd manages the entire lifecycle of the container: creating namespaces for isolation, setting up control groups (cgroups) for resource limits, and supervising the running process.

---

## 2. How They Communicate

The components pass messages down the line using two highly efficient communication protocols:

1. **Client to Daemon (REST API):** The Docker Client communicates with the Daemon using standard HTTP REST API calls. This travels via local **UNIX Sockets** (`/var/run/docker.sock`) by default, but can use **TCP Sockets** for remote cloud servers.
2. **Daemon to Containerd (gRPC):** The Daemon communicates with containerd using **gRPC** (Google Remote Procedure Call), an ultra-fast, high-performance protocol designed for internal microservices.

---

## Lifecycle Example: Running a Container

When you type a command into your terminal, it triggers the following sequence of events:
---
1. **Input:** You execute `docker run my-app` in your terminal.
2. **Translation:** The **Docker Client** translates this into an HTTP REST API request.
3. **Logistics:** The **Docker Daemon** receives the request via the UNIX socket, verifies that the `my-app` image exists, and prepares the network configuration.
4. **Execution:** The Daemon sends a swift gRPC request to **containerd**, saying *"Launch this container process now."*
5. **Output:** Containerd interfaces with the host Linux/Windows kernel to create an isolated container environment and executes your application.

---

## Verifying the Architecture

You can see these distinct layers on your machine by running:

```bash
docker version
```

The output will separate the **Client** details from the **Server** engine details. 

If you are on a Linux machine, you can also view both background processes running independently using your system monitor:
```bash
ps aux | grep -E "dockerd|containerd"
```
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
