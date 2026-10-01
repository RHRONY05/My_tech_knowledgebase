---
tags:
  - database
  - mongodb
  - mongoose
  - backend
last_reviewed: 2026-10-02
related_notes:
  - "[[mongodb-indexing-strategies]]"
  - "[[nosql-polymorphic-schema-design]]"
---

# Mongoose Schema Extensions (Methods, Statics & Virtuals)

> **Active Recall Self-Test:**
> 1. What is the fundamental difference between an **Instance Method** (`schema.methods`) and a **Static Method** (`schema.statics`) in Mongoose?
> 2. Why does using ES6 arrow functions `() => {}` inside Mongoose methods or virtual getters break document field access?
> 3. What are Mongoose **Virtuals**, and why are they never physically written to MongoDB on-disk storage?
> 4. Why does an API endpoint returning a document omit virtual properties by default, and how does `toJSON: { virtuals: true }` fix this?

---

## 1. The Core Problem
### The Flaws of Fat Controllers & Duplicate Storage

#### Anti-Pattern A: Business Logic Duplicated Across Controllers
When computing business rules—such as checking whether a user is an admin, calculating a discounted price, or formatting an avatar URL—developers often repeat logic in every controller:
```javascript
// In controller A:
const finalPrice = product.price - (product.price * (product.discount / 100));

// In controller B:
const finalPrice = product.price * (1 - (product.discountPercentage / 100)); // Subtle discrepancy!
```
- **Domain Logic Drift:** When business formulas change, every single controller file must be audited.
- **Fat Controllers:** Controllers become bloated with low-level property calculations instead of orchestrating HTTP request/response lifecycles.

#### Anti-Pattern B: Redundant Denormalized Disk Persistence
Storing computed values directly in the database:
```javascript
{ price: 100, discountPercentage: 20, finalPrice: 80 }
```
- **Stale State:** If an admin updates `price` from 100 to 120 via an atomic update query (`$set: { price: 120 }`), `finalPrice` remains `80`, permanently corrupting financial state.

#### The Architectural Solution: Schema Extensions
1. **Virtuals:** Dynamic, computed fields calculated on-the-fly in memory when accessed. Zero disk storage, zero risk of data desynchronization.
2. **Instance Methods:** Domain logic encapsulated directly on individual document instances (`product.applyDiscount(20)`).
3. **Static Methods:** Collection-level query helpers encapsulated on the model itself (`Product.findFeaturedByCategory('Electronics')`).

---

## 2. The Mental Model

### Instance vs. Static Methods vs. Virtuals
```
┌────────────────────────────────────────────────────────────────────────┐
│                        MONGOOSE MODEL: Product                         │
│                                                                        │
│  Static Methods (schema.statics.findTopSellers)                        │
│  Operates on: THE ENTIRE COLLECTION / MODEL                            │
│  `this` points to: Product Model (can invoke .find(), .aggregate())    │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │
                                   │ Executes Query: await Product.findById(id)
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      DOCUMENT INSTANCE: product                        │
│                                                                        │
│  Persisted Fields (Saved on Disk):                                     │
│  ├── price: 100                                                        │
│  ├── discount: 20                                                      │
│  └── stock: 4                                                          │
│                                                                        │
│  Virtuals (Computed in RAM, NEVER saved to Disk):                      │
│  ├── finalPrice ────────► (price * (1 - discount/100)) = 80            │
│  └── isLowStock ────────► (stock <= 5) = true                          │
│                                                                        │
│  Instance Methods (schema.methods.decrementStock):                     │
│  Operates on: THIS SPECIFIC DOCUMENT                                   │
│  `this` points to: The single document instance                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Production Code Breakdown

### A. Defining Virtuals, Statics & Methods (`src/models/product.model.js`)
```javascript
import mongoose from 'mongoose';

const productSchema = new mongoose.Schema(
  {
    title: { type: String, required: true, trim: true },
    price: { type: Number, required: true, min: 0 },
    discountPercentage: { type: Number, default: 0, min: 0, max: 100 },
    stock: { type: Number, required: true, min: 0 },
    lowStockThreshold: { type: Number, default: 5 },
    isFeatured: { type: Boolean, default: false },
    category: { type: String, required: true },
  },
  {
    timestamps: true,
    // ⚠️ CRITICAL CONFIGURATION: Enable virtuals in serialization
    toJSON: { virtuals: true },
    toObject: { virtuals: true },
  }
);

// =========================================================================
// 1. VIRTUALS: Dynamic In-Memory Getters & Setters
// =========================================================================

// Compute final selling price dynamically
productSchema.virtual('finalPrice').get(function () {
  const discountMultiplier = (100 - this.discountPercentage) / 100;
  return Number((this.price * discountMultiplier).toFixed(2));
});

// Compute low stock status flag dynamically
productSchema.virtual('isLowStock').get(function () {
  return this.stock <= this.lowStockThreshold;
});

// =========================================================================
// 2. INSTANCE METHODS: Logic bound to a Single Document
// =========================================================================

/**
 * Decrements stock safely and enforces business invariants.
 * Uses function() syntax so `this` references the document!
 */
productSchema.methods.reserveStock = async function (quantity) {
  if (this.stock < quantity) {
    throw new Error(`INSUFFICIENT_STOCK: Only ${this.stock} items remaining`);
  }
  this.stock -= quantity;
  return await this.save(); // Saves modified document to DB
};

// =========================================================================
// 3. STATIC METHODS: Queries bound to the Global Model
// =========================================================================

/**
 * Custom collection query returning featured products for a category.
 * Uses function() syntax so `this` references the Product Model!
 */
productSchema.statics.findFeaturedByCategory = function (categoryName) {
  return this.find({
    category: categoryName,
    isFeatured: true,
    stock: { $gt: 0 },
  }).sort({ price: -1 });
};

export const Product = mongoose.model('Product', productSchema);
```

### B. Controller Invocation (`src/controllers/product.controller.js`)
```javascript
import { Product } from '../models/product.model.js';

export const purchaseProduct = async (req, res, next) => {
  try {
    const { id } = req.params;
    const { quantity } = req.body;

    // 1. Fetch document instance
    const product = await Product.findById(id);
    if (!product) {
      return res.status(404).json({ success: false, message: 'Product not found' });
    }

    // 2. Access Virtual property (computed immediately)
    console.log(`Unit price to charge: $${product.finalPrice}`);

    // 3. Execute Instance Method
    await product.reserveStock(quantity);

    // 4. Return serialized document (Virtuals are included due to toJSON: { virtuals: true })
    return res.status(200).json({
      success: true,
      data: product,
      message: 'Stock reserved successfully',
    });
  } catch (error) {
    next(error);
  }
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The ES6 Arrow Function `this` Lexical Binding Bug
- **The Trap:** Defining methods or virtuals with arrow syntax:
  ```javascript
  productSchema.virtual('finalPrice').get(() => {
    return this.price; // ❌ `this` is undefined or points to the empty module scope!
  });
  ```
- **Why:** Arrow functions bind `this` lexically at module definition time. Mongoose cannot dynamically bind `this` to the document or model instance.
- **The Fix:** **ALWAYS use traditional `function()` declarations** for all Mongoose schema methods, statics, and virtuals.

### ⚠️ Gotcha 2: The Invisible Virtual in API JSON Responses
- **The Trap:** You define virtual `finalPrice`. In your controller you return `res.json({ product })`. When the frontend inspects the payload, `finalPrice` is completely missing!
- **Why:** `res.json(doc)` invokes `JSON.stringify(doc)`, which internally calls Mongoose’s `doc.toJSON()`. By default, Mongoose excludes virtuals from JSON output to prevent accidental payload bloat.
- **The Fix:** Always pass `{ toJSON: { virtuals: true }, toObject: { virtuals: true } }` in the schema options:
  ```javascript
  const schema = new Schema({ ... }, { toJSON: { virtuals: true } });
  ```

### ⚠️ Gotcha 3: Virtuals Cannot Be Queried Directly in MongoDB
- **The Trap:** Running `Product.find({ finalPrice: { $lt: 50 } })` returns empty or errors.
- **Why:** Virtuals exist **only in Node.js process memory**. They do not exist inside MongoDB B-Tree indexes or storage files. MongoDB's query engine cannot filter on fields that are not in the database.
- **The Fix:** For queries filtering on computed fields, compute the criteria mathematically in query filters (`{ price: { $lt: 50 / multiplier } }`) or use MongoDB Aggregation pipelines (`$addFields` / `$project`).
