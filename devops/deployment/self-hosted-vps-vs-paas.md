---
tags:
  - devops
  - deployment
  - cloud
  - linux
  - vps
last_reviewed: 2026-10-02
related_notes:
  - "[[docker-compose-production-patterns]]"
  - "[[nginx-reverse-proxy-and-spa-routing]]"
  - "[[ssl-tls-and-certbot]]"
---

# Production Cloud Deployment: Self-Hosted Linux VPS vs. Managed PaaS

> **Active Recall Self-Test:**
> 1. Why does a 1 GB RAM Linux VPS crash during `docker compose build`, and how does allocating a **2 GB Swap file with `vm.swappiness=10`** prevent Out-Of-Memory (OOM) kernel panics?
> 2. What are the key architectural trade-offs between a **Self-Hosted VPS** (All-in-One Docker network) and a **Decoupled Managed PaaS** (Vercel + Render + Neon)?
> 3. How does Uncomplicated Firewall (UFW) enforce the security perimeter on a production VPS, and why must database port 5432 remain closed?
> 4. Why does Windows reject SSH private keys until permissions are stripped using `icacls`, and what is the Linux equivalent (`chmod 600`)?

---

## 1. The Core Problem
### The Dilemma of Infrastructure Ownership vs. Convenience

When deploying production web applications, developers face two contrasting architectural paradigms:

#### Paradigm A: The Cloud PaaS Trap (Vercel / Render / Heroku / Neon)
- **The Convenience:** Push to GitHub $\rightarrow$ automated build, automatic SSL, serverless scalability.
- **The Hidden Gotchas:**
  - **Cold Start Latency:** Serverless backends sleep when idle; first requests take 30–50 seconds to boot.
  - **Cross-Cloud Latency:** Frontend on Vercel (AWS US-East) talking to backend on Render (Frankfurt) talking to database on Neon (US-West) adds 150ms–300ms network transit latency to **every single database query**.
  - **Bandwidth & Compute Cost Spikes:** Free tiers have harsh limits; scaling compute costs escalate exponentially.

#### Paradigm B: The Self-Hosted VPS Reality (Azure / AWS EC2 / DigitalOcean)
- **The Power:** You own the machine. A flat predictable rate ($5–$15/mo) runs unlimited containers with `< 0.1ms` inter-service latency over private Docker bridge networks.
- **The Operational Hardship:**
  - **Out-of-Memory (OOM) Crashes:** Low-spec VPS instances (1 GB RAM) crash when npm/Vite compilers exhaust physical RAM.
  - **Security Surface:** You are solely responsible for OS patches, firewall rules (UFW), SSH key protection, and disk space management.

---

## 2. The Mental Model

### Architectural Topology Comparison
```
TRACK A: Self-Hosted Monolithic VPS               TRACK B: Decoupled Multi-Cloud PaaS
┌────────────────────────────────────────┐       ┌─────────────────────────────────────┐
│ Microsoft Azure VM (Ubuntu 24.04 LTS)  │       │ 1. Vercel Edge CDN (Global Anycast) │
│ ┌────────────────────────────────────┐ │       │    Serves React Frontend            │
│ │ UFW Firewall: Only 22, 80, 443     │ │       └──────────────────┬──────────────────┘
│ └─────────────────┬──────────────────┘ │                          │ Cross-Cloud TLS
│                   ▼                    │                          │ (Latency: ~80ms)
│ ┌────────────────────────────────────┐ │                          ▼
│ │ Docker Bridge (movie_network)      │ │       ┌─────────────────────────────────────┐
│ │                                    │ │       │ 2. Render Web Service (Node/Express)│
│ │  [ Nginx ]                         │ │       │    Cold starts if idle; ephemeral   │
│ │     ├── Serves static dist/        │ │       └──────────────────┬──────────────────┘
│ │     └── Reverse proxies /api/      │ │                          │ Public Transit
│ │             │ Inter-container      │ │                          │ (Latency: ~60ms)
│ │             ▼ (< 0.1ms latency)    │ │                          ▼
│ │  [ Express Backend Container ]     │ │       ┌─────────────────────────────────────┐
│ │             │ Internal only        │ │       │ 3. Neon Serverless PostgreSQL       │
│ │             ▼                      │ │       │    Connection pooling over internet │
│ │  [ PostgreSQL Container (pgdata) ] │ │       └─────────────────────────────────────┘
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘
Inter-service latency: ~0.05ms (Local SSD)       Inter-service latency: 150ms - 250ms
```

---

## 3. Production Code Breakdown

### A. Linux Host Hardening & Swap Configuration (`setup_vps.sh`)
```bash
#!/bin/bash
# Run on fresh Ubuntu 24.04 LTS VPS instance

# 1. Update OS packages
sudo apt-get update && sudo apt-get upgrade -y

# 2. Configure 2 GB Swap Space (CRITICAL for 1 GB RAM servers)
# Prevents OOM crashes during 'docker compose build' and npm compilation
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make swap permanent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Optimize swappiness (10 = prefer physical RAM, use swap only as emergency overflow)
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# 3. Configure UFW (Uncomplicated Firewall)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp   # SSH (Remote administration)
sudo ufw allow 80/tcp   # HTTP (Certbot verification)
sudo ufw allow 443/tcp  # HTTPS (Encrypted web traffic)
# ⚠️ Port 5432 is NEVER opened; database remains sealed!
sudo ufw --force enable

# 4. Install Official Docker Engine & Compose Plugin
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Allow current user to run Docker without sudo
sudo usermod -aG docker $USER
```

### B. Windows SSH Permission Hardening (`icacls`)
```powershell
# Windows PowerShell command to fix "Permissions for private key are too open" error:
icacls.exe .\server_key.pem /reset
icacls.exe .\server_key.pem /grant:r "$($env:USERNAME):(R)"
icacls.exe .\server_key.pem /inheritance:r

# Connect securely to VPS
ssh -i .\server_key.pem azureuser@104.208.80.230
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The OOM Killer during `docker compose build`
- **The Trap:** Running `docker compose build` on a 1 GB RAM VPS without Swap.
- **The Crash:** The Vite/esbuild compiler spikes memory usage to 850 MB. The Linux kernel runs out of memory, triggers the `Out of Memory: Kill process` killer, and instantly terminates the SSH daemon or Docker build abruptly.
- **The Fix:** Always provision a Swap file equal to or double your physical RAM (e.g. 2 GB swap on a 1 GB RAM machine) with `vm.swappiness=10`.

### ⚠️ Gotcha 2: The SSH Key Permission Rejection
- **The Trap:** Trying to connect via SSH using a private key file that has inherited permissions on Windows or permissions looser than `chmod 600` on Linux.
- **The Error:** `Permissions 0644 for 'key.pem' are too open. It is required that your private key files are NOT accessible by others.`
- **The Fix:** Strip all inherited access so only your current operating system user account has read permissions.

### ⚠️ Gotcha 3: The Architecture Decision Framework
- **Choose Managed PaaS (Vercel + Render + Neon) when:**
  - You are building an early MVP and need instant deployment with zero server maintenance.
  - You lack Linux/DevOps expertise and cannot manage firewalls or security patches.
- **Choose Self-Hosted VPS (Docker + Nginx on Linux) when:**
  - You need maximum database throughput, low latency (`< 0.1ms`), and high transactions.
  - You want predictable fixed costs and full control over your infrastructure.
