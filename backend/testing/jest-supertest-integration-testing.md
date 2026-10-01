---
tags:
  - backend
  - testing
  - jest
  - supertest
  - integration-testing
last_reviewed: 2026-10-02
related_notes:
  - "[[multi-environment-db-isolation]]"
  - "[[transactions-and-pessimistic-locking]]"
  - "[[github-actions-ci-pipeline]]"
---

# Automated Backend Integration Testing: Jest, Supertest & Database Teardown

> **Active Recall Self-Test:**
> 1. Why does Supertest NOT require your Express server to listen on a physical TCP port (`app.listen()`), and how does in-memory request injection work?
> 2. What is the **AAA Pattern (Arrange, Act, Assert)**, and how does it structure every integration test?
> 3. Why does running integration tests concurrently across files break relational database test suites, and how does `--runInBand` guarantee isolation?
> 4. How do you mock third-party external services (like Google OAuth or Stripe) without firing live external HTTP requests during tests?

---

## 1. The Core Problem
### The Slowness and Fragility of Manual API Testing

Without automated integration tests:
- Developers test endpoints manually via Postman or browser clicks.
- Regression bugs slip into production unnoticed: fixing a movie query breaks the seat booking transaction.
- Testing edge cases (e.g. 20 concurrent booking requests hitting a race condition) is physically impossible to test manually.

#### Problem A: The Server Port Collision Trap
If your test runner has to boot a live HTTP server on port 5000:
- If a local dev server is already running, tests crash with `EADDRINUSE: address already in use :::5000`.
- Tests run slower due to TCP socket handshake overhead.

#### Problem B: Database State Cross-Contamination
If tests run in parallel across worker threads, one test's `TRUNCATE` wipes out records another test was asserting on, resulting in flaky, non-deterministic failures.

#### The Architectural Solution: In-Memory Integration Testing
1. **Supertest:** Injects simulated HTTP requests directly into the Express `app` object in process memory without opening network ports.
2. **Sequential Database Sandboxing (`--runInBand`):** Runs test suites sequentially against an isolated test database.
3. **Deterministic Lifecycle Hooks (`beforeEach` / `afterAll`):** Clean tables before each test and close connection pools cleanly to prevent memory leaks.

---

## 2. The Mental Model

### Jest vs. Supertest: Division of Labor
```
┌────────────────────────────────────────────────────────┐
│ TEST RUNNER: Jest                                      │
│ - The Judge: Discovers test files, manages mocks,      │
│   evaluates expect() assertions, and outputs reports.   │
└───────────────────────────┬────────────────────────────┘
                            │ Calls in-process
                            ▼
┌────────────────────────────────────────────────────────┐
│ HTTP CLIENT: Supertest                                 │
│ - The Lawyer: Takes your Express `app` object and      │
│   simulates HTTP requests (GET, POST) in Node memory.  │
│ - ZERO TCP ports opened! No EADDRINUSE collisions!     │
└───────────────────────────┬────────────────────────────┘
                            │ In-memory HTTP dispatch
                            ▼
┌────────────────────────────────────────────────────────┐
│ YOUR APPLICATION: Express app.js                       │
│ - Middlewares, Validators, Controllers, Services       │
└───────────────────────────┬────────────────────────────┘
                            │ SQL Queries
                            ▼
┌────────────────────────────────────────────────────────┐
│ TEST DATABASE: PostgreSQL ("movie_booking_test")       │
│ - Isolated from development database                   │
│ - Cleaned before each test via TRUNCATE                │
└────────────────────────────────────────────────────────┘
```

### The AAA Testing Pattern (Arrange - Act - Assert)
```
1. ARRANGE ──► Set up sterile test state:
               - Truncate dirty tables
               - Insert target test movie and seat record
               - Generate valid test JWT token

2. ACT     ──► Execute the operation under test:
               - Dispatch Supertest request: POST /api/bookings/initiate

3. ASSERT  ──► Verify the outcomes:
               - expect(response.status).toBe(201)
               - expect(response.body.success).toBe(true)
               - Query database: verify seat status changed to 'RESERVED'
```

---

## 3. Production Code Breakdown

### A. NPM Configuration with ES Modules (`package.json`)
```json
{
  "scripts": {
    "test": "cross-env NODE_ENV=test NODE_OPTIONS=--experimental-vm-modules jest --runInBand --detectOpenHandles"
  },
  "jest": {
    "testEnvironment": "node",
    "transform": {},
    "testTimeout": 10000
  }
}
```

### B. Complete Concurrency & Integration Test Suite (`tests/booking.concurrency.test.js`)
```javascript
import request from 'supertest';
import jwt from 'jsonwebtoken';
import app from '../src/app.js';
import pool from '../src/config/db.js';

describe('Integration Test: Seat Booking Concurrency & Pessimistic Locking', () => {
  let authTokenUser1;
  let authTokenUser2;
  const TEST_SEAT_ID = 1;

  // 1. LIFECYCLE: Setup database and test users before suite runs
  beforeAll(async () => {
    // Generate valid test JWT tokens
    authTokenUser1 = jwt.sign({ id: 1, role: 'customer' }, process.env.JWT_SECRET || 'test_secret');
    authTokenUser2 = jwt.sign({ id: 2, role: 'customer' }, process.env.JWT_SECRET || 'test_secret');
  });

  // 2. ISOLATION: Reset table rows before every single test
  beforeEach(async () => {
    await pool.query('TRUNCATE TABLE bookings, seats RESTART IDENTITY CASCADE;');

    // Insert clean test seat
    await pool.query(`
      INSERT INTO seats (id, seat_number, price, status)
      VALUES (${TEST_SEAT_ID}, 'A1', 15.00, 'AVAILABLE');
    `);
  });

  // 3. TEARDOWN: Close database connection pool to prevent open handles
  afterAll(async () => {
    await pool.end();
  });

  // Test Case: Race Condition Simulation
  it('prevents overselling: when 10 users click simultaneously, exactly ONE succeeds', async () => {
    // ARRANGE: Create 10 concurrent requests fired at the exact same millisecond
    const concurrentRequests = Array.from({ length: 10 }).map((_, idx) => {
      const token = jwt.sign({ id: idx + 10, role: 'customer' }, process.env.JWT_SECRET || 'test_secret');
      // ACT: Dispatch requests simultaneously without sequential await
      return request(app)
        .post('/api/bookings/initiate')
        .set('Authorization', `Bearer ${token}`)
        .send({ seatId: TEST_SEAT_ID });
    });

    const responses = await Promise.all(concurrentRequests);

    // ASSERT: Exactly one response must be HTTP 201 Created
    const successfulBookings = responses.filter((res) => res.status === 201);
    const rejectedBookings = responses.filter((res) => res.status === 409);

    expect(successfulBookings.length).toBe(1);
    expect(rejectedBookings.length).toBe(9);

    // Verify database state: only 1 booking record exists
    const dbBookingCount = await pool.query('SELECT COUNT(*) FROM bookings WHERE seat_id = $1;', [
      TEST_SEAT_ID,
    ]);
    expect(Number(dbBookingCount.rows[0].count)).toBe(1);

    // Verify seat status is RESERVED
    const seatRow = await pool.query('SELECT status FROM seats WHERE id = $1;', [TEST_SEAT_ID]);
    expect(seatRow.rows[0].status).toBe('RESERVED');
  });
});
```

### C. Mocking External Services (e.g. Google OAuth)
```javascript
import { OAuth2Client } from 'google-auth-library';

// Mock verifyIdToken without making real network calls to Google
jest.spyOn(OAuth2Client.prototype, 'verifyIdToken').mockResolvedValue({
  getPayload: () => ({
    email: 'mockuser@gmail.com',
    name: 'Mock Test User',
    sub: 'google_oauth_id_99999',
  }),
});
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The App vs Server Separation Rule
- **The Trap:** Exporting `app.listen(PORT)` from `app.js` and passing that to Supertest.
- **The Consequence:** Importing `app.js` in test files starts a physical HTTP server, triggering port conflicts.
- **The Golden Rule:** Always maintain two files:
  - `src/app.js`: Configures Express middleware, routes, and error handlers; **exports `app` without calling `.listen()`**.
  - `src/server.js`: Imports `app`, connects to database, and calls `app.listen(PORT)` for local and production running.

### ⚠️ Gotcha 2: The `NODE_OPTIONS=--experimental-vm-modules` ES Module Trap
- **The Trap:** Running Jest with native ES6 `import/export` statements without configuration flags.
- **The Error:** `SyntaxError: Cannot use import statement outside a module`.
- **The Fix:** Inject `NODE_OPTIONS=--experimental-vm-modules` via `cross-env` in your npm test script.

### ⚠️ Gotcha 3: The Hanging Jest Process (`--detectOpenHandles`)
- **The Trap:** Tests finish, but Jest hangs indefinitely.
- **The Cause:** Active database connection pools, unclosed Redis clients, or uncancelled timers keep the Node.js event loop active.
- **The Fix:** Run `jest --detectOpenHandles` to identify which file or socket is still alive, and ensure all pools are closed inside `afterAll()`.
