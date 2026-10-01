---
tags:
  - database
  - postgresql
  - migrations
  - devops
last_reviewed: 2026-10-02
related_notes:
  - "[[transactions-and-pessimistic-locking]]"
  - "[[multi-environment-db-isolation]]"
  - "[[docker-compose-production-patterns]]"
---

# PostgreSQL Database Migrations: Schema Version Control with `node-pg-migrate`

> **Active Recall Self-Test:**
> 1. Why is running manual SQL `CREATE TABLE` scripts in production dangerous, and how do **Database Migrations** act as Git version control for schemas?
> 2. Why do migration files always begin with a **Unix Timestamp** prefix (e.g. `1787331883988_init-schema.js`)?
> 3. What is the contract between `exports.up` and `exports.down`, and what catastrophic failure occurs during a rollback if `exports.down` is neglected?
> 4. How does `node-pg-migrate` use a dedicated internal table (`pgmigrations`) to track which schema patches have already been applied?

---

## 1. The Core Problem
### The Chaos of Manual Schema Alterations

In junior development, when a new feature requires an extra column or table:
1. Developer Alice opens pgAdmin or DBeaver on her local machine and runs `ALTER TABLE users ADD COLUMN phone VARCHAR(20);`.
2. Her local code works. She pushes her feature to Git.
3. Developer Bob pulls the code. His local machine has never seen that `ALTER TABLE` statement. His app crashes with `column "phone" does not exist`.
4. In production, someone has to remember to log into the live database terminal and type the exact SQL statements without typos.

#### The Failure Modes of Manual SQL
- **Schema Drift:** Local dev, staging, and production have differing columns and mismatched constraints.
- **Irreversible Mistakes:** A developer drops a column accidentally and has no standardized script to revert the database state.
- **Deployment Downtime:** Automating application deployment in Docker or CI/CD is impossible if database schema setup requires manual human typing.

#### The Architectural Solution: Automated Migrations
Migrations treat database schema transformations as **first-class version-controlled code**:
- Every change is an immutable JavaScript/SQL file stored in Git.
- Chronologically sequenced using Unix timestamps.
- **Bidirectional:** Every forward change (`up`) has an exact inverse rollback (`down`).
- Automatically executed in CI/CD and Docker startup pipelines.

---

## 2. The Mental Model

### The Migration Lifecycle & The `pgmigrations` Ledger
PostgreSQL maintains an internal ledger table named `pgmigrations`:

```
Local Repository: migrations/
├── 1710000000001_create-users.js
├── 1710000000002_create-movies.js
└── 1710000000003_add-phone-to-users.js

                                  │ npm run migrate up
                                  ▼
PostgreSQL Database:
┌────────────────────────────────────────────────────────┐
│ Ledger Table: "pgmigrations"                           │
├─────────────────────────────────────────┬──────────────┤
│ name                                    │ run_on       │
├─────────────────────────────────────────┼──────────────┤
│ 1710000000001_create-users              │ 2026-09-01   │
│ 1710000000002_create-movies             │ 2026-09-01   │
└─────────────────────────────────────────┴──────────────┘
                                  │
                                  ▼
Engine detects that 1710000000003 is NOT in pgmigrations!
└── Executes exports.up block of 1710000000003
└── Records 1710000000003 into pgmigrations table
└── Database schema is now up to date! ✅
```

---

## 3. Production Code Breakdown

### A. NPM Script Setup (`package.json`)
```json
{
  "scripts": {
    "migrate": "node-pg-migrate",
    "migrate:create": "node-pg-migrate create",
    "migrate:up": "node-pg-migrate up",
    "migrate:down": "node-pg-migrate down 1"
  }
}
```

### B. Creating a Migration File
```bash
# Creates: migrations/1787331883988_init-movie-schema.js
npm run migrate:create init-movie-schema
```

### C. Complete Reversible Migration File (`migrations/1787331883988_init-movie-schema.js`)
```javascript
/**
 * @type {import('node-pg-migrate').ColumnDefinitions | undefined}
 */
exports.shorthands = undefined;

/**
 * 1. UP (The "Do" Action): Moves database schema forward
 * @param pgm {import('node-pg-migrate').MigrationBuilder}
 */
exports.up = (pgm) => {
  // 1. Create Enums
  pgm.createType('seat_status', ['AVAILABLE', 'RESERVED', 'BOOKED']);
  pgm.createType('booking_status', ['PENDING', 'CONFIRMED', 'FAILED', 'CANCELLED']);

  // 2. Create Users Table
  pgm.createTable('users', {
    id: 'id', // Auto-incrementing primary key serial
    email: { type: 'varchar(255)', notNull: true, unique: true },
    name: { type: 'varchar(255)', notNull: true },
    google_id: { type: 'varchar(255)', unique: true },
    created_at: {
      type: 'timestamp',
      notNull: true,
      default: pgm.func('current_timestamp'),
    },
  });

  // 3. Create Seats Table with Foreign Key
  pgm.createTable('seats', {
    id: 'id',
    seat_number: { type: 'varchar(10)', notNull: true },
    price: { type: 'numeric(10,2)', notNull: true },
    status: { type: 'seat_status', notNull: true, default: 'AVAILABLE' },
    version: { type: 'integer', notNull: true, default: 0 },
    created_at: { type: 'timestamp', notNull: true, default: pgm.func('current_timestamp') },
    updated_at: { type: 'timestamp', notNull: true, default: pgm.func('current_timestamp') },
  });

  // 4. Create Indexes for High-Frequency Lookups
  pgm.createIndex('seats', 'status');
};

/**
 * 2. DOWN (The "Undo" Action): Reverts changes cleanly in exact reverse order
 * @param pgm {import('node-pg-migrate').MigrationBuilder}
 */
exports.down = (pgm) => {
  // Drop in reverse dependency order!
  pgm.dropIndex('seats', 'status');
  pgm.dropTable('seats');
  pgm.dropTable('users');
  pgm.dropType('booking_status');
  pgm.dropType('seat_status');
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Dependency Drop Order in `exports.down`
- **The Trap:** In `exports.down`, dropping the `users` table before dropping the `bookings` table that references it via Foreign Key.
- **The Crash:** PostgreSQL rejects the rollback with `ERROR: cannot drop table users because other objects depend on it`.
- **The Rule:** In `exports.down`, **always drop objects in exact reverse order of creation** (Child tables with foreign keys must be dropped before Parent tables).

### ⚠️ Gotcha 2: The Mutable Migration Trap
- **The Trap:** A developer alters an old migration file (e.g. `init-schema.js`) that has already run in production last week.
- **The Breakdown:** Production will never execute those edits because `init-schema` is already recorded in the `pgmigrations` table. Meanwhile, a new developer setting up their local machine runs the modified file and ends up with a different schema than production!
- **The Golden Rule:** **Migration files are immutable once merged/committed**. If you need to add a column or alter a type, create a brand-new migration file (`npm run migrate:create add-column-xyz`).

### ⚠️ Gotcha 3: The Auto `.env` Loading Feature
- **The Quirk:** `node-pg-migrate` automatically scans the working directory for a `.env` file containing `DATABASE_URL`.
- **The Fix:** Ensure your `.env` contains the fully qualified URI:
  ```env
  DATABASE_URL=postgres://user:password@localhost:5432/movie_booking
  ```
  This eliminates the need to pass manual CLI database flags.
