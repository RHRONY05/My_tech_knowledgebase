---
tags:
  - devops
  - docker
  - networking
  - infrastructure
last_reviewed: 2026-10-02
related_notes:
  - "[[dockerfile-and-layer-caching]]"
  - "[[docker-compose-production-patterns]]"
---

# Docker Core Architecture, Container Anatomy & Bridge Networking

> **Active Recall Self-Test:**
> 1. Why is a Docker container NOT a Virtual Machine (VM), and what two Linux kernel primitives (Namespaces vs. Cgroups) make containerization possible?
> 2. How does port mapping (`-p 5000:5000` Host:Container) punch a hole through the container NAT boundary?
> 3. Why does the default `bridge` network lack internal DNS resolution, and why must multi-container systems use a user-defined custom bridge network?
> 4. Inside a Docker container, why does pointing your database client to `localhost:5432` fail with connection refused?

---

## 1. The Core Problem
### The Flaws of Bare Metal "Works on My Machine" & Heavy VMs

#### Problem A: The Environment Drift Dilemma
Developers build applications on differing host operating systems (macOS, Windows, Ubuntu). Variations in installed Node.js patch versions, global npm binaries, system libraries (`glibc` vs `musl`), and environment variables cause code that runs locally to fail unpredictably when deployed to production.

#### Problem B: The Virtual Machine Overhead Trap
Before containers, teams used Virtual Machines (VMs) like VirtualBox or VMware to achieve isolation:
- **Fake Hardware Emulation:** A VM virtualizes an entire hardware computer (virtual CPU, RAM, BIOS, hard drive) and boots a full-blown guest OS (Windows/Ubuntu).
- **Extreme Resource Waste:** Running 3 simple microservices requires 3 guest operating systems, consuming gigabytes of RAM and taking minutes to boot.

#### Problem C: The Container Networking Paradox
Containers are isolated network sandboxes. If you run an Express backend in Container A and PostgreSQL in Container B:
- Neither container can see the other by default.
- Connecting to `localhost:5432` from Container A points to Container A itself, crashing because PostgreSQL is not inside Container A!
- Hardcoding private IPs (`172.17.0.2`) fails because container IPs are dynamic and change whenever containers reboot.

#### The Architectural Solution
1. **Linux Containers:** Isolated user-space processes running directly on the host kernel via **Namespaces** (virtual walls) and **Cgroups** (resource limits). Boots in milliseconds with near-zero overhead.
2. **User-Defined Bridge Networks:** Software-defined virtual switches that isolate inter-container traffic from host Wi-Fi and provide **Embedded DNS Service Discovery**, allowing containers to communicate using stable service names (e.g. `postgres:5432`).

---

## 2. The Mental Model

### VM vs. Container Architecture
```
      VIRTUAL MACHINE (Heavy)                       DOCKER CONTAINER (Lightweight)
┌─────────────────────────────────┐               ┌─────────────────────────────────┐
│ App 1 Code   │   App 2 Code     │               │ App 1 Process │  App 2 Process  │
├──────────────┼──────────────────┤               ├───────────────┼─────────────────┤
│ Guest OS 1   │   Guest OS 2     │               │ Namespaces & Cgroups Isolation   │
│ (2GB RAM)    │   (2GB RAM)      │               │ (Shares Host Kernel Directly)   │
├──────────────┴──────────────────┤               ├─────────────────────────────────┤
│ Hypervisor (Hardware Emulation) │               │ Docker Engine (Containerd)      │
├─────────────────────────────────┤               ├─────────────────────────────────┤
│ Physical Host OS & Hardware     │               │ Physical Host OS & Hardware     │
└─────────────────────────────────┘               └─────────────────────────────────┘
```

### Port Mapping (`-p Host:Container`)
```
 WINDOWS / LINUX HOST (Browser)                     CONTAINER
 ┌───────────────────────┐   Network Address    ┌───────────────────────┐
 │ http://localhost:5000 ├─────────────────────►│ Port 5000             │
 └───────────────────────┘   Translation (NAT)  │ (Express HTTP Server) │
                             (-p 5000:5000)     └───────────────────────┘
```

### Docker Network Topology & Internal DNS Resolution
```
┌────────────────────────────────────────────────────────────────────────┐
│ DOCKER ENGINE                                                          │
│                                                                        │
│  [ Custom Bridge: "movie_network" ] (Software Virtual Switch)          │
│         ├── Embedded DNS: "backend"  ──► Resolves to 172.20.0.3        │
│         └── Embedded DNS: "postgres" ──► Resolves to 172.20.0.4        │
│                                                                        │
│  ┌─────────────────────────┐             ┌───────────────────────────┐ │
│  │ Container: backend      │   TCP/IP    │ Container: postgres       │ │
│  │ DATABASE_URL=           ├────────────►│ Port 5432                 │ │
│  │ postgres://.../postgres │   DNS:      │ (Sealed from Internet;    │ │
│  │                         │   postgres  │  no public host ports!)   │ │
│  └─────────────────────────┘             └───────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Production Code Breakdown

### A. Essential Docker CLI Inspection Commands
```bash
# 1. Run a detached container with port mapping and memory limits (Cgroups)
docker run -d \
  --name movie_db \
  --memory="512m" \
  --cpus="1.0" \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# 2. Inspect active processes inside a container (Namespaces)
docker top movie_db

# 3. Stream real-time CPU/RAM resource usage across all containers
docker stats

# 4. Execute an interactive shell inside a running container
docker exec -it movie_db psql -U postgres

# 5. Inspect container logs with streaming follow
docker logs -f --tail 100 movie_db
```

### B. User-Defined Bridge Network Setup
```bash
# 1. Create an isolated custom bridge network
docker network create --driver bridge app_internal_network

# 2. Run the database container attached to the custom network (NO public ports!)
docker run -d \
  --name db_server \
  --network app_internal_network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# 3. Run the backend container on the same network
# Backend connects directly via service name 'db_server'
docker run -d \
  --name api_server \
  --network app_internal_network \
  -p 5000:5000 \
  -e DATABASE_URL="postgres://postgres:secret@db_server:5432/app_db" \
  my-backend-image:latest
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The `localhost` Loopback Trap Inside Containers
- **The Trap:** Setting `DATABASE_URL=postgres://user:pass@localhost:5432/db` inside a containerized backend.
- **The Bug:** Inside a container, `localhost` (127.0.0.1) refers strictly to **that specific container's private network namespace**. It does NOT refer to your laptop, and it does NOT refer to sibling containers.
- **The Fix:** Place both containers on the same user-defined bridge network and use the target container's name (e.g. `postgres:5432`) as the hostname.

### ⚠️ Gotcha 2: The Default `bridge` DNS Blindspot
- **The Trap:** Using `docker run` without `--network` attaches containers to the default `bridge` network.
- **The Crash:** On the default `bridge` network, Docker disables automatic DNS resolution. Containers cannot resolve names like `ping db_server`—they can only communicate via hardcoded IP addresses.
- **The Fix:** Always create and use a custom user-defined network:
  ```bash
  docker network create custom_network
  ```
  All user-defined bridge networks automatically enable Docker's embedded DNS server (listening at `127.0.0.11`).

### ⚠️ Gotcha 3: The Exposed Database Port Security Hazard
- **The Trap:** Mapping `-p 5432:5432` on your production database container in cloud VPS deployments.
- **The Consequence:** Docker automatically writes rules to the Linux `iptables` firewall, bypassing UFW and exposing PostgreSQL directly to the public internet where automated botnets brute-force passwords.
- **The Fix:** In production, **omit the `ports:` section entirely** on database containers. Only containers on the same private Docker network can reach port 5432; the host port remains closed to the outside world.
