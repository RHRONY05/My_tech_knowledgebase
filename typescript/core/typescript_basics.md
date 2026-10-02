---
tags:
  - typescript
  - javascript
  - type-safety
  - architecture
last_reviewed: 2026-10-02
related_notes:
  - "[[typescript/core/tsconfig_conecpts]]"
  - "[[typescript/zod/zod-schema-validation]]"
---

# TypeScript — Production Cheat Sheet & Mental Model

### 1. The Core Problem
*Why did standard JavaScript fail here? What exact engineering problem made this tool necessary?*

Standard JavaScript is **dynamically typed**, meaning types are only evaluated at runtime inside the Node.js/browser engine. This creates three critical production risks:
1. **Silent Data Corruption:** A string `"85"` and number `15` passed into a pricing calculation silently concatenate into `"8515"` instead of `100`, resulting in incorrect database records or payment gateway amounts.
2. **Missing Property Crashes:** Calling methods on undefined fields (`item.name.toUpperCase()`) results in `TypeError: Cannot read properties of undefined` in production.
3. **No Refactoring Safety:** Renaming a database column or API field (e.g. `title` to `name`) leaves hidden bugs throughout the codebase that only surface when a user triggers that specific route.

TypeScript solves this by introducing **static analysis**: it checks all contracts, arguments, and shapes at **compile time** (before runtime execution), catching errors during development.

---

### 2. The Mental Model
*How do I visualize how this works?*

- **Compile Time vs. Runtime:** TypeScript only exists while writing and compiling code.
- **Type Erasure:** The TypeScript compiler (`tsc`) validates types and then completely erases all type annotations (`: string`, `interface`, `type`). The output in `dist/` is pure, standard JavaScript.
- **Node.js is Unaware of TypeScript:** When Node.js executes the `.js` files, there is zero type checking performed at runtime. Types occupy 0 bytes in the output bundle.

```text
[ Developer writes .ts ] 
         │
         ▼
[ TypeScript Compiler (tsc) ]
  ├── 1. Type Check (halts build if any contract is violated)
  └── 2. Transpile & Type Erasure (strips all :types and interfaces)
         │
         ▼
[ Pure JavaScript .js in dist/ ]
         │
         ▼
[ Node.js / V8 Engine executes JS at runtime ]
```

---

### 3. Detailed Topic Breakdown

#### 1. Static Typing vs. Dynamic Typing
JavaScript checks types at runtime when the user triggers the code. TypeScript checks types at compile time before the server ever starts.
```typescript
function calculateTotal(price: number, qty: number): number {
  return price * qty;
}
// calculateTotal(85, "2"); // ❌ Compile error: 'string' not assignable to 'number'
```

#### 2. Type Erasure & Transpilation
All TypeScript constructs (`interface`, `type`, type annotations) are erased during compilation. They do not exist at runtime in Node.js.
```typescript
interface User { id: string; }
// console.log(typeof User); // ❌ Error: 'User' only refers to a type, not a value!
```

#### 3. `tsconfig.json` Core Directives
Controls how `tsc` converts TypeScript to JavaScript:
- `target`: JavaScript output version (e.g., `"ES2022"`).
- `module`: Module system (e.g., `"NodeNext"` for ES Modules `import`/`export`).
- `strict`: Enables all strict type-checking options (`strictNullChecks`, etc.).
- `rootDir`: Source code folder (`"./src"`).
- `outDir`: Compiled JavaScript output destination (`"./dist"`).

#### 4. Primitive Types
Explicit annotations for basic values:
```typescript
const clubName: string = "Arsenal";
const basePrice: number = 85.00;
const inStock: boolean = true;
const discountCode: string | null = null;
let trackingId: undefined = undefined;
```

#### 5. Type Inference
TypeScript automatically infers types from assigned values. Explicit annotations are only needed when declaring without initializing or when types can vary:
```typescript
let quantity = 2; // Automatically inferred as number
// quantity = "two"; // ❌ Compile error
```

#### 6. Arrays vs. Tuples
- **Array (`Type[]`):** Open-ended list of the same type.
- **Tuple (`[Type1, Type2]`):** Fixed-length, fixed-position array (e.g., coordinates, HTTP status pairs).
```typescript
const sizes: string[] = ["S", "M", "L"]; // Array
const httpStatus: [number, string] = [200, "OK"]; // Tuple (exactly 2 elements)
```

#### 7. Object Data Contracts (`interface` vs. `type`)
- Use `interface` for object models, database entities, and API payloads.
- Use `type` for unions, primitives, and tuples.
```typescript
interface Product {
  id: string;
  name: string;
}

// Extending an interface:
interface Jersey extends Product {
  club: string;
}
```

#### 8. Immutability & Optionality (`readonly`, `?`)
- `readonly`: Prevents modifying a property after creation (e.g., primary keys).
- `?`: Marks a property as optional (can be `undefined`).
```typescript
interface CartItem {
  readonly id: string; // Cannot be reassigned
  name: string;
  customName?: string; // Optional property
}
```

#### 9. Function Signatures & Return Types (`void`)
Explicitly type function parameters and return values. Use `void` if the function does not return anything:
```typescript
function logOrder(orderId: string): void {
  console.log(`Order processed: ${orderId}`);
}
```

#### 10. Typing Function Callbacks
Contract for functions passed as arguments: `(param: Type) => ReturnType`.
```typescript
function filterItems(
  items: CartItem[], 
  predicate: (item: CartItem) => boolean
): CartItem[] {
  return items.filter(predicate);
}
```

#### 11. Asynchronous TypeScript (`Promise<T>`, `async/await`)
Every `async` function must return `Promise<T>`. Always check for `null` before reading properties:
```typescript
async function fetchJersey(id: string): Promise<CartItem | null> {
  const result = await db.jerseys.findUnique({ where: { id } });
  if (!result) return null;
  return result;
}
```

#### 12. Generics Demystified (`<T>`)
Type parameters that act as placeholders, allowing reusable contracts without resorting to `any`:
```typescript
interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T;
}

// Reusable across different data types:
const jerseyRes: ApiResponse<Jersey> = { success: true, message: "OK", data: jersey };
const orderRes: ApiResponse<Order> = { success: true, message: "OK", data: order };
```

#### 13. Literal Union Types
Restricting values to exact specific strings or numbers:
```typescript
type KitSize = "S" | "M" | "L" | "XL";
type OrderStatus = "PENDING" | "PAID" | "DELIVERED";
```

#### 14. Defensive Typing & Narrowing (`any` banned, `unknown`)
Never use `any` (turns off compiler). Use `unknown` for untrusted input and narrow it with `typeof` or `in` before using:
```typescript
function processPrice(value: unknown): number {
  if (typeof value === "number") {
    return value * 1.1; // Safe
  }
  throw new Error("Invalid price type");
}
```

---

### 4. Top Gotchas & Pitfalls to Avoid

1. **Using Type/Interface as a Runtime Value:**
   - *Wrong:* `console.log(typeof CartItem)`
   - *Reason:* Types are completely erased during compilation.
2. **The `any` Trap:**
   - *Pitfall:* Disables TypeScript entirely. Always use specific types or `unknown` with narrowing.
3. **Unchecked Optional/Nullable Properties:**
   - *Pitfall:* Reading `item.customName.toUpperCase()` when `customName` is optional. Always use `if (item.customName)` guard first.
