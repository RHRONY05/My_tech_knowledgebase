---
tags:
  - devops
  - docker
  - optimization
  - security
last_reviewed: 2026-10-02
related_notes:
  - "[[docker-core-and-networking]]"
  - "[[docker-compose-production-patterns]]"
---

# Production Dockerfiles, Layer Caching Optimization & Multi-Stage Builds

> **Active Recall Self-Test:**
> 1. Why does copying `package*.json` and running `npm install` BEFORE copying source code (`COPY . .`) dramatically accelerate build times?
> 2. How do **Multi-Stage Builds** reduce production image sizes from 1.2 GB to under 30 MB for frontend applications?
> 3. Why should production Node.js images run `npm ci --omit=dev` instead of standard `npm install`?
> 4. What critical security and performance hazards occur if `.dockerignore` omits `node_modules` or `.env` files?

---

## 1. The Core Problem
### The Bloated, Slow, and Insecure Docker Build

#### Anti-Pattern A: Cache Busting on Every Single Line Edit
In naive Dockerfiles, developers copy everything at once:
```dockerfile
FROM node:20
WORKDIR /app
COPY . .                  # ❌ BUSTS CACHE ON EVERY KEYSTROKE
RUN npm install           # ❌ Re-downloads 800MB of npm dependencies every build!
CMD ["npm", "start"]
```
- **The Pain:** A developer fixes a single typo in `src/index.js`. Because `COPY . .` changed, Docker invalidates the build cache for that line and **every line below it**. The developer waits 45 seconds for `npm install` to re-download all packages.

#### Anti-Pattern B: Shipping the Entire Build Toolchain to Production
For React/Vite frontends, shipping Node.js, npm, compilers, and `node_modules` to production:
- **Massive Attack Surface:** Security scanners flag dozens of vulnerabilities (CVEs) in development tools (`esbuild`, `jest`, `vite`) that have no business existing on a production server.
- **Image Bloat:** The container image balloons to **1.2 GB**, taking minutes to transfer across CI/CD registries and consuming server disk space.

#### The Architectural Solution
1. **Deterministic Layer Caching:** Order instructions from least frequently changed (dependencies) to most frequently changed (source code).
2. **Multi-Stage Builds:** Use a heavy build stage to compile code, then copy **only the static output artifacts (`dist/`)** into an ultra-lean production runner (e.g. `nginx:alpine` or minimal runtime).

---

## 2. The Mental Model

### The Docker Layer Caching Hierarchy
Docker builds images in stackable, read-only layers. If a layer's input hash hasn't changed, Docker reuses the cached layer instantly (0.0 seconds).
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. FROM node:20-alpine     ──► CACHED (Base OS layer downloaded once)       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. WORKDIR /app            ──► CACHED (Directory creation)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. COPY package*.json ./   ──► CACHED (Unless you added/removed dependencies)│
├─────────────────────────────────────────────────────────────────────────────┤
│ 4. RUN npm ci --omit=dev   ──► CACHED! (Reused instantly if package.json     │
│                                hasn't changed!)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ 5. COPY src/ ./src/        ──► REBUILDS in 0.1s when you edit application   │
│                                code, reusing cached npm layers above!       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Multi-Stage Build Architecture (Frontend Example)
```
STAGE 1: BUILDER (Discarded after build)
┌────────────────────────────────────────────────────────┐
│ node:20-alpine (~180 MB)                               │
│ - Copies all source files & JSX                        │
│ - Installs full dependencies (Vite, Tailwind, etc.)    │
│ - Runs: npm run build ──► Produces /app/dist/          │
└───────────────────────────┬────────────────────────────┘
                            │ (Extract ONLY /app/dist/)
                            ▼
STAGE 2: PRODUCTION RUNNER (Final Shipped Image)
┌────────────────────────────────────────────────────────┐
│ nginx:alpine (~25 MB)                                  │
│ - Zero Node.js runtime!                                │
│ - Zero node_modules!                                   │
│ - Copies plain HTML/CSS/JS into /usr/share/nginx/html  │
│                                                        │
│ Final Image Size: ~28 MB (97% size reduction!) ✅      │
└────────────────────────────────────────────────────────┘
```

---

## 3. Production Code Breakdown

### A. Essential `.dockerignore` (`.dockerignore`)
```
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
dist
build
coverage
*.md
```

### B. Multi-Stage React Frontend Dockerfile (`frontend/Dockerfile`)
```dockerfile
# =========================================================================
# STAGE 1: Compilation Builder (Discarded)
# =========================================================================
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependency manifests first for layer caching
COPY package*.json ./

# Install ALL dependencies (including devDependencies required for vite build)
RUN npm ci

# Copy application source code
COPY . .

# Compile JSX to static HTML, CSS, and JS bundles
RUN npm run build

# =========================================================================
# STAGE 2: Ultra-Lean Production Server (Shipped Image)
# =========================================================================
FROM nginx:alpine

# Copy custom Nginx configuration with SPA routing fallback
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy compiled static assets from builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Document port 80 for HTTP ingress
EXPOSE 80

# Nginx runs automatically in foreground
CMD ["nginx", "-g", "daemon off;"]
```

### C. Hardened Backend Multi-Stage Dockerfile (`backend/Dockerfile`)
```dockerfile
# =========================================================================
# STAGE 1: Dependency Resolver
# =========================================================================
FROM node:20-alpine AS dependencies

WORKDIR /app

COPY package*.json ./

# Install ONLY production dependencies (skips jest, supertest, nodemon)
RUN npm ci --omit=dev

# =========================================================================
# STAGE 2: Hardened Production Runner
# =========================================================================
FROM node:20-alpine AS runner

WORKDIR /app

# Set production environment
ENV NODE_ENV=production

# Copy production node_modules from dependencies stage
COPY --from=dependencies /app/node_modules ./node_modules
COPY package*.json ./
COPY src/ ./src/
COPY migrations/ ./migrations/

# ⚠️ SECURITY HARDENING: Run as unprivileged non-root user
# Node alpine includes a pre-configured 'node' user (UID 1000)
USER node

EXPOSE 5000

CMD ["node", "src/server.js"]
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Missing `.dockerignore` Trap
- **The Trap:** Omitting `.dockerignore` when building an image on a machine that has local `node_modules/`.
- **The Bug:** Docker copies your local Windows/macOS compiled `node_modules` into the Linux container. Any native C++ binary packages (e.g. `bcrypt`, `sharp`) immediately crash at runtime with `invalid ELF header` or architecture mismatches.
- **The Fix:** Always exclude `node_modules` in `.dockerignore` so dependencies are compiled natively inside the Linux container via `npm ci`.

### ⚠️ Gotcha 2: `npm install` vs `npm ci`
- **The Trap:** Using `RUN npm install` inside Dockerfiles.
- **Why:** `npm install` can resolve and install newer minor/patch versions than what you tested locally if your `package.json` uses semver ranges (`^1.2.0`). It also writes back to `package-lock.json`.
- **The Fix:** Always use `RUN npm ci` (Clean Install). It strictly installs the exact frozen dependency tree from `package-lock.json` and deletes any pre-existing `node_modules`.

### ⚠️ Gotcha 3: Leaking Secrets via Docker Image History
- **The Trap:** Passing API keys or passwords via `ENV SECRET_KEY=abc123` or copying `.env` into the image.
- **Why:** Every layer in a Docker image is immutable and stored in the manifest. Anyone with access to the image can run `docker history --no-trunc <image>` and read your plain-text secrets in seconds.
- **The Fix:** Never bake secrets into images. Pass configuration at runtime via environment variables (`-e` or Docker Compose `env_file`).
