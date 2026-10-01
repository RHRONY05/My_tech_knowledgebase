---
tags:
  - database
  - redis
  - caching
  - devops
  - docker
last_reviewed: 2026-10-02
related_notes:
  - "[[devops/docker/docker-compose-production-patterns]]"
  - "[[backend/architecture/production-backend-5-pillars]]"
  - "[[database/postgresql/transactions-and-pessimistic-locking]]"
---

# Redis Architecture: Sub-Millisecond Caching, TTL & Multi-Container Orchestration

> [!CHECKLIST] Active Recall Self-Test
> - [ ] Why does storing ephemeral data (e.g. AI chat sessions, rate limits) in PostgreSQL cause disk I/O bottlenecks compared to Redis?
> - [ ] What is the fundamental latency difference between in-memory RAM access (<1ms) and disk/SSD storage (5–25ms)?
> - [ ] How does Time-To-Live (TTL) with `SETEX` eliminate the need for scheduled cron jobs to purge expired session data?
> - [ ] In Docker Compose, why can containers communicate via service names (`redis:6379`) while the host connects via `localhost:6379`?
> - [ ] Why does vector similarity search for AI embeddings require `pgvector/pgvector:pg16` instead of standard `postgres:16`?

---

## 1. The Core Problem: Disk Latency & Ephemeral Data Bloat

Traditional relational and document databases (PostgreSQL, MongoDB) are engineered for **durable, ACID-compliant disk storage**. When applied to high-velocity, short-lived workloads, they fail in three critical areas:

1. **Disk I/O Bottlenecks for Ephemeral State:** Storing rapid user session heartbeats, shopping cart drafts, or multi-turn AI chatbot context in PostgreSQL forces disk head movement, write-ahead log (WAL) synchronization, and page flushes. Reads take 5–25ms instead of sub-millisecond speeds.
2. **Table Bloat & Manual Cron Purging:** Databases do not have native, per-row auto-destruction timers. Without Redis, developers must write complex cron jobs or database triggers (`DELETE FROM sessions WHERE expires_at < NOW()`), which cause table fragmentation and lock contention under heavy write traffic.
3. **The Local Host OS Collision Disaster:** Installing PostgreSQL, Redis, and specialized C extensions (like `pgvector` for 768-dimensional AI embeddings) directly onto host developer machines causes port conflicts, OS pollution, and the infamous "works on my machine" deployment failure.

---

## 2. The Mental Model: In-Memory RAM vs. Disk & Container Networking

Redis stores all data structures directly in **System Memory (RAM)**, achieving sub-millisecond latency. Docker Compose connects your relational database (PostgreSQL with `pgvector`), in-memory cache (Redis), and administrative tooling (RedisInsight) over an isolated virtual bridge network.

```
Host OS (Developer Laptop / VPS)
       │
       ├──── Browser / GUI ──(Port 5432)───► [ shopmate_postgres (pgvector:pg16) ]
       │                                     └── Volume: postgres_data (Local Disk / SSD)
       │
       ├──── Browser (localhost:5540) ──────► [ shopmate_redis_insight (Web GUI) ]
       │                                     │
       │                              (Internal DNS: redis:6379)
       │                                     ▼
       └──── Node API / CLI ──(Port 6379)──► [ shopmate_redis (In-Memory RAM) ]
                                             └── Volume: redis_data (AOF/RDB Persistence)
```

### The Cache-Aside Access Pattern

```
API Request ──▶ 1. Check Redis Cache (GET key)
                      │
        ┌─────────────┴─────────────┐
     Cache HIT                   Cache MISS
        │                           │
        ▼                           ▼
Return data (<1ms)            2. Query PostgreSQL (15-30ms)
                                    │
                                    ▼
                              3. Write to Redis with TTL (SETEX key, 3600)
                                    │
                                    ▼
                              Return data to client
```

---

## 3. Production Code Breakdown

### A. Production Multi-Container Orchestration (`docker-compose.yml`)

Combines PostgreSQL 16 bundled with `pgvector`, Redis with memory management, and RedisInsight for developer inspection:

```yaml
version: "3.8"

services:
  # Relational Database with AI Vector Extension
  postgres:
    image: pgvector/pgvector:pg16 # Pre-compiled with pgvector for cosine similarity (<=>)
    container_name: shopmate_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-shopmate_db}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      timeout: 5s
      retries: 5

  # In-Memory Cache & Session Store
  redis:
    image: redis:7-alpine
    container_name: shopmate_redis
    restart: unless-stopped
    command: >
      redis-server 
      --maxmemory 512mb 
      --maxmemory-policy allkeys-lru 
      --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Redis Web GUI (Development Only)
  redis-insight:
    image: redis/redisinsight:latest
    container_name: shopmate_redis_insight
    restart: unless-stopped
    ports:
      - "5540:5540"
    depends_on:
      redis:
        condition: service_healthy
    volumes:
      - redis_insight_data:/data

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  redis_insight_data:
    driver: local
```

### B. Battle-Tested Redis Client with Reconnection Logic (`src/config/redis.ts`)

```typescript
// src/config/redis.ts
import Redis from "ioredis";

const REDIS_URL = process.env.REDIS_URL || "redis://localhost:6379";

export const redis = new Redis(REDIS_URL, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: true, // Controlled connection during bootstrap
  retryStrategy(times) {
    const delay = Math.min(times * 100, 3000);
    return delay;
  },
});

redis.on("error", (err) => {
  console.error("⚠️ Redis connection error:", err);
});

redis.on("connect", () => {
  console.log("⚡ Connected to Redis instance");
});

/**
 * Health check utility for fail-fast server bootstrap
 */
export async function checkRedisConnection(): Promise<void> {
  await redis.connect();
  const pong = await redis.ping();
  if (pong !== "PONG") {
    throw new Error("Redis ping response invalid");
  }
}
```

### C. Cache-Aside Utility with Self-Expiring Keys (TTL)

```typescript
// src/utils/cacheService.ts
import { redis } from "../config/redis.js";

export class CacheService {
  /**
   * Stores value with automatic Time-To-Live (TTL) expiration.
   * @param key Cache key identifier
   * @param value Serializable data object
   * @param ttlSeconds Expiration window in seconds (e.g. 3600 = 1 hour)
   */
  static async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    const serialized = JSON.stringify(value);
    // SETEX: Atomic set + expiration in a single round-trip
    await redis.setex(key, ttlSeconds, serialized);
  }

  /**
   * Retrieves and deserializes cached value.
   */
  static async get<T>(key: string): Promise<T | null> {
    const data = await redis.get(key);
    if (!data) return null;
    return JSON.parse(data) as T;
  }

  /**
   * Invalidates specific cache key.
   */
  static async del(key: string): Promise<void> {
    await redis.del(key);
  }

  /**
   * Cache-Aside orchestrator: Returns cached value or fetches and caches fresh result.
   */
  static async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttlSeconds = 900
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null) {
      return cached; // Cache HIT (<1ms)
    }

    // Cache MISS: Query slow data source
    const freshData = await fetcher();
    if (freshData !== null && freshData !== undefined) {
      await this.set(key, freshData, ttlSeconds);
    }
    return freshData;
  }
}
```

---

## 4. Production Gotchas & Best Practices

### 1. The Missing Eviction Policy OOM Crash
By default, Redis runs with `noeviction`. Once RAM fills up, any subsequent `SET` command throws `OOM command not allowed when used memory > 'maxmemory'`.
- **Production Requirement:** Always configure `--maxmemory` and an eviction policy like `--maxmemory-policy allkeys-lru` (Least Recently Used) in production, ensuring Redis automatically drops old keys when RAM is saturated.

### 2. Exposing Redis GUI in Production
`redis/redisinsight` on port 5540 has zero default authentication in many setups. Exposing this port to the public internet on a cloud VPS allows anyone to browse, modify, and delete all session keys and user data.
- **Rule:** Never expose port 5540 in production; keep Redis behind your private Docker bridge network.

### 3. Cache Stampede (Dog-piling)
When a high-traffic cache key expires, 500 concurrent requests may experience a cache miss at the exact same millisecond, hammering the database with identical expensive queries.
- **Mitigation:** Use probabilistic early expiration or mutual exclusion locking (mutex) on cache misses for critical hot keys.
