---
tags:
  - devops
  - docker
  - orchestration
  - database
last_reviewed: 2026-10-02
related_notes:
  - "[[docker-core-and-networking]]"
  - "[[dockerfile-and-layer-caching]]"
  - "[[database-migrations-node-pg-migrate]]"
---

# Docker Compose Production Patterns: The 4 Pillars, Healthchecks & Volumes

> **Active Recall Self-Test:**
> 1. What are the **4 Pillars** that define every Docker Compose architecture?
> 2. Why does naive `depends_on: [postgres]` cause application startup crashes, and how does `condition: service_healthy` solve this race condition?
> 3. What is the fundamental difference between a **Named Volume** (`pgdata:/var/lib/postgresql/data`) and a **Bind Mount** (`./src:/app/src`)?
> 4. Why should database migrations run as a dedicated **One-Shot Task** before backend services boot?

---

## 1. The Core Problem
### The Fragility of Multi-Container Orchestration

Running multi-tier applications (Frontend, Backend, Database, Cache) by typing long individual `docker run` commands is error-prone and brittle:
- Passing dozens of flags (`-e`, `-v`, `-p`, `--network`) manually leads to forgotten configurations.
- Containers boot in non-deterministic order.

#### Trap A: The `depends_on` Startup Race Condition
In a standard Compose file:
```yaml
backend:
  depends_on:
    - postgres # ❌ NAIVE DEPENDENCY
```
- **Why it crashes:** Docker interprets `depends_on` as: *"Start the postgres container process, then immediately start the backend container."*
- **The Failure:** The PostgreSQL engine requires 3 to 10 seconds to initialize its storage engine, allocate memory buffers, and open its listening socket on port 5432. The backend boots in 1 second, attempts `pool.connect()`, receives `ECONNREFUSED`, and crashes with exit code 1!

#### Trap B: Ephemeral Database Data Loss
If a database runs without a volume, all data resides in the container's writable layer. Running `docker compose down` destroys the container and **permanently wipes every customer record, seat booking, and user password**.

#### The Architectural Solution
1. **The 4-Pillar Model:** Structure configurations across Services, Networks, Volumes, and Config.
2. **Healthcheck Contracts:** Gate dependent service boot sequences on genuine health readiness checks (`pg_isready`).
3. **Dedicated Named Volumes:** Decouple stateful database persistence from ephemeral container lifecycles.
4. **One-Shot Migration Tasks:** Ensure database schemas migrate forward before application traffic starts.

---

## 2. The Mental Model

### The 4 Pillars of Docker Compose
```
┌────────────────────────────────────────────────────────────────────────┐
│                        DOCKER COMPOSE ECOSYSTEM                        │
│                                                                        │
│  1. SERVICES  ──► Who computes? (Nginx, Express Backend, PostgreSQL)   │
│  2. NETWORKS  ──► Private software-defined bridge switch               │
│  3. VOLUMES   ──► Persistent storage on host SSD (survives rebuilds)   │
│  4. CONFIG    ──► Secrets, healthchecks, ports, restart policies       │
└────────────────────────────────────────────────────────────────────────┘
```

### Deterministic Startup Sequencing with Healthchecks
```
Time (seconds) ──►
0s: [ postgres ] starts booting...
    ├── backend WAITS (State: created, not started)
    └── migrations WAIT

3s: [ postgres ] completes internal engine startup
    └── Healthcheck executes: pg_isready -U postgres
    └── Health Status: HEALTHY ✅

4s: [ migration (One-Shot Task) ] boots up!
    ├── Runs: npm run migrate up
    └── Applies new tables, exits with code 0 ✅

5s: [ backend ] starts booting!
    ├── Database is 100% ready & fully migrated
    ├── pool.connect() succeeds instantly
    └── Backend listens on port 5000 ✅

6s: [ frontend / Nginx ] starts routing client traffic
```

---

## 3. Production Code Breakdown

### Production Full-Stack Orchestration (`docker-compose.prod.yml`)
```yaml
services:
  # =========================================================================
  # 1. DATABASE TIER (PostgreSQL 15)
  # =========================================================================
  postgres:
    image: postgres:15-alpine
    container_name: movie_booking_prod_db
    restart: always
    environment:
      POSTGRES_DB: ${DB_NAME:-movie_booking}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      # Named volume ensures data survives container destroy/rebuild
      - pgdata_prod:/var/lib/postgresql/data
    networks:
      - movie_network
    # ⚠️ CRITICAL: Healthcheck verifies DB engine is actively accepting queries
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres} -d ${DB_NAME:-movie_booking}"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    # In production, NO ports: exposed to public host!

  # =========================================================================
  # 2. ONE-SHOT MIGRATION TIER
  # =========================================================================
  migration:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: movie_booking_prod_migration
    command: ["npm", "run", "migrate", "up"]
    environment:
      DATABASE_URL: postgres://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-movie_booking}
    networks:
      - movie_network
    depends_on:
      postgres:
        condition: service_healthy # Waits for healthcheck before running!
    restart: "no" # One-shot task: exits upon completion

  # =========================================================================
  # 3. BACKEND API TIER
  # =========================================================================
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: movie_booking_prod_backend
    restart: always
    environment:
      PORT: 5000
      NODE_ENV: production
      DATABASE_URL: postgres://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-movie_booking}
      JWT_SECRET: ${JWT_SECRET}
    volumes:
      # Bind mount for persistent log file shipping
      - ./backend/logs:/app/logs
    networks:
      - movie_network
    depends_on:
      postgres:
        condition: service_healthy
      migration:
        condition: service_completed_successfully # Runs only after schema migration!

  # =========================================================================
  # 4. FRONTEND / REVERSE PROXY TIER
  # =========================================================================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: movie_booking_prod_frontend
    restart: always
    ports:
      - "80:80"   # HTTP ingress
      - "443:443" # HTTPS ingress
    networks:
      - movie_network
    depends_on:
      - backend

# =========================================================================
# NETWORKS & VOLUMES DEFINITION
# =========================================================================
networks:
  movie_network:
    driver: bridge

volumes:
  pgdata_prod:
    driver: local
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Dev vs Prod Volume Trap
- **The Trap:** Leaving bind mounts (`./backend:/app`) in your production Compose file.
- **The Danger:** A bind mount overrides whatever code was built into the Docker image with your host folder. If someone accidentally edits or deletes a file on the host server, the production container's code mutates live in memory without a traceable git commit or image tag.
- **The Rule:** In production, bake code immutably into the image using `COPY . .`. Use volumes strictly for:
  - Persistent database storage (`Named Volumes`).
  - Log directories and TLS certificates (`Read-Only Bind Mounts`).

### ⚠️ Gotcha 2: The `restart: always` Crash Loop Trap
- **The Trap:** Setting `restart: always` on a one-shot container like `migration` or `seed`.
- **The Consequence:** The migration executes successfully, exits with status 0, and Docker immediately restarts it in an infinite loop, constantly rerunning migrations and spamming logs.
- **The Fix:** Always specify `restart: "no"` on one-shot task services.

### ⚠️ Gotcha 3: The Untracked `.env` Deployment Crash
- **The Trap:** Running `docker compose up -d` on a fresh VPS and receiving empty string values for `${DB_PASSWORD}` and `${JWT_SECRET}`.
- **The Consequence:** PostgreSQL initializes with blank or default passwords, leaving security gates open.
- **The Fix:** Maintain a `.env.example` in source control listing all required variable keys. Use a deployment checklist or CI/CD secret injection to verify `.env` presence before triggering compose commands.
