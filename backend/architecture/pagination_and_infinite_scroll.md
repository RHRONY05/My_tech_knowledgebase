---
tags: [backend, architecture, pagination, performance, database, api-design]
last_reviewed: 2026-10-04
related_notes: ["[[backend/architecture/production-backend-5-pillars]]", "[[database/prisma/prisma_fundamentals]]"]
---

# Pagination & Infinite Scroll Feeds — Architecture & Production Mental Model

### 1. The Core Problem
*Why is unbounded data fetching a fatal anti-pattern? What breaks at runtime without pagination?*

1. **Database & Memory Exhaustion:** If a database table grows to 100,000 records, executing `SELECT * FROM items` forces PostgreSQL to scan massive disk blocks, allocates hundreds of megabytes of RAM in Node.js to deserialize rows, and risks an Out-Of-Memory (`heap out of memory`) crash.
2. **Network Bandwidth Saturation:** Serializing 50,000 JSON items produces an HTTP payload exceeding 20MB. Over mobile cellular connections, this blocks bandwidth and causes multi-second API latency.
3. **Browser DOM Freezes:** A web or mobile browser attempting to create and paint 10,000 DOM nodes or list cards will drop frame rates to zero, freezing the user's interface.

**The Architectural Solution:** Slice datasets into small, predictable batches (e.g. 10 to 50 items per network request).

---

### 2. The Mental Model

There are two primary paradigms for delivering paginated data to a client:

```text
A. TRADITIONAL E-COMMERCE PAGINATION (PAGE REPLACEMENT)
=======================================================
User Clicks "Page 2" -> Frontend replaces current items with new items

[Page 1: Items 1..10]  ---> Click [2] --->  [Page 2: Items 11..20]
(Items 1..10 are removed from DOM, memory footprint stays constant)


B. INFINITE FEED (ARRAY CONCATENATION + LOOKAHEAD BUFFER)
========================================================
User scrolls near bottom -> Frontend silently pre-fetches and appends

[Posts 1..10]
      | (User scrolls to Post #7 -> Threshold triggers background fetch)
      v
[Posts 1..10] + [Posts 11..20]
      | (User scrolls to Post #17 -> Threshold triggers background fetch)
      v
[Posts 1..10] + [Posts 11..20] + [Posts 21..30]
(List grows continuously; pre-fetching hides network latency)
```

---

### 3. Detailed Topic Breakdown

#### 3.1 Offset-Based Pagination (`skip` & `take`)
Used primarily in **e-commerce catalogs, admin tables, and search engines** where users want to see exact page numbers and jump between pages.

- **`take` (SQL `LIMIT`):** The number of items to return (e.g. `limit = 10`).
- **`skip` (SQL `OFFSET`):** How many items to bypass before taking.

```text
Formula: skip = (page - 1) * limit
```

| Requested Page | Skip Calculation | SQL Generated | Items Delivered |
| :--- | :--- | :--- | :--- |
| **Page 1** | `(1 - 1) * 10 = 0` | `LIMIT 10 OFFSET 0` | Items 1 to 10 |
| **Page 2** | `(2 - 1) * 10 = 10` | `LIMIT 10 OFFSET 10` | Items 11 to 20 |
| **Page 3** | `(3 - 1) * 10 = 20` | `LIMIT 10 OFFSET 20` | Items 21 to 30 |

---

#### 3.2 The Two-Query Parallel Execution Pattern
Because `findMany({ take: 10 })` only retrieves 10 records, the backend does not inherently know how many total items or pages exist. 

To provide the frontend with complete pagination controls, the backend must execute **two queries concurrently**:

```typescript
const [totalItems, items] = await Promise.all([
  // Query 1: Fast headcount of matching rows (does not read row data)
  prisma.product.count({ where }),

  // Query 2: Retrieve the exact slice of data for the requested page
  prisma.product.findMany({
    where,
    orderBy: { createdAt: 'desc' },
    skip: (page - 1) * limit,
    take: limit,
  }),
]);
```

`Promise.all` runs both queries concurrently over the database connection pool, eliminating sequential waiting.

---

#### 3.3 The Pagination Metadata Object
The backend calculates metadata so the frontend UI can render navigation controls without guessing:

```typescript
const totalPages = Math.ceil(totalItems / limit);

const pagination = {
  totalItems,                             // e.g. 48 total matching rows
  totalPages,                             // Math.ceil(48 / 10) = 5 pages
  currentPage: page,                      // e.g. 2
  pageSize: limit,                        // e.g. 10 items per page
  hasNextPage: page < totalPages,         // true if more pages remain
  hasPrevPage: page > 1,                  // true if not on the first page
};
```

- **`Math.ceil(totalItems / limit)`:** Prevents dropping remainder items. 48 items at 10 per page produces 5 pages (10, 10, 10, 10, 8).
- **`hasNextPage` / `hasPrevPage`:** Explicit boolean flags allowing the frontend to enable or disable `< Previous` and `Next >` buttons without duplicate math on the client.

---

#### 3.4 Cursor-Based Pagination (Social Media Feeds)
While offset pagination works well for static catalogs, it introduces a severe bug in fast-moving social feeds: **The Duplicate Item Drift Bug**.

```text
THE DUPLICATE DRIFT BUG WITH OFFSET:
1. User loads Page 1 (Posts 1 to 10).
2. While reading, another user creates 2 brand-new posts.
3. User requests Page 2 (skip: 10).
4. Because 2 new posts were inserted at the top, Post #9 and #10 shifted down to positions #11 and #12.
5. Result: The user sees Post #9 and #10 a second time!
```

#### The Solution: Keyset / Cursor Pagination
Instead of an offset number, the client sends a **Cursor**—the unique identifier of the last item seen:

```typescript
// Cursor Query: "Give me the 10 posts created AFTER post_999"
const posts = await prisma.post.findMany({
  take: 10,
  skip: 1, // Skip the anchor record itself
  cursor: {
    id: lastSeenId,
  },
  orderBy: { createdAt: 'desc' },
});
```

Because the query is anchored to a specific record ID, new posts inserted at the top do not shift the cursor position. Duplicate items are completely eliminated.

---

#### 3.5 Lookahead Buffering & `IntersectionObserver`
How do infinite feeds (Instagram, Twitter) load new items without showing a loading spinner on every scroll?

They trigger the background request **before** the user reaches the end:

```text
SCREEN VIEWPORT (User's View):
+------------------------------------+
|  Post #6                           |
|  Post #7 (User is reading here)    |  <-- Threshold reached! Frontend fires background fetch:
|  Post #8                           |      GET /api/feed?cursor=post_10&limit=10
+------------------------------------+
|  Post #9                           |
|  Post #10 (Bottom of current list) |
+------------------------------------+
```

1. **Pre-fetching:** While the user reads Post #7, Posts #11 through #20 are fetched and buffered in the background (~150ms).
2. **Seamless Arrival:** By the time the user scrolls to Post #10, the new items are already in memory and appended to the screen.
3. **Why fast scrolling reveals the spinner:** Normal scrolling takes 2–3 seconds (the network wins). Violent flick-scrolling reaches the bottom in 0.1 seconds, outpacing the 0.2-second network latency and exposing the spinner.

#### The Modern Tool: Native `IntersectionObserver`
Instead of polling CPU-heavy `window.onscroll` events, modern frontends place an invisible sentinel element near the bottom:

```typescript
// Client-side Infinite Scroll Sentinel
const observer = new IntersectionObserver(
  (entries) => {
    if (entries[0].isIntersecting) {
      // User is 500px from the bottom! Fetch next batch now.
      fetchNextPage();
    }
  },
  { rootMargin: '500px' } // Pre-fetch 500 pixels BEFORE reaching the sentinel
);
```

---

#### 3.6 Client-Side Caching (Instant Back-Navigation)
In modern single-page applications (using Redux Toolkit Query or TanStack Query):
- When Page 1 is fetched, the response is stored in client-side memory under a unique cache key (`getItems?page=1`).
- When the user navigates from Page 1 -> Page 2 -> back to Page 1:
  - The client serves Page 1 from memory cache in **0 milliseconds**.
  - No network request is required, and no loading spinner is shown.

---

### 4. Top Gotchas & Pitfalls to Avoid

#### ❌ Gotcha 1: The High-Offset Performance Cliff
- **The Issue:** Running `SELECT * FROM items OFFSET 1000000 LIMIT 10` on large databases is slow. PostgreSQL must read and discard 1,000,000 rows in memory before returning the 10 requested rows.
- **The Fix:** For massive datasets (millions of rows) or continuous feeds, use **Cursor-Based Pagination** (`WHERE id > last_seen_id ORDER BY id ASC LIMIT 10`), which jumps directly to the index position in logarithmic time `O(log N)` without scanning preceding rows.

#### ❌ Gotcha 2: Integer Truncation in Total Pages
- **The Issue:** Calculating `totalPages = Math.floor(totalItems / limit)` or `parseInt(totalItems / limit)`.
- **The Consequence:** If `totalItems = 25` and `limit = 10`, integer division produces `2`. The last 5 items become permanently unreachable.
- **The Fix:** Always use `Math.ceil(totalItems / limit)` so any remainder creates an additional page.

#### ❌ Gotcha 3: The Scroll Event Loop Freeze
- **The Issue:** Attaching event listeners to `window.addEventListener('scroll', handler)` to detect the bottom of a feed.
- **The Consequence:** The scroll event fires 60+ times per second on high-refresh-rate displays, triggering layout thrashing and freezing animations.
- **The Fix:** Use the browser's native `IntersectionObserver` with a `rootMargin` buffer, running asynchronously off the main thread.
