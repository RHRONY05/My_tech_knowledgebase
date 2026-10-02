---
tags:
  - typescript
  - tooling
  - vite
  - architecture
  - compiler
last_reviewed: 2026-10-02
related_notes:
  - "[[typescript/core/typescript-contracts-and-generics]]"
  - "[[frontend/tooling/code-quality-linters-oxlint]]"
---

# TypeScript Compilation Architecture & The Multi-tsconfig Model

> [!CHECKLIST] Active Recall Self-Test
> - [ ] Why can neither Node.js nor web browsers execute TypeScript directly?
> - [ ] How does the TS-to-JS conversion differ between the **Backend** (`tsc` generates `dist/`) and the **Frontend** (Vite bundles `dist/`, `tsc` only checks)?
> - [ ] Does `tsc` remove types while checking them, and what does **Type Erasure** mean for emitted JavaScript?
> - [ ] Why is **one** `tsconfig.json` enough for a backend project, while a modern frontend project requires **three** config files?
> - [ ] What specific roles do `tsconfig.json`, `tsconfig.app.json`, and `tsconfig.node.json` play in isolating Browser code from Build Tool code?

---

## 1. The Fundamental Reality: Neither Node.js nor Browsers Run TypeScript

No production runtime on Earth understands TypeScript syntax natively:
- **Node.js** (backend runtime) only understands plain JavaScript.
- **Web Browsers** (Chrome, Safari, Firefox, Edge) only understand plain HTML, CSS, and plain JavaScript.

Browsers and Node engines have **zero concept** of:
- Type annotations (`const count: number = 5`)
- Interfaces (`interface CartItem { id: string }`)
- Type aliases (`type Size = 'S' | 'M' | 'L'`)
- Generics (`ApiResponse<T>`)

No matter what we write in development, **every single line of TypeScript must be converted into plain JavaScript before it can ever be executed.**

---

## 2. How Backend vs. Frontend Convert TS to JS

Although both ends require JavaScript at runtime, the way they convert TypeScript to JavaScript is completely different:

```
BACKEND CONVERSION:
  src/*.ts ────────► [ tsc (TypeScript Compiler) ] ────────► dist/*.js
                     • Checks types against contracts        (Node runs this directly)
                     • Erases types & emits JS files

FRONTEND CONVERSION:
  src/*.tsx ──┐
  styles.css  ┼─► [ Vite (Bundler + esbuild) ] ────────► dist/ (HTML, CSS, bundled JS)
  assets/     │   • Bundles, minifies, extracts CSS
              │
  src/*.tsx ──┴─► [ tsc -b (noEmit: true) ] ────────────► 0 files emitted!
                  • ONLY validates type safety
```

### In the Backend (`backend/`)
- Node.js can execute individual `.js` files directly without any bundling.
- Therefore, **`tsc` does both jobs**:
  1. It inspects all types to ensure contracts are honored.
  2. It converts `.ts` into `.js` files and outputs them into `backend/dist/`.
- In production, you run: `node dist/server.js`.

### In the Frontend (`frontend/`)
- Browsers cannot run a folder of 50 separate, unbundled `.js` files efficiently. Browsers also need CSS processing (Tailwind), asset optimization, and code minification.
- `tsc` is not a bundler and cannot process CSS or images.
- Therefore, **Vite does the conversion and creates `frontend/dist/`**.
- **`tsc`'s only job in the frontend is safety inspection (`"noEmit": true`)**: it validates types in memory, but emits zero JavaScript files.

---

## 3. Does `tsc` Remove Types While Checking Them?

**How the compiler works internally:**

1. **Step 1: Parsing into Memory (AST)**  
   When you run `tsc`, it reads your `.ts` files into memory and constructs an **Abstract Syntax Tree (AST)**.
2. **Step 2: Type Checking (Types are Active)**  
   `tsc` walks through the syntax tree and evaluates your type rules. While checking, **types are NOT removed**—`tsc` needs the types to verify that `user.name` is indeed a string and that `cart.items.push(item)` matches the contract.
3. **Step 3: Code Emission / Type Erasure (Types are Erased)**  
   - If `tsc` finds type errors, it halts and reports them.
   - If the code is valid (and `"noEmit": false` as in the backend), `tsc` generates the final `.js` files by **erasing 100% of all types, interfaces, and generics**. The resulting JavaScript files contain exactly **0 bytes** of TypeScript.
   - If `"noEmit": true` (as in the frontend), `tsc` finishes the check and exits immediately. It writes nothing to disk. Vite's embedded `esbuild` handles the type erasure in milliseconds.

---

## 4. The Role of `tsconfig`: The Inspector's Rulebook

`tsc` is the engine (the executable program). `tsconfig.json` is its rulebook.

Without a `tsconfig`, `tsc` cannot answer:
- Which files to inspect (and which to ignore, like `node_modules`).
- What JavaScript version to target (`ES2022`, `ESNext`, etc.).
- How strict to be (e.g. banning unused variables with `"noUnusedLocals": true`).
- What path aliases mean (e.g. mapping `@/*` to `./src/*`).

### Why ONE `tsconfig.json` is Enough for the Backend
In `backend/`, everything runs in **one single environment**: Node.js.
- `backend/tsconfig.json` sets `"module": "NodeNext"` and `"outDir": "./dist"`.
- It does not need to know about browser buttons or the DOM. One file configures everything.

### Why the Frontend Needs THREE `tsconfig` Files
In `frontend/`, code is written for **two completely different environments**:

| Environment | Files | Where it runs | What globals exist? |
| :--- | :--- | :--- | :--- |
| **Browser Runtime** | `src/App.tsx`, `src/main.tsx` | The user's browser (Chrome, Safari) | `window`, `document`, `localStorage`, `HTMLElement` |
| **Developer Build Tool** | `vite.config.ts` | Node.js on the developer's PC | `process`, `__dirname`, `node:path`, `node:url` |

If you tried to use **one** config file:
- Allowing browser DOM types would let you accidentally use `document` inside `vite.config.ts` (crashing Node.js).
- Allowing Node types would let you use `process.env` inside `App.tsx` (crashing in Chrome).

---

## 5. The Frontend 3-File Architecture (Project References)

```
                           tsconfig.json
                    (Solution Master / Coordinator)
                           "files": []
                           "references": [...]
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
       tsconfig.app.json               tsconfig.node.json
     (Browser Application)            (Build Script Config)
      • include: ["src"]               • include: ["vite.config.ts"]
      • lib: ["DOM", "ES2022"]         • lib: ["ES2022"] (NO DOM!)
      • jsx: "react-jsx"               • moduleResolution: "bundler"
      • paths: { "@/*": ["./src/*"] }  • noEmit: true
      • noEmit: true
```

1. **`tsconfig.json` (The Master Coordinator):**  
   Has `"files": []`. It doesn't check any code directly. It tells `tsc -b`: *"This project has two distinct sub-projects. Run them with Project References."*
2. **`tsconfig.app.json` (The Browser App):**  
   Includes `src/`. Supplies browser DOM declarations (`window`, `document`), configures `"jsx": "react-jsx"` for React 19, and sets path aliases (`@/*`).
3. **`tsconfig.node.json` (The Build Tool):**  
   Includes only `vite.config.ts`. Strictly excludes the DOM and configures Node.js build-environment rules.

---

## 6. The 3 Error Watchers Timeline

| Watcher | When it runs | What it checks | Where you see it |
| :--- | :--- | :--- | :--- |
| **1. Editor TS Server (`tsserver`)** | **While typing** (live on keystroke) | Type mismatches, broken contracts, typos | Red squiggly lines in your editor |
| **2. Vite Dev Server** | **While running dev** (`npm run dev`) | Syntax errors, broken imports, JSX parsing | Error overlay on your browser screen |
| **3. TypeScript Compiler (`tsc`)** | **When building** (`npm run build`) | Workspace-wide type safety validation | In your terminal (halts build on error) |
