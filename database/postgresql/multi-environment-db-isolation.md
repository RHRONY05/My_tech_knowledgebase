---
tags:
  - database
  - postgresql
  - testing
  - docker
  - devops
last_reviewed: 2026-10-02
related_notes:
  - "[[transactions-and-pessimistic-locking]]"
  - "[[database-migrations-node-pg-migrate]]"
  - "[[jest-supertest-integration-testing]]"
---

# Multi-Environment Database Isolation: Server Instances vs. Logical Databases

> **Active Recall Self-Test:**
> 1. Why does running automated Jest test suites against your active local development database cause data pollution, flaky tests, and accidental data wipes?
> 2. What is the fundamental difference between a **Database Server Process (Instance)** and a **Logical Database** in PostgreSQL?
> 3. How does Docker's `/docker-entrypoint-initdb.d/` directory automatically provision isolated test databases on container initialization?
> 4. How does dynamic environment switching in `src/config/db.js` guarantee that tests NEVER touch development data, even if connection strings are omitted?

---

## 1. The Core Problem
### The "Shared Database" Contamination Trap

In junior development setups, automated tests run against the same database used for local feature building and UI testing:
```
❌ THE SHARED DATABASE ANTI-PATTERN:
┌────────────────────────────────────────────────────────┐
│               PostgreSQL: "movie_booking"              │
│                                                        │
│  [Dev Work / UI] ─────────┐                            │
│  - Real sample movies     │                            │
│  - Active user profiles   ▼                            │
│                      DATABASE                          │
│  [Jest Test Suites] ──────▲                            │
│  - Inserts dummy seats    │                            │
│  - Random fake users      │                            │
│  - Truncates tables ──────┘                            │
└────────────────────────────────────────────────────────┘
```

#### Why This Breaks in Industry:
1. **UI Pollution:** Random dummy records (`"Test User #84729"`, `"Test Seat Dummy"`) populate your React development UI, confusing manual verification.
2. **Brittle / Flaky Tests:** If you are testing a booking in your browser while Jest runs in another terminal, database rows mutate concurrently, causing test assertions to fail intermittently.
3. **Accidental Data Loss:** Test teardown hooks often run `TRUNCATE TABLE seats CASCADE`. If pointed at the dev database, weeks of carefully seeded development data vanish in 200ms.

#### The Architectural Solution: Logical Database Sandboxing
- A single PostgreSQL container process manages **multiple completely isolated logical databases** (`movie_booking` for Dev, `movie_booking_test` for Automated Testing).
- Zero extra memory or CPU overhead—both share port 5432 and the same underlying PostgreSQL engine.
- Dynamic environment configuration switches databases automatically based on `NODE_ENV`.

---

## 2. The Mental Model

### PostgreSQL Instance vs. Logical Databases
```
┌─────────────────────────────────────────────────────────────────────────┐
│ Docker Container ("postgres:15-alpine" - Port 5432)                     │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ PostgreSQL Database Engine (RDBMS Daemon)                         │  │
│  │                                                                   │  │
│  │  ┌─────────────────────────────┐   ┌────────────────────────────┐ │  │
│  │  │ Logical DB: "movie_booking" │   │ Logical DB:                │ │  │
│  │  │ (Development)               │   │ "movie_booking_test"       │ │  │
│  │  │                             │   │ (Test Sandbox)             │ │  │
│  │  │  - users (dev accounts)     │   │  - users (sterile)         │ │  │
│  │  │  - movies (catalog)         │   │  - movies (ephemeral)      │ │  │
│  │  │  - bookings (manual test)   │   │  - bookings (ephemeral)    │ │  │
│  │  └─────────────────────────────┘   └────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

Queries running inside `movie_booking_test` cannot read, lock, or truncate tables in `movie_booking`. They are physically separated catalogs.

---

## 3. Production Code Breakdown

### A. Automatic DB Initialization Script (`docker/init-test-db.sh`)
PostgreSQL Docker images automatically execute any `.sql` or `.sh` script placed in `/docker-entrypoint-initdb.d/` upon the very first container initialization:
```bash
#!/bin/bash
set -e

# Create dedicated test database alongside default development database
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE DATABASE movie_booking_test;
    GRANT ALL PRIVILEGES ON DATABASE movie_booking_test TO $POSTGRES_USER;
EOSQL
```

### B. Dynamic Environment Switching (`src/config/db.js`)
```javascript
import pkg from 'pg';
const { Pool } = pkg;

// Detect active runtime environment
const isTestEnv = process.env.NODE_ENV === 'test';

// Deterministic database name resolution
const getDatabaseUrl = () => {
  if (isTestEnv) {
    return (
      process.env.TEST_DATABASE_URL ||
      'postgres://postgres:password123@localhost:5432/movie_booking_test'
    );
  }
  return (
    process.env.DATABASE_URL ||
    'postgres://postgres:password123@localhost:5432/movie_booking'
  );
};

// Centralized Connection Pool
const pool = new Pool({
  connectionString: getDatabaseUrl(),
  max: isTestEnv ? 5 : 20, // Smaller connection pool for test runners
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export default pool;
```

### C. Automated NPM Test Lifecycle Hooks (`package.json`)
Using npm's built-in `pre<script>` hook ensures the test database schema is always in 100% sync before Jest starts:
```json
{
  "scripts": {
    "migrate:test": "cross-env DATABASE_URL=postgres://postgres:password123@localhost:5432/movie_booking_test node-pg-migrate up",
    "pretest": "npm run migrate:test",
    "test": "cross-env NODE_ENV=test NODE_OPTIONS=--experimental-vm-modules jest --runInBand"
  }
}
```

### D. Test Suite Isolation Lifecycle (`tests/booking.concurrency.test.js`)
```javascript
import pool from '../src/config/db.js';

describe('Booking Concurrency & Locking', () => {
  // Clean table state before each test run
  beforeEach(async () => {
    await pool.query('TRUNCATE TABLE bookings, seats RESTART IDENTITY CASCADE;');
    await pool.query(`
      INSERT INTO seats (id, seat_number, price, status)
      VALUES (1, 'A1', 12.50, 'AVAILABLE');
    `);
  });

  // Close connection pool after all tests in the file complete
  afterAll(async () => {
    await pool.end(); // ⚠️ Critical: prevents Jest open-handle hangs
  });

  it('allows only ONE reservation out of 20 concurrent requests', async () => {
    // Test logic...
  });
});
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Docker Named Volume Initialization Trap
- **The Trap:** Adding `init-test-db.sh` to `/docker-entrypoint-initdb.d/` on a Docker container that **already has an existing named volume (`pgdata`) mounted**.
- **The Quirk:** PostgreSQL executes init scripts **ONLY ONCE** when the volume directory is completely empty. If an existing `pgdata` volume is detected, PostgreSQL skips `/docker-entrypoint-initdb.d/` entirely.
- **The Fix:** If you add an init script after the DB container was already run, recreate the volume cleanly:
  ```bash
  docker compose down -v
  docker compose up -d
  ```

### ⚠️ Gotcha 2: The Parallel Jest Database Race Condition
- **The Trap:** Running Jest without `--runInBand` against a shared test database.
- **The Crash:** Jest runs test files in parallel across multiple worker threads. If File A runs `TRUNCATE TABLE users` while File B is executing `POST /api/auth/login`, File B crashes intermittently!
- **The Fix:** Always specify `--runInBand` when testing relational database integrations so test suites run sequentially on the test sandbox.

### ⚠️ Gotcha 3: The Jest Open-Handle Hang (`pool.end()`)
- **The Trap:** Omitting `afterAll(async () => { await pool.end(); });`.
- **The Symptom:** All your tests pass green, but the Jest process hangs for 30 seconds before printing: `Jest did not exit one second after the test run has completed. This usually means that there are asynchronous operations that weren't stopped in your tests.`
- **The Fix:** Always gracefully shut down the PostgreSQL connection pool using `await pool.end()` in your test teardown hooks.
