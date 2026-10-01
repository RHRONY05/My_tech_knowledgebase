---
tags:
  - devops
  - ci-cd
  - github-actions
  - automation
  - testing
last_reviewed: 2026-10-02
related_notes:
  - "[[jest-supertest-integration-testing]]"
  - "[[multi-environment-db-isolation]]"
---

# Continuous Integration (CI) with GitHub Actions: Sterile Runners & Service Containers

> **Active Recall Self-Test:**
> 1. If a developer already runs `npm test` locally on their laptop, what three critical failure modes (e.g. "Forgotten File" trap, Windows case-insensitivity) make cloud CI mandatory?
> 2. How does GitHub Actions use **Service Containers** (`services: postgres`) to provision an ephemeral database instance during test execution?
> 3. What is the difference between running tests against a "dirty" developer laptop vs. a 100% "sterile" cloud virtual machine runner?
> 4. How does `actions/cache` optimize CI pipeline execution times by preserving `node_modules` based on `package-lock.json` hash keys?

---

## 1. The Core Problem
### The "It Works on My Machine" Syndrome

In team engineering, developers merge code into `main` after testing locally on their personal laptops.

#### Catastrophe A: The "Forgotten File" Git Gotcha
A developer implements a new feature and creates a helper file: `backend/src/utils/dateFormatter.js`.
- They run `npm test` locally. It passes 100%. Why? Because the file physically exists on their SSD!
- But they forgot to `git add` that file, or an overbroad `.gitignore` rule excluded it.
- **Without Cloud CI:** The Pull Request is merged into `main`. The production server pulls the code and immediately crashes for all live customers with `Error: Cannot find module './utils/dateFormatter.js'`.

#### Catastrophe B: The Windows vs. Linux Case-Sensitivity Trap
- **Windows / macOS are case-insensitive:** If a component file is named `SeatMap.jsx`, Windows happily resolves `import SeatMap from './seatMap.jsx'`. Local tests pass!
- **Linux (Production) is strictly case-sensitive:** When deployed to production Ubuntu servers, Node.js cannot find `seatMap.jsx` and crashes.

#### Catastrophe C: The Dirty Local State Trap
A developer's laptop has residual database records from yesterday, global npm packages (`npm install -g`), or forgotten environment variables. The code works locally only because of that leftover state.

#### The Architectural Solution: Cloud CI Gatekeeper
Continuous Integration spins up a **brand-new, completely sterile Linux virtual machine** on every push. It clones strictly what was committed to Git, spins up disposable service containers, and executes builds and tests. If even a single test or lint check fails, **GitHub physically blocks the merge into `main`**.

---

## 2. The Mental Model

### The CI Gatekeeper Pipeline
```
[ Developer Git Push / Pull Request ]
                 │
                 ▼
[ GitHub Cloud Event Trigger ]
                 │
                 ▼
[ Spins up Sterile Ubuntu 24.04 Virtual Machine ]
  ├── 1. Clones Repository (Strictly Git-committed files only!)
  ├── 2. Restores Cached Dependencies (npm cache)
  ├── 3. Spins up Docker Service Container: PostgreSQL 15 (Port 5432)
  │      └── Executes Healthcheck: pg_isready
  ├── 4. Runs Database Migrations: npm run migrate up
  ├── 5. Runs Static Analysis & Linters: npm run lint
  └── 6. Executes Test Suites: npm test
                 │
         ┌───────┴───────┐
         ▼               ▼
      [ PASS ]        [ FAIL ]
      Merged to main  Merge Blocked! ❌
                      Sends alert; protects production
```

---

## 3. Production Code Breakdown

### Complete Production CI Workflow (`.github/workflows/ci.yml`)
```yaml
name: Continuous Integration (CI) Gatekeeper

# Trigger on pushes to main and all Pull Requests targeting main
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test-and-verify:
    name: Backend Lint, Migrate & Integration Tests
    runs-on: ubuntu-latest # 100% Sterile Linux Environment

    # =========================================================================
    # DOCKER SERVICE CONTAINERS (Ephemeral PostgreSQL)
    # =========================================================================
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: movie_booking_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        # Healthcheck ensures database engine is listening before tests run
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    # =========================================================================
    # STEP-BY-STEP EXECUTION PIPELINE
    # =========================================================================
    steps:
      # Step 1: Check out source code
      - name: Checkout Repository
        uses: actions/checkout@v4

      # Step 2: Set up Node.js runtime
      - name: Setup Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: 'backend/package-lock.json'

      # Step 3: Clean install dependencies from lockfile
      - name: Install Backend Dependencies
        working-directory: ./backend
        run: npm ci

      # Step 4: Run Static Code Analysis / Linter
      - name: Execute Linter
        working-directory: ./backend
        run: npm run lint --if-present

      # Step 5: Run Database Migrations against the Service Container
      - name: Execute Database Migrations
        working-directory: ./backend
        env:
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/movie_booking_test
        run: npm run migrate up

      # Step 6: Execute Automated Integration Test Suites
      - name: Run Jest & Supertest Suites
        working-directory: ./backend
        env:
          NODE_ENV: test
          PORT: 5001
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/movie_booking_test
          JWT_SECRET: super_secret_ci_jwt_key_12345
        run: npm test
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Service Container Port Mapping on Linux Runners
- **The Trap:** When running service containers on `ubuntu-latest`, developers assume they must refer to `postgres:5432`.
- **The Quirk:** In GitHub Actions on Linux, service container ports mapped as `5432:5432` are forwarded directly to the runner's host network. Your connection string must use `localhost:5432` (or `127.0.0.1:5432`), NOT `postgres:5432`.

### ⚠️ Gotcha 2: The Database Migration Pre-Flight Requirement
- **The Trap:** Running `npm test` without running `npm run migrate up` first in CI.
- **The Crash:** The service container starts with an empty PostgreSQL catalog. Tests immediately fail with `relation "users" does not exist`.
- **The Fix:** Always run your migration step (`npm run migrate up`) before invoking the test suite step.

### ⚠️ Gotcha 3: The Untracked Dependency Trap (`npm install` vs `npm ci`)
- **The Trap:** Using `npm install` in your CI workflow step.
- **The Danger:** `npm install` tolerates discrepancies between `package.json` and `package-lock.json`.
- **The Rule:** Always use `npm ci`. If someone edited `package.json` without updating the lockfile, `npm ci` fails immediately with an error, guaranteeing that production matches the exact locked dependencies.
