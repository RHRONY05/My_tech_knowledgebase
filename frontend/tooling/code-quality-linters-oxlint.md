---
tags:
  - frontend
  - tooling
  - linting
  - oxlint
  - react
last_reviewed: 2026-10-02
related_notes:
  - "[[react-hook-form-architecture]]"
  - "[[react-router-v6-architecture]]"
---

# Modern Frontend Code Quality: The 4 Layers & Rust-Based Linters (Oxlint)

> **Active Recall Self-Test:**
> 1. What are the **4 distinct layers** of frontend code quality, and what specific role does each tool (Compiler, Type Checker, Formatter, Linter) perform?
> 2. Why is **Oxlint** 50x–100x faster than legacy ESLint, and how does Abstract Syntax Tree (AST) parsing in Rust achieve this?
> 3. What catastrophic runtime bugs does the `react/rules-of-hooks` lint rule prevent?
> 4. Why does exporting non-component constants from a React component file break Vite's Fast Refresh (Hot Module Reloading)?

---

## 1. The Core Problem
### The Slow and Confusing Web Toolchain

Frontend developers often confuse compilers, formatters, and linters, running overlapping or sluggish tools:
- Running legacy JavaScript-based ESLint across a 1,000-component codebase takes **15 to 45 seconds** in CI/CD pipelines, frustrating team velocity.
- Developers rely on Prettier to catch bugs (which it cannot do) or rely on TypeScript to catch bad Hook ordering (which it cannot do).

#### The Failure: The Rules of Hooks Violation
In React, Hooks (`useState`, `useEffect`) rely on internal array index call order:
```jsx
function BadComponent({ isAuthorized }) {
  if (!isAuthorized) {
    return <Login />; // ❌ Early return before hook!
  }
  const [data, setData] = useState(null); // Hook order changes dynamically!
}
```
- **The Catastrophe:** If `isAuthorized` toggles, the number of hooks called changes. React loses track of internal state pointers, corrupting component state and throwing `Error: Rendered fewer hooks than expected`.
- Neither the Babel compiler nor Prettier can catch this bug. Only a **Linter analyzing the Abstract Syntax Tree (AST)** can enforce these invariants.

#### The Architectural Solution
1. Understand the strict separation of concerns across the **4 Quality Layers**.
2. Adopt **Rust-based next-generation linters (Oxlint)** that execute static analysis in milliseconds across all CPU cores in parallel.

---

## 2. The Mental Model

### The 4 Layers of Frontend Code Quality
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. COMPILER (Vite / SWC / esbuild / Babel)                                  │
│    - Job: Translates modern JSX and ES6+ into browser-compatible JavaScript.│
│    - Speed: Blazing fast (runs during dev server & production builds).      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. TYPE CHECKER (TypeScript / tsc)                                          │
│    - Job: Validates function contracts, interface shapes, and types.       │
│    - Example: Rejects passing a String to a function expecting a Number.    │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. CODE FORMATTER (Prettier / Biome)                                        │
│    - Job: Pure aesthetics (tabs vs spaces, semicolons, line wrapping).       │
│    - Does NOT care if code has bugs; only cares that it looks uniform.      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 4. STATIC LINTER (Oxlint / ESLint)                                          │
│    - Job: Catches logic bugs, security hazards, dead code & React traps.   │
│    - Analyzes the Abstract Syntax Tree (AST) to enforce engineering rules.  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ESLint (JavaScript) vs. Oxlint (Rust) Execution
```
ESLint (JavaScript on Node.js):
- Single-threaded V8 execution
- Heavy AST memory allocation
- Traverses 500 files in ~25 seconds ⏳

──────────────────────────────────────────────────────────

Oxlint (Compiled Rust Binary):
- Parallel multi-threaded execution across all CPU cores
- Zero-garbage-collection native machine code
- Traverses 500 files in ~0.08 seconds (50x - 100x faster!) ⚡
```

---

## 3. Production Code Breakdown

### A. Oxlint Configuration (`frontend/.oxlintrc.json`)
```json
{
  "rules": {
    // ⚠️ CRITICAL: Enforce React Hook Call Order (Prevents State Corruption)
    "react/rules-of-hooks": "error",

    // ⚠️ Prevent Fast Refresh hot-reload breakage in Vite
    "react/only-export-components": [
      "warn",
      { "allowConstantExport": true }
    ],

    // Dead code and common JS traps
    "no-unused-vars": "warn",
    "no-debugger": "error",
    "no-console": "warn",
    "eqeqeq": "error"
  }
}
```

### B. Fast NPM Lint Scripts (`frontend/package.json`)
```json
{
  "scripts": {
    "lint": "oxlint",
    "lint:fix": "oxlint --fix"
  },
  "devDependencies": {
    "oxlint": "^0.15.0"
  }
}
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Fast Refresh Component Export Rule
- **The Trap:** Exporting helper functions or mock objects from the same file as a React component:
  ```jsx
  export const formatPrice = (val) => `$${val}`; // ❌ Breaks Fast Refresh
  export default function ProductCard() { ... }
  ```
- **The Bug:** Vite's Fast Refresh uses React Hot Reloading. When Vite detects an exported non-component function, it cannot hot-reload the file in isolation; it is forced to reload the entire web page, wiping your current UI state!
- **The Fix:** Move helper functions and constants to dedicated `src/utils/` files so component files export **only React components**.

### ⚠️ Gotcha 2: The Rules of Hooks Invariants
- **Rule 1:** Only call Hooks at the **top level**. Never call Hooks inside loops, conditions (`if`), or nested functions.
- **Rule 2:** Only call Hooks from **React function components** or **custom Hooks** (prefixed with `use`).

### ⚠️ Gotcha 3: Formatter vs. Linter Conflict
- **The Trap:** Configuring ESLint rules that fight with Prettier (e.g. single quotes vs double quotes).
- **The Rule:** Let Prettier (or Biome) handle 100% of formatting. Let Oxlint handle 100% of logic and bug detection.
