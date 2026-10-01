---
tags:
  - database
  - postgresql
  - transactions
  - concurrency
  - locking
last_reviewed: 2026-10-02
related_notes:
  - "[[database-migrations-node-pg-migrate]]"
  - "[[multi-environment-db-isolation]]"
---

# PostgreSQL Concurrency Control: ACID Transactions & Pessimistic Locking (`FOR UPDATE`)

> **Active Recall Self-Test:**
> 1. Why does running concurrent queries with `pool.query()` fail for ACID transactions, and why must transactions check out a dedicated client with `pool.connect()`?
> 2. How does a **Race Condition** occur in high-traffic inventory systems (e.g. 50 users clicking "Book Seat A1" at the exact same millisecond)?
> 3. What does `SELECT ... FOR UPDATE` do to a database row at the storage engine level, and how does it queue competing transactions?
> 4. Why is placing `client.release()` inside a `finally` block mandatory to prevent server-wide connection pool starvation?

---

## 1. The Core Problem
### The High-Traffic Double-Booking Disaster

In movie ticketing, concert bookings, or flash e-commerce sales, hundreds of users attempt to purchase the exact same limited resource at the exact same millisecond.

#### The Naive Implementation (No Locks, Standalone Queries)
```javascript
// Step 1: Check availability
const seat = await pool.query('SELECT status FROM seats WHERE id = $1', [seatId]);
if (seat.rows[0].status === 'AVAILABLE') {
  // Step 2: Reserve seat
  await pool.query('UPDATE seats SET status = $1 WHERE id = $2', ['RESERVED', seatId]);
  // Step 3: Create booking
  await pool.query('INSERT INTO bookings (user_id, seat_id) VALUES ($1, $2)', [userId, seatId]);
}
```

#### Why This Catastrophically Fails in Production
- **The Race Window:** 50 users click "Book" simultaneously.
- All 50 Node.js requests execute Step 1 concurrently. Because no updates have committed yet, **all 50 requests read `status = 'AVAILABLE'`**.
- All 50 requests enter the `if` block, run Step 2, and insert 50 booking records for a single physical seat!
- **Result:** You just oversold 1 cinema seat or plane ticket to 50 angry customers.

#### The Second Failure: Ghost Records (Lack of Atomicity)
If Step 2 succeeds (`UPDATE seats SET status = 'RESERVED'`), but Node.js crashes or loses network connectivity before Step 3 (`INSERT INTO bookings`), the seat remains marked `RESERVED` forever, but has no owner and was never paid for.

#### The Architectural Solution
1. **ACID Transactions (`BEGIN`, `COMMIT`, `ROLLBACK`):** Wrap multi-step mutations into an atomic unit of work on a single dedicated database client.
2. **Pessimistic Row-Level Locking (`SELECT ... FOR UPDATE`):** Instruct PostgreSQL to place an exclusive write lock on the target seat row. Competing transactions are forced to pause in a queue until the first transaction commits or rolls back.

---

## 2. The Mental Model

### The Connection Pool: `pool.query` vs. `pool.connect`
```
CONNECTION POOL (e.g. 10 Open Database TCP Sockets)
┌────────────────────────────────────────────────────────┐
│ Line 1  │  Line 2  │  Line 3  │  Line 4  │  Line 5 ... │
└────────────────────────────────────────────────────────┘

pool.query(): (Quick Standalone Access)
Node borrows Line 1 ──► Sends ONE query ──► Immediately releases Line 1 back to pool.
(Cannot be used for transactions because queries would scatter across different lines!)

client = await pool.connect(): (Dedicated Transaction Checkout)
Node checks out Line 1 exclusively ──► Holds line open for BEGIN, LOCK, UPDATE, COMMIT
──► Must call client.release() in `finally` to return Line 1 to the pool!
```

### Pessimistic Locking Sequence (`SELECT ... FOR UPDATE`)
```
Request #1 (User A)                         Request #2 (User B)
       │                                           │
1. BEGIN                                    1. BEGIN
       │                                           │
2. SELECT status FROM seats                 2. SELECT status FROM seats
   WHERE id = 101 FOR UPDATE                   WHERE id = 101 FOR UPDATE
       │                                           │
   ─── PostgreSQL places ROW LOCK on Seat 101! ───
   Status: 'AVAILABLE'                             │
       │                                           ▼
3. UPDATE seats SET status = 'RESERVED'     PAUSED IN DATABASE QUEUE ⏳
       │                                    (PostgreSQL blocks Request #2
4. INSERT INTO bookings (User A)             because Request #1 holds the lock)
       │                                           │
5. COMMIT ─────────────────────────────────────────┤ (Lock released on Seat 101!)
   (Transaction completed successfully)            │
                                                   ▼
                                            Request #2 unpauses and reads row:
                                            Status is now 'RESERVED'!
                                                   │
                                            Triggers business exception:
                                            "Seat already booked!"
                                                   │
                                            6. ROLLBACK ❌ (Graceful HTTP 409)
```

---

## 3. Production Code Breakdown

### Production Transaction Implementation (`src/services/booking.service.js`)
```javascript
import pool from '../config/db.js';

export const initiateSeatBooking = async (userId, seatId) => {
  // 1. Check out a dedicated client connection from the pool
  const client = await pool.connect();

  try {
    // 2. Start the atomic transaction
    await client.query('BEGIN');

    // 3. PESSIMISTIC LOCK: Lock this specific seat row exclusively
    const seatQuery = `
      SELECT id, status, price 
      FROM seats 
      WHERE id = $1 
      FOR UPDATE;
    `;
    const seatResult = await client.query(seatQuery, [seatId]);

    if (seatResult.rows.length === 0) {
      throw { status: 404, message: 'Seat not found' };
    }

    const seat = seatResult.rows[0];

    // 4. Verify invariant while holding the lock
    if (seat.status !== 'AVAILABLE') {
      throw { status: 409, message: 'Seat has already been reserved or booked by another user' };
    }

    // 5. Update seat status to RESERVED
    await client.query(
      `UPDATE seats SET status = 'RESERVED', updated_at = NOW() WHERE id = $1;`,
      [seatId]
    );

    // 6. Create booking record linking user and seat
    const bookingQuery = `
      INSERT INTO bookings (user_id, seat_id, amount, status)
      VALUES ($1, $2, $3, 'PENDING')
      RETURNING id, status, created_at;
    `;
    const bookingResult = await client.query(bookingQuery, [userId, seatId, seat.price]);

    // 7. Commit changes permanently to disk
    await client.query('COMMIT');

    return {
      success: true,
      booking: bookingResult.rows[0],
    };
  } catch (error) {
    // ⚠️ ATOMICITY GUARANTEE: Roll back all changes if any error occurs
    await client.query('ROLLBACK');
    throw error;
  } finally {
    // ⚠️ CRITICAL INVARIANT: Always release client back to pool to prevent connection leaks!
    client.release();
  }
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Connection Pool Starvation Leak
- **The Trap:** Forgetting `client.release()` inside a `finally` block when an error is thrown.
- **The Catastrophe:** If your pool has a max capacity of 10 connections, and 10 requests throw an error without calling `client.release()`, all 10 lines remain permanently locked. The 11th request hangs indefinitely waiting for a free line until all subsequent API requests timeout (`timeout exceeded when acquiring client from pool`).
- **The Fix:** **ALWAYS place `client.release()` inside a `finally` block**.

### ⚠️ Gotcha 2: The Deadlock Trap (Lock Ordering)
- **The Trap:** If User 1 books Seat A then Seat B, while User 2 books Seat B then Seat A at the same millisecond.
  - Transaction 1 locks Seat A and waits for Seat B.
  - Transaction 2 locks Seat B and waits for Seat A.
- **The Crash:** Neither can proceed. PostgreSQL detects a circular dependency and terminates one with `ERROR: deadlock detected`.
- **The Fix:** When locking multiple rows in a single transaction, always sort the resource IDs in a deterministic order (e.g. `ORDER BY id ASC`) before applying `FOR UPDATE`.

### ⚠️ Gotcha 3: The 1-to-1 vs 1-to-Many Audit Trail Decision
- **The Trap:** Setting a `UNIQUE(seat_id)` constraint on the `bookings` table.
- **The Flaw:** If User 1 reserves Seat 10, their payment OTP expires, and the booking is marked `FAILED`. If `seat_id` is unique, User 2 can never book Seat 10 again!
- **The Fix:** Allow a 1-to-Many relationship between `seats` and `bookings` so historical failed/cancelled attempts are preserved, and enforce single-ownership by verifying `status = 'AVAILABLE'` under row lock.
