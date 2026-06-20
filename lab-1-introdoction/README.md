# Introduction to Docker

A quick guide to what Docker is and how it works.

---

## What is Docker?

Docker packages an application together with everything it needs to run —
code, libraries, and settings — into a single unit called a **container**.

That container runs the same way on any machine, solving the classic
*"it works on my machine"* problem.
---

## Docker Engine

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


# Part 2: Dockerfile Components Explained in Simple Terms

Think of a **Dockerfile** as a **recipe** for baking a digital cake (your application container). The Docker engine reads this recipe line by line, from top to bottom, to build your final image.

---

### 1. FROM — The Base Ingredient
* **What it means:** "Start with this foundation."
* **Analogy:** Choosing whether you are baking a chocolate cake or a vanilla cake. You must always start with a base.
* **Simple Example:** `FROM python:3.11` (Starts your project with Python already installed and ready to go).

### 2. WORKDIR — The Mixing Bowl
* **What it means:** "Create a folder inside the container and move into it."
* **Analogy:** Setting up your clean workspace or mixing bowl on the kitchen counter before you start adding other ingredients.
* **Simple Example:** `WORKDIR /app` (Every command written after this line will happen inside the `/app` folder).

### 3. COPY — Adding Local Ingredients
* **What it means:** "Take files from my computer and put them inside the container."
* **Analogy:** Grabbing flour and sugar from your home pantry and dumping them into the mixing bowl.
* **Simple Example:** `COPY . .` (The first dot means "everything in my current computer folder," and the second dot means "put it into the container's current working directory").

### 4. RUN — The Preparation Step
* **What it means:** "Run a command to install tools or configure things WHILE BUILDING the image."
* **Analogy:** Chopping nuts or melting butter before the cake goes into the oven. It changes the ingredients permanently.
* **Simple Example:** `RUN pip install requests` (Installs a Python library inside the image during the build phase).

### 5. EXPOSE — Opening a Window
* **What it means:** "Note that this application listens on a specific network port."
* **Analogy:** Telling your guests, "Hey, when the food is ready, I will be serving it out of the kitchen window on port 8080."
* **Simple Example:** `EXPOSE 8080` (Tells Docker that traffic will come in through port 8080).

### 6. ENV — Setting the Room Temperature
* **What it means:** "Create a global variable/setting that the application can read."
* **Analogy:** Setting a kitchen timer or adjusting the thermostat so everyone knows the environment rules.
* **Simple Example:** `ENV PORT=8080` (Saves a setting named `PORT` so your application code can use it later).

### 7. CMD — Turning on the Oven
* **What it means:** "This is the very final command to run ONLY WHEN THE CONTAINER STARTS UP."
* **Analogy:** Actually turning on the oven to bake. This only happens *after* the recipe is fully built and you are ready to eat.
* **Simple Example:** `CMD ["python", "main.py"]` (Starts your Python application when someone runs your container).

---

### Full Recipe Example

Here is how they all look put together in a standard setup:

```dockerfile
# 1. Start with the base runtime
FROM node:20

# 2. Make a workspace folder
WORKDIR /my_project

# 3. Copy our code into that workspace
COPY . .

# 4. Install the necessary project tools
RUN npm install

# 5. Tell Docker which port we use
EXPOSE 3000

# 6. Start the app when the container turns on
CMD ["node", "server.js"]
```

```
## Author: ONEIL KIMBI
---
⭐ *If these 7 labs helped you understand Docker, please give this repository a star!*
