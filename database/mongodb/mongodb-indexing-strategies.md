---
tags:
  - database
  - mongodb
  - performance
  - indexing
last_reviewed: 2026-10-02
related_notes:
  - "[[nosql-polymorphic-schema-design]]"
  - "[[mongodb-relationships-and-populate]]"
---

# MongoDB Indexing Strategies & Query Optimization

> **Active Recall Self-Test:**
> 1. What is the execution difference between a `COLLSCAN` (Collection Scan) and an `IXSCAN` (Index Scan) in terms of computational complexity and disk I/O?
> 2. How does the B-Tree data structure allow MongoDB to achieve $O(\log n)$ lookup times on indexed fields?
> 3. What is the "Equality, Sort, Range" (ESR) rule when designing compound indexes?
> 4. Why does adding an index speed up queries but degrade write throughput (`insert`, `update`, `delete`)?

---

## 1. The Core Problem
### The Unindexed Database Bottleneck

In development with 50 test documents, all queries execute in under 2 milliseconds. But when an application reaches production with 1,000,000 documents:

#### The Naive Reality: Collection Scan (`COLLSCAN`)
Without an index on a queried field (e.g. `category` or `slug`):
- The database engine must perform a sequential linear scan across **every single document on disk** from start to finish.
- **Complexity:** $O(n)$ where $n = 1,000,000$.
- **Disk I/O & Memory:** Millions of bytes are read from disk into the database buffer pool, evicting frequently cached data and pegging CPU utilization at 100%.
- **Response Time:** A simple query takes 2,000ms–5,000ms, causing cascading request timeouts in your Node.js API.

#### The Architectural Solution: B-Tree Indexes
An index is a specialized, sorted, in-memory data structure (typically a B-Tree) that holds a subset of document fields alongside a direct pointer (`RecordId`) to the on-disk document location:
- **Complexity:** $O(\log n)$ logarithmic binary search.
- **Lookup Cost:** Looking up a record in a 1,000,000 document collection requires only ~20 pointer operations instead of 1,000,000 scans. Query time drops from 2,000ms to **0.5ms (1000x to 4000x faster)**.

---

## 2. The Mental Model

### Collection Scan vs. B-Tree Index Scan
```
COLLSCAN (Sequential Disk Traversal):
[ Doc 1 ] ──► [ Doc 2 ] ──► [ Doc 3 ] ──► ... ──► [ Doc 1,000,000 ]
Scans all 1,000,000 docs sequentially. Time: ~2000ms ❌

──────────────────────────────────────────────────────────────────

IXSCAN (Balanced B-Tree Traversal in RAM):
                         ["Electronics"]
                        /               \
              ["Clothing"]              ["Footwear"]
             /            \             /          \
      ["Accessories"]   ["Books"]   ["Home"]     ["Sports"]
             │              │           │            │
       [RecordPtrs]   [RecordPtrs] [RecordPtrs]  [RecordPtrs]

B-Tree branch navigation. Traverses 3-4 nodes directly to pointers.
Time: ~0.8ms ✅
```

### The ESR (Equality, Sort, Range) Golden Rule
When creating a multi-field compound index, field ordering dictates whether MongoDB can use the index for both filtering and sorting:
1. **E - Equality:** Fields queried with exact matches (`{ status: "active" }`) go **FIRST**.
2. **S - Sort:** Fields used for ordering (`{ createdAt: -1 }`) go **SECOND**.
3. **R - Range:** Fields queried with ranges (`{ price: { $gte: 50, $lte: 200 } }`) go **LAST**.

If you place a Range field before a Sort field, MongoDB can filter by range, but must perform an expensive in-memory sort (`SORT_KEY_GENERATOR`) on the matching subset.

---

## 3. Production Code Breakdown

### A. Mongoose Schema Index Definitions (`src/models/product.model.js`)
```javascript
import mongoose from 'mongoose';

const productSchema = new mongoose.Schema(
  {
    title: { type: String, required: true, trim: true },
    slug: {
      type: String,
      required: true,
      unique: true, // Creates a UNIQUE single-field index
      lowercase: true,
      index: true,
    },
    category: { type: String, required: true, index: true }, // Single-field index
    brand: { type: String, required: true },
    price: { type: Number, required: true },
    isFeatured: { type: Boolean, default: false },
    inStock: { type: Boolean, default: true },
  },
  { timestamps: true }
);

// =========================================================================
// COMPOUND INDEXES (Applying ESR Rule)
// =========================================================================

// Query: Find products by category, filter in-stock, sort by price (Equality -> Sort)
productSchema.index({ category: 1, inStock: 1, price: -1 });

// Query: Find featured products in category, with price range (Equality -> Range)
productSchema.index({ isFeatured: 1, category: 1, price: 1 });

// TEXT INDEX for search queries
productSchema.index({ title: 'text', brand: 'text' });

export const Product = mongoose.model('Product', productSchema);
```

### B. Analyzing Query Execution with `explain()` (`src/scripts/analyzeQuery.js`)
```javascript
import { Product } from '../models/product.model.js';

export const inspectQueryPerformance = async () => {
  // Run query with executionStats profiling
  const explanation = await Product.find({
    category: 'Footwear',
    inStock: true,
    price: { $gte: 50, $lte: 150 },
  })
    .sort({ price: -1 })
    .explain('executionStats');

  const stats = explanation.executionStats;

  console.log('Query Performance Report:');
  console.log(`- Winning Stage: ${explanation.queryPlanner.winningPlan.stage}`);
  console.log(`- Total Docs Examined (nExamined): ${stats.totalDocsExamined}`);
  console.log(`- Total Docs Returned (nReturned): ${stats.nReturned}`);
  console.log(`- Total Execution Time: ${stats.executionTimeMillis} ms`);

  // Target Invariant: nExamined === nReturned
  if (stats.totalDocsExamined === stats.nReturned) {
    console.log('✅ Query is fully covered / optimally indexed.');
  } else if (stats.totalDocsExamined > stats.nReturned * 10) {
    console.warn('⚠️ Warning: High doc examination overhead. Revisit index design.');
  }
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Index Write Penalty (Write Amplification)
- **The Trap:** A developer indexes 15 fields on a collection to make every conceivable query fast. While reads are instant, `insert`, `update`, and `delete` throughput slows to a crawl.
- **Why:** Every time a document is created or modified, MongoDB must update the primary document store PLUS all 15 B-Tree index structures in RAM and sync them to disk journal files.
- **The Fix:** Index only fields used in high-frequency queries, sort operations, and unique constraints. Audit unused indexes using `db.collection.aggregate([{ $indexStats: {} }])`.

### ⚠️ Gotcha 2: The Compound Index Prefix Rule
- **The Trap:** You create a compound index `{ category: 1, brand: 1, price: 1 }`. Then you execute a query searching only for `{ brand: "Nike" }`. MongoDB ignores your index and executes a full collection scan (`COLLSCAN`)!
- **Why:** Compound indexes can only be utilized if the query includes the **leading prefix** of the index.
  - `{ category }` ✅ (Leading prefix)
  - `{ category, brand }` ✅ (Leading prefix)
  - `{ category, brand, price }` ✅ (Full index)
  - `{ brand }` ❌ (No leading prefix; cannot traverse B-Tree root)
  - `{ price }` ❌ (Cannot traverse without prefix)

### ⚠️ Gotcha 3: In-Memory Sort Overflow (32 MB Limit)
- **The Trap:** When executing `.sort({ createdAt: -1 })` on a query that returns 50,000 documents without an index supporting the sort, MongoDB attempts an in-memory sort.
- **The Crash:** If the dataset exceeds 32 MB in RAM, MongoDB aborts with `Executor error during find command: Sort operation used more than the maximum 33554432 bytes of RAM`.
- **The Fix:** Ensure your compound index includes the sort field according to the ESR rule so sorting is handled directly by the pre-ordered B-Tree structure.
