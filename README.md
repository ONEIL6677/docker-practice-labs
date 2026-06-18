# 🐳 Docker Practice Labs

Welcome to the **Docker Practice Labs** repository! This is a curated, step-by-step learning environment featuring **7 hands-on labs** designed to take you from a Docker beginner to confidently managing multi-container production environments.

Instead of just reading theory, you will build, break, debug, and optimize real-world container configurations.

## 🚀 The 7-Lab Roadmap

Each lab lives in its own directory and includes a dedicated guide and challenge.

*   **Lab 1: Docker CLI Basics** – Spin up, stop, inspect, and manage basic container lifecycles (Nginx, Alpine).
*   **Lab 2: Custom Dockerfiles** – Write your first build instructions and containerize a basic web application.
*   **Lab 3: Multi-Stage Builds** – Optimize image sizes and security by separating build and runtime environments.
*   **Lab 4: Container Networking** – Connect isolated containers using custom bridge networks and DNS discovery.
*   **Lab 5: Persistent Storage** – Master named volumes and bind mounts to keep database data safe across restarts.
*   **Lab 6: Docker Compose** – Define and launch multi-container application stacks with a single command.
*   **Lab 7: Debugging & Troubleshooting** – Inspect container logs, use `exec` commands, and fix broken entrypoints.

## 📂 Repository Structure

```text
├── 01-cli-basics/          # Lab 1: Essential container commands
├── 02-custom-dockerfiles/   # Lab 2: Building custom images
├── 03-multi-stage-builds/  # Lab 3: Production image optimization
├── 04-container-networks/  # Lab 4: Custom networks and linking
├── 05-persistent-storage/  # Lab 5: Volumes and bind mounts
├── 06-docker-compose/      # Lab 6: Multi-container orchestration
└── 07-troubleshooting/     # Lab 7: Reading logs and debugging
```

## 🛠️ Prerequisites

Before you start, make sure you have installed:
*   [Docker Desktop](https://docker.com) or Docker Engine
*   [Git](https://git-scm.com)

## 🏃 How to Start

1. **Clone this repository:**
   ```bash
   git clone https://github.com
   cd YOUR-REPO-NAME
   ```
2. **Choose a lab:**
   Navigate into any lab folder (e.g., `cd 01-cli-basics`) and follow the instructions in its local `README.md`.

---
⭐ *If these 7 labs helped you understand Docker, please give this repository a star!*
