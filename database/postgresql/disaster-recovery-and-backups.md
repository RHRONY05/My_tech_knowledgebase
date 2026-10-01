---
tags:
  - database
  - postgresql
  - devops
  - disaster-recovery
  - security
last_reviewed: 2026-10-02
related_notes:
  - "[[transactions-and-pessimistic-locking]]"
  - "[[docker-compose-production-patterns]]"
  - "[[self-hosted-vps-vs-paas]]"
---

# PostgreSQL Disaster Recovery: Logical Backups, `pg_dump` & Automated Retention

> **Active Recall Self-Test:**
> 1. Why does copying raw `/var/lib/postgresql/data` files from a running database risk severe data corruption, and how does `pg_dump` guarantee a 100% consistent transactional snapshot?
> 2. What is the industry **"3-2-1" Backup Rule** for critical stateful systems?
> 3. Why must `pg_dump` include the `--clean --if-exists` flags when generating restoration scripts?
> 4. What are the 6 essential components of an enterprise-grade automated backup script?

---

## 1. The Core Problem
### The Irreplaceability of Database State

In modern containerized deployments:
- **Code is Disposable:** If an Express or Nginx container crashes or is deleted, Docker recreates it from a read-only image in seconds.
- **Database State is Irreplaceable:** If your PostgreSQL disk volume is corrupted, deleted, or ransomed, every customer record, seat reservation, and transaction history is permanently destroyed.

#### Why Copying Raw Files Fails:
A common beginner mistake is copying `/var/lib/postgresql/data` directly while PostgreSQL is running:
- PostgreSQL writes pages to disk continuously, manages write-ahead logs (WAL), and buffers dirty blocks in RAM.
- Copying raw files mid-write captures half-written blocks and uncommitted transactions, resulting in an unrecoverable corrupted database on restore.

#### The Architectural Solution: Logical Snapshot Backups (`pg_dump`)
- `pg_dump` opens a read-only transaction snapshot at a specific point in time.
- It serializes tables, schemas, and rows into plain, clean SQL text statements (`CREATE TABLE`, `INSERT INTO`).
- Even while hundreds of concurrent users are reserving seats, `pg_dump` produces a 100% consistent, uncorrupted backup without locking the database against reads or writes.

---

## 2. The Mental Model

### The 3-2-1 Backup Strategy
```
┌────────────────────────────────────────────────────────────────────────┐
│                        THE 3-2-1 BACKUP STANDARD                       │
├────────────────────────────────────────────────────────────────────────┤
│ 3 COPIES OF DATA:                                                      │
│ ├── 1. Primary Live Database (PostgreSQL Production Volume)            │
│ ├── 2. Local Server Snapshot (Gzipped SQL on VPS host disk)            │
│ └── 3. Offsite Cloud Backup (Exported to AWS S3 / Cloudflare R2)       │
├────────────────────────────────────────────────────────────────────────┤
│ 2 DIFFERENT MEDIA:                                                     │
│ ├── Local Block Storage (Fast restore)                                 │
│ └── Cloud Object Storage (Resilient to total datacenter loss)          │
├────────────────────────────────────────────────────────────────────────┤
│ 1 COPY STORED OFFSITE:                                                 │
│ └── Geographically separated region (Protects against provider outage) │
└────────────────────────────────────────────────────────────────────────┘
```

### The 6-Point Backup Script Checklist
```
┌────────────────────────────────────────────────────────────────────────┐
│                   THE BACKUP SCRIPT CHECKLIST                          │
│                                                                        │
│  1. Security First      ➔ Add backups/ & *.sql to .gitignore           │
│  2. Fail-Fast Check     ➔ Verify container/DB is alive before dumping  │
│  3. Unique Timestamps   ➔ backup_YYYY-MM-DD_HH-mm-ss (no overwrites)   │
│  4. Clean Overwrite     ➔ Use --clean --if-exists in pg_dump           │
│  5. Compress & Delete   ➔ gzip to .sql.gz & delete raw uncompressed    │
│  6. Auto-Retention      ➔ Prune files older than 7 (or 30) days        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Production Code Breakdown

### A. Production Bash Backup Script (`scripts/backup.sh`)
```bash
#!/bin/bash
set -e

# =========================================================================
# CONFIGURATION
# =========================================================================
CONTAINER_NAME="movie_booking_prod_db"
DB_NAME="movie_booking"
DB_USER="postgres"
BACKUP_DIR="/var/backups/postgres"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
BACKUP_FILE="${BACKUP_DIR}/backup_${DB_NAME}_${TIMESTAMP}.sql"
RETENTION_DAYS=7

# Step 1: Ensure backup directory exists
mkdir -p "$BACKUP_DIR"

# Step 2: PRE-FLIGHT CHECK: Ensure database container is alive and ready
echo "🔍 Checking database container status..."
if ! docker ps --filter "name=${CONTAINER_NAME}" --filter "status=running" | grep -q "${CONTAINER_NAME}"; then
    echo "❌ ERROR: Database container ${CONTAINER_NAME} is not running! Aborting backup."
    exit 1
fi

# Step 3: EXECUTE LOGICAL DUMP
# --clean: Adds DROP TABLE before CREATE TABLE
# --if-exists: Prevents errors during restoration if a table doesn't exist
echo "📦 Starting database dump for '${DB_NAME}'..."
docker exec "$CONTAINER_NAME" pg_dump \
    -U "$DB_USER" \
    -d "$DB_NAME" \
    --clean \
    --if-exists \
    --no-owner \
    --no-privileges > "$BACKUP_FILE"

# Step 4: COMPRESS & PURGE RAW SQL
echo "🗜️ Compressing backup file..."
gzip "$BACKUP_FILE"
echo "✅ Compressed snapshot created: ${BACKUP_FILE}.gz"

# Step 5: AUTOMATED RETENTION PRUNING (Delete backups older than 7 days)
echo "🧹 Pruning backups older than ${RETENTION_DAYS} days..."
find "$BACKUP_DIR" -name "backup_${DB_NAME}_*.sql.gz" -mtime +"$RETENTION_DAYS" -exec rm {} \;

echo "🎉 Backup workflow completed successfully at $(date)!"
```

### B. Disaster Recovery Restoration Script (`scripts/restore.sh`)
```bash
#!/bin/bash
set -e

TARGET_BACKUP="$1"
CONTAINER_NAME="movie_booking_prod_db"
DB_NAME="movie_booking"
DB_USER="postgres"

if [ -z "$TARGET_BACKUP" ]; then
    echo "❌ Usage: ./restore.sh <path-to-backup.sql.gz>"
    exit 1
fi

if [ ! -f "$TARGET_BACKUP" ]; then
    echo "❌ Backup file not found: $TARGET_BACKUP"
    exit 1
fi

echo "⚠️ WARNING: Restoring will overwrite existing data in '${DB_NAME}'!"
read -p "Are you sure you want to proceed? (y/N): " CONFIRM
if [[ "$CONFIRM" != "y" && "$CONFIRM" != "Y" ]]; then
    echo "Restoration aborted."
    exit 0
fi

# Decompress and stream SQL directly into container via standard input
echo "🔄 Restoring database snapshot..."
gunzip -c "$TARGET_BACKUP" | docker exec -i "$CONTAINER_NAME" psql -U "$DB_USER" -d "$DB_NAME"

echo "✅ Database restored successfully from ${TARGET_BACKUP}!"
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Zero-Byte Silent Failure Trap
- **The Trap:** Running `docker exec ... pg_dump > backup.sql` without verifying that the container is alive.
- **The Disaster:** If the container stopped or authentication failed, bash still creates `backup.sql` as a completely empty **0-byte file**. You assume you have backups, until a disaster strikes and you realize you have been saving empty text files for months!
- **The Fix:** Always verify container liveness first and check that the resulting file size is greater than zero:
  ```bash
  [ -s "$BACKUP_FILE" ] || { echo "Empty backup!"; exit 1; }
  ```

### ⚠️ Gotcha 2: The "Relation Already Exists" Restore Failure
- **The Trap:** Running `pg_dump` without `--clean --if-exists`.
- **The Breakdown:** When restoring over an existing database, the SQL script tries to execute `CREATE TABLE users;`. Since the table already exists, PostgreSQL aborts with `ERROR: relation "users" already exists`, leaving the restore half-executed.
- **The Fix:** Always include `--clean --if-exists` so the script cleanly drops old tables before recreating them.

### ⚠️ Gotcha 3: The Untested Backup Fallacy
- **The Rule:** **A backup that has never been restored is NOT a backup—it is merely a wish.**
- **The Fix:** Schedule automated quarterly or monthly recovery drills where a script boots an isolated test container and restores the latest snapshot, verifying table row counts and integrity.
