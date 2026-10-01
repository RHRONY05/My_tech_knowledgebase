---
tags:
  - active-recall
  - qna
  - backend
  - docker
  - database
last_reviewed: 2026-10-02
related_notes:
  - "[[master-index]]"
  - "[[transactions-and-pessimistic-locking]]"
  - "[[docker-core-and-networking]]"
---

# Backend, Database & Docker Mastery — Active Recall Q&A Guide

> **How to practice:** Read each question, verbalize your answer or write it on a scratchpad, then expand the details block to verify your understanding.

---

## 🏛️ Part 1: Architecture, Node.js & Database

### Q1: What is the "Fail-Fast" startup pattern, and why must the database connect BEFORE `app.listen()`?
<details>
<summary><b>View Answer</b></summary>

- **What it is:** Testing the database connection pool (`await pool.connect()`) before starting the Express HTTP server (`app.listen()`).
- **Why:** If the database credentials are wrong or the DB container is down, we want the server process to crash immediately at startup (`process.exit(1)`) rather than accepting incoming client traffic that will inevitably fail with 500 errors.
</details>

---

### Q2: What is the difference between `pool.query()` and `pool.connect()` in PostgreSQL?
<details>
<summary><b>View Answer</b></summary>

- **`pool.query()`:** Automatically borrows a client connection from the pool, runs a single standalone query, and releases it back immediately. Used for simple reads and writes.
- **`pool.connect()`:** Checks out a dedicated client connection and holds it. **Mandatory for ACID Transactions** because all commands (`BEGIN`, `SELECT ... FOR UPDATE`, `COMMIT`, `ROLLBACK`) must execute sequentially on the *exact same* physical database socket. Must call `client.release()` in a `finally` block when done.
</details>

---

### Q3: Why is the relationship between `seats` and `bookings` 1-to-Many (`1 : *`) instead of 1-to-1?
<details>
<summary><b>View Answer</b></summary>

- A single physical seat can have **multiple booking attempts over time** in the historical audit log (e.g. User 1 reserves $\rightarrow$ OTP expires $\rightarrow$ marked `FAILED`; 10 mins later User 2 reserves $\rightarrow$ marked `CONFIRMED`).
- If we enforced a 1-to-1 unique constraint (`UNIQUE (seat_id)`), once a single booking failed or was cancelled, no user would ever be able to book that seat again!
</details>

---

## 🔒 Part 2: Concurrency, Locking & Transactions

### Q4: What is a Race Condition, and how do we simulate it in integration tests?
<details>
<summary><b>View Answer</b></summary>

- **Race Condition:** When multiple concurrent requests attempt to read and modify the same resource simultaneously (e.g. 20 users clicking "Book Seat A1" at the exact same millisecond).
- **Simulation:** In integration tests, use `Promise.all()` to dispatch 20 simultaneous `supertest` HTTP requests to `/api/bookings/initiate` without awaiting them sequentially.
</details>

---

### Q5: How does Pessimistic Locking (`SELECT ... FOR UPDATE`) solve the double-booking problem?
<details>
<summary><b>View Answer</b></summary>

- When Transaction #1 runs `SELECT * FROM seats WHERE id = $1 FOR UPDATE`, PostgreSQL places an exclusive **Row-Level Lock** on that specific seat record.
- When Transactions #2 through #20 arrive, PostgreSQL **pauses them in a queue** until Transaction #1 finishes (`COMMIT` or `ROLLBACK`).
- Transaction #1 marks the seat `RESERVED` and commits.
- When Transaction #2 unpauses, it re-reads the row, sees `status = 'RESERVED'`, and is immediately rejected with HTTP `409 Conflict`.
</details>

---

### Q6: What does the ACID acronym stand for, and why does atomicity matter?
<details>
<summary><b>View Answer</b></summary>

- **A - Atomicity:** "All or Nothing." If reserving a seat succeeds but inserting the booking record fails, the entire transaction is rolled back so no ghost reserved seats exist.
- **C - Consistency:** The database moves from one valid state to another, satisfying all foreign keys and constraints.
- **I - Isolation:** Concurrent transactions execute without interfering with one another.
- **D - Durability:** Once committed, changes are permanently written to disk write-ahead logs.
</details>

---

## 🐳 Part 3: Docker & DevOps

### Q7: What is the fundamental difference between a Docker Container and a Virtual Machine?
<details>
<summary><b>View Answer</b></summary>

- A **Virtual Machine** simulates fake physical hardware (CPU, RAM, BIOS) and runs a heavy guest OS (takes gigabytes of RAM and boots in minutes).
- A **Docker Container** is a lightweight, isolated Linux process running **directly on the host kernel** using **Namespaces** (virtual walls for PID, network, filesystem) and **Cgroups** (CPU/RAM limits). It boots in milliseconds with near-zero overhead.
</details>

---

### Q8: Inside a Docker container, why does `postgres://localhost:5432` fail?
<details>
<summary><b>View Answer</b></summary>

- Inside a container, `localhost` (127.0.0.1) refers strictly to **that specific container's private network namespace**.
- To reach sibling containers on a custom Docker bridge network, use the **Docker service name** (e.g. `postgres://postgres:5432`), which Docker's embedded DNS server resolves automatically.
</details>

---

### Q9: Why does Dockerfile instruction order matter for Layer Caching?
<details>
<summary><b>View Answer</b></summary>

- Docker builds images in stackable layers from top to bottom. If a line changes, Docker invalidates the cache for that line and every line below it.
- **Golden Rule:** Always copy `package*.json` and run `npm ci` **before** copying source code (`COPY . .`). That way, editing application code rebuilds in 0.1s instead of re-downloading all npm dependencies.
</details>

---

### Q10: Why does naive `depends_on: [postgres]` cause startup crashes in Docker Compose?
<details>
<summary><b>View Answer</b></summary>

- Docker only waits for the `postgres` container process to launch, NOT for PostgreSQL to finish loading its database engine.
- The backend starts in 1 second, attempts to query before PostgreSQL is listening on port 5432, and crashes.
- **Fix:** Define a `healthcheck` (`pg_isready`) on PostgreSQL and set `depends_on: { postgres: { condition: service_healthy } }`.
</details>
