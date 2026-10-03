---
tags:
  - active-recall
  - index
  - revision
last_reviewed: 2026-10-02
---

# Master Active Recall Index

> **How to use this checklist:**  
> Before opening a note, read the questions below. If you can answer them out loud or on a whiteboard in 60 seconds, your mental model is solid. If you struggle or hesitate, click the link to review the note.

---

## 🛠️ Backend Architecture

### Universal Architecture Standards (`backend/architecture/`)

#### [[backend/architecture/production-backend-5-pillars|The Five Pillars of Production Backend Architecture]]
- [ ] What are the **5 Non-Negotiable Pillars** of modern production backend engineering?
- [ ] Why is typing raw `process.env.VARIABLE` directly inside route controllers considered a critical architectural violation?
- [ ] What is the "Fail-Fast" boot sequence, and why must database and cache health checks resolve *before* `app.listen()`?
- [ ] How does standardizing `ApiResponse` and `ApiError` envelopes eliminate frontend JSON parsing errors?
- [ ] What is the "One-Shot Agent Prompt" to scaffold an enterprise backend in seconds?

### Express.js & Node (`backend/express/`)

#### [[backend/express/file-upload-pipeline-multer-cloudinary|File Upload Pipeline (Multer + Cloudinary)]]
- [ ] Why does uploading Base64 strings inside JSON bodies crash Node.js production servers?
- [ ] What is the memory hazard of using `multer.memoryStorage()` under concurrent traffic, and when should you switch to direct presigned cloud uploads?
- [ ] How does the HTTP `multipart/form-data` protocol delineate text fields from binary streams using boundaries?
- [ ] If a database insert fails after an image is uploaded to Cloudinary, how do you prevent an orphaned cloud asset leak?

#### [[backend/express/auth-lifecycle-jwt-cookies|Full-Stack Authentication Lifecycle (JWT + HttpOnly Cookies)]]
- [ ] Why is storing JWTs in `localStorage` vulnerable to XSS attacks, and how do `HttpOnly`, `SameSite`, and `Secure` cookie flags protect user sessions?
- [ ] What is the dual-token architecture (Access Token vs Refresh Token), and why should Access Tokens have short lifespans?
- [ ] How does the initial client boot (`checkAuth` / `/auth/me`) prevent the "flash of unauthorized content" on page refresh?
- [ ] Why must user verification (e.g. email confirmation) gate login rather than auto-authenticating upon registration?

#### [[backend/express/oauth2-google-authentication|Google OAuth 2.0 Backend Architecture: ID Token Verification & Session Minting]]
- [ ] In Google Sign-In, what is the critical architectural difference between an **ID Token (`id_token`)**, an **Access Token (`access_token`)**, and a **Refresh Token (`refresh_token`)**?
- [ ] Why does the backend verify Google ID Tokens cryptographically using `google-auth-library` rather than trusting the user profile payload sent from the frontend?
- [ ] Why must the backend mint its **own custom JWT** after verifying Google credentials instead of using the Google ID Token for subsequent API requests?
- [ ] How can you test Google OAuth backend endpoints using the **Google OAuth 2.0 Playground** without building a frontend UI?

#### [[backend/express/structured-logging-pino|High-Performance Structured Logging in Node.js (Pino Ecosystem)]]
- [ ] Why is `console.log` a serious performance and security anti-pattern in production Node.js applications?
- [ ] How does `pino-http` intercept the Express request/response lifecycle without having to manually invoke logger statements in every route handler?
- [ ] Why must production logs be formatted as **Structured JSON** rather than colorful human-readable text?
- [ ] How do **Custom Serializers** prevent accidental leakage of sensitive authorization tokens and passwords into log files?

#### [[backend/express/centralized-error-handling-and-api-contracts|Centralized Error Handling & Enterprise API Contracts (ApiError & ApiResponse)]]
- [ ] Why does inheriting from the native JavaScript `Error` class and capturing `Error.captureStackTrace` improve production debugging?
- [ ] What is the purpose of the `asyncHandler` higher-order function, and how does it prevent unhandled promise rejections without repetitive try/catch blocks?
- [ ] Why does Express require error-handling middleware to strictly declare **four parameters** `(err, req, res, next)`?
- [ ] How does standardized `ApiResponse<T>` envelope formatting prevent frontend parsing crashes across diverse API endpoints?

#### [[backend/express/express-middleware-architecture|Express Middleware Pipeline: Execution Chains, Factory Patterns & CORS]]
- [ ] What is the internal call stack behavior of Express middleware, and what happens when neither `next()` nor a response method is called?
- [ ] What is the crucial difference between passing a middleware reference `router.use(auth)` vs. invoking a factory function `router.use(validate(schema))`?
- [ ] Why does setting `origin: "*"` with `credentials: true` trigger a browser CORS failure, and how does dynamic whitelist reflection resolve it?
- [ ] What critical security problem happens if error-handling middleware calls `next(err)` instead of terminating with `res.status().json()`?

#### [[backend/express/jwt-refresh-token-rotation|JWT Dual-Token Lifecycle & Refresh Token Rotation]]
- [ ] What is the fundamental difference between an Opaque Token (Stateful) and a JWT (Stateless)?
- [ ] Why should Access Tokens have a short lifespan (10–15m) while Refresh Tokens live for days in HttpOnly, SameSite=Strict cookies?
- [ ] What is Refresh Token Rotation, and how does it detect token replay attacks by malicious actors?
- [ ] Why does verifying a refresh token still require querying the database even if the cryptographic signature is valid?

#### [[backend/node/production-backend-architecture-and-structure|Production Node.js Architecture & Clean Enterprise Structure]]
- [ ] Why should you strictly distinguish `dependencies` from `devDependencies` in containerized production deployments?
- [ ] What is the architectural role of `Object.freeze()` when declaring enums or constant maps in JavaScript?
- [ ] Why does an enterprise API follow MVC where the "View" is replaced by JSON API Contracts and Validators?
- [ ] Where does `public/temp` fit into the file upload lifecycle, and why is `localPath` tracked in models?

#### [[backend/node/email-service-nodemailer-mailgen|Enterprise Email Architecture: Nodemailer & Mailgen]]
- [ ] What are the "Three Pillars" of transactional email architecture in Node.js?
- [ ] Why should email content generation (Mailgen) be completely decoupled from email transport (Nodemailer)?
- [ ] What is the role of an SMTP sandbox like Mailtrap in development vs. SendGrid/SES in production?
- [ ] Why is `await sendEmail(...)` directly in an HTTP route dangerous for latency and how should it be mitigated?

#### [[backend/express/validation-defense-in-depth|Defense-in-Depth Validation (Client UX vs Server Security)]]
- [ ] Why is relying exclusively on client-side form validation (e.g. Zod or HTML5 `required`) a critical security vulnerability?
- [ ] What distinct roles do validation and sanitization play in preventing SQL/NoSQL injection and XSS?
- [ ] How does the "Fail-Fast" middleware pattern in Express prevent corrupt payloads from reaching database controllers?
- [ ] Why should API responses return standardized 400 Bad Request error dictionaries rather than unhandled 500 server crashes?

### Automated Testing (`backend/testing/`)

#### [[backend/testing/jest-supertest-integration-testing|Automated Backend Integration Testing: Jest, Supertest & Database Teardown]]
- [ ] Why does Supertest NOT require your Express server to listen on a physical TCP port (`app.listen()`), and how does in-memory request injection work?
- [ ] What is the **AAA Pattern (Arrange, Act, Assert)**, and how does it structure every integration test?
- [ ] Why does running integration tests concurrently across files break relational database test suites, and how does `--runInBand` guarantee isolation?
- [ ] How do you mock third-party external services (like Google OAuth or Stripe) without firing live external HTTP requests during tests?

---

## 🗄️ Database & Concurrency Architecture

### Prisma ORM & Data Access (`database/prisma/`)

#### [[database/prisma/prisma_fundamentals|Prisma ORM: Architecture, Lifecycle & Production Mind Map]]
- [ ] What is the exact role difference between `prisma` (devDependencies) and `@prisma/client` (dependencies)?
- [ ] What happens under the hood when you run `npx prisma migrate dev` vs `npx prisma generate`?
- [ ] Why does creating multiple `new PrismaClient()` instances crash PostgreSQL with connection starvation, and how does the singleton pattern prevent it?
- [ ] Why must `prisma migrate dev` NEVER be run in production CI/CD or Docker entrypoint scripts, and what command replaces it?
- [ ] How does Prisma Client dynamically translate type-safe JavaScript query methods into parameterized raw SQL queries at runtime?

### PostgreSQL & Relational (`database/postgresql/`)

#### [[database/postgresql/transactions-and-pessimistic-locking|PostgreSQL Concurrency Control: ACID Transactions & Pessimistic Locking (FOR UPDATE)]]
- [ ] Why does running concurrent queries with `pool.query()` fail for ACID transactions, and why must transactions check out a dedicated client with `pool.connect()`?
- [ ] How does a **Race Condition** occur in high-traffic inventory systems (e.g. 50 users clicking "Book Seat A1" at the exact same millisecond)?
- [ ] What does `SELECT ... FOR UPDATE` do to a database row at the storage engine level, and how does it queue competing transactions?
- [ ] Why is placing `client.release()` inside a `finally` block mandatory to prevent server-wide connection pool starvation?

#### [[database/postgresql/database-migrations-node-pg-migrate|PostgreSQL Database Migrations: Schema Version Control with node-pg-migrate]]
- [ ] Why is running manual SQL `CREATE TABLE` scripts in production dangerous, and how do **Database Migrations** act as Git version control for schemas?
- [ ] Why do migration files always begin with a **Unix Timestamp** prefix (e.g. `1787331883988_init-schema.js`)?
- [ ] What is the contract between `exports.up` and `exports.down`, and what catastrophic failure occurs during a rollback if `exports.down` is neglected?
- [ ] How does `node-pg-migrate` use a dedicated internal table (`pgmigrations`) to track which schema patches have already been applied?

#### [[database/postgresql/multi-environment-db-isolation|Multi-Environment Database Isolation: Server Instances vs. Logical Databases]]
- [ ] Why does running automated Jest test suites against your active local development database cause data pollution, flaky tests, and accidental data wipes?
- [ ] What is the fundamental difference between a **Database Server Process (Instance)** and a **Logical Database** in PostgreSQL?
- [ ] How does Docker's `/docker-entrypoint-initdb.d/` directory automatically provision isolated test databases on container initialization?
- [ ] How does dynamic environment switching in `src/config/db.js` guarantee that tests NEVER touch development data, even if connection strings are omitted?

#### [[database/postgresql/disaster-recovery-and-backups|PostgreSQL Disaster Recovery: Logical Backups, pg_dump & Automated Retention]]
- [ ] Why does copying raw `/var/lib/postgresql/data` files from a running database risk severe data corruption, and how does `pg_dump` guarantee a 100% consistent transactional snapshot?
- [ ] What is the industry **"3-2-1" Backup Rule** for critical stateful systems?
- [ ] Why must `pg_dump` include the `--clean --if-exists` flags when generating restoration scripts?
- [ ] What are the 6 essential components of an enterprise-grade automated backup script?

### MongoDB & NoSQL (`database/mongodb/`)

#### [[database/mongodb/mongodb-indexing-strategies|MongoDB Indexing Strategies & Query Optimization]]
- [ ] What is the execution difference between a `COLLSCAN` (Collection Scan) and an `IXSCAN` (Index Scan) in terms of computational complexity and disk I/O?
- [ ] How does the B-Tree data structure allow MongoDB to achieve $O(\log n)$ lookup times on indexed fields?
- [ ] What is the "Equality, Sort, Range" (ESR) rule when designing compound indexes?
- [ ] Why does adding an index speed up queries but degrade write throughput (`insert`, `update`, `delete`)?

#### [[database/mongodb/mongoose-methods-and-virtuals|Mongoose Schema Extensions (Methods, Statics & Virtuals)]]
- [ ] What is the fundamental difference between an **Instance Method** (`schema.methods`) and a **Static Method** (`schema.statics`) in Mongoose?
- [ ] Why does using ES6 arrow functions `() => {}` inside Mongoose methods or virtual getters break document field access?
- [ ] What are Mongoose **Virtuals**, and why are they never physically written to MongoDB on-disk storage?
- [ ] Why does an API endpoint returning a document omit virtual properties by default, and how does `toJSON: { virtuals: true }` fix this?

#### [[database/mongodb/mongoose-lifecycle-hooks-and-bcrypt|Mongoose Lifecycle Hooks, Schema Methods & Bcrypt Security]]
- [ ] Why is `if (!this.isModified("password")) return next();` mandatory in a `pre("save")` hook to prevent double-hashing?
- [ ] What happens if you define a Mongoose middleware or schema method using an ES6 arrow function `() => {}`?
- [ ] What is the architectural distinction between a Mongoose Model (Static methods) and a Document (Instance methods)?
- [ ] Why must temporary reset/verification tokens be stored in the database as SHA-256 hashes instead of plain text?
- [ ] What is the purpose of `user.save({ validateBeforeSave: false })` when updating internal tokens?

#### [[database/mongodb/mongoose-connection-and-fail-fast|Mongoose Database Connection, Fail-Fast Lifecycle & Process Control]]
- [ ] Why is starting an Express HTTP server before database connection resolution considered a critical production anti-pattern?
- [ ] What does `process.exit(1)` communicate to the Operating System and container orchestrators like Docker and Kubernetes?
- [ ] Why must `process.exit()` NEVER be called inside route handlers or Express middleware?
- [ ] How does a graceful shutdown handler (`SIGINT`/`SIGTERM`) prevent database connection leaks and aborted in-flight transactions?

#### [[database/mongodb/nosql-polymorphic-schema-design|NoSQL Polymorphic Schema Design (Heterogeneous Catalogs)]]
- [ ] Why does creating separate database collections for every product category cripple multi-category search, pagination, and sorting?
- [ ] How does the **Layered Subdocument Pattern** enable polymorphic data modeling while keeping core e-commerce fields uniformly indexable?
- [ ] What is the **Attribute Pattern** in MongoDB, and when should you choose key-value attribute arrays over structured subdocuments?
- [ ] How do you prevent schema drift and enforce category-specific validation in Mongoose when saving heterogeneous products?

#### [[database/mongodb/mongodb-relationships-and-populate|MongoDB Relationships: Embedding vs. Referencing (.populate())]]
- [ ] Why does Mongoose `.populate()` fundamentally differ from a native SQL `JOIN`, and why does calling `.populate()` inside a loop cause the N+1 query bottleneck?
- [ ] What are the rules of thumb for choosing between **Embedding (Denormalization)** and **Referencing (Normalized ObjectId links)** in document databases?
- [ ] What is MongoDB's strict 16MB BSON document size limit, and how does unbound array growth cause catastrophic write failures?
- [ ] Why must e-commerce orders denormalize an immutable snapshot of product price and title at the moment of checkout?

### Redis & In-Memory Caching (`database/redis/`)

#### [[database/redis/redis-architecture-caching-and-ttl|Redis Architecture: Sub-Millisecond Caching, TTL & Multi-Container Orchestration]]
- [ ] Why does storing ephemeral data (e.g. AI chat sessions, rate limits) in PostgreSQL cause disk I/O bottlenecks compared to Redis?
- [ ] What is the fundamental latency difference between in-memory RAM access (<1ms) and disk/SSD storage (5–25ms)?
- [ ] How does Time-To-Live (TTL) with `SETEX` eliminate the need for scheduled cron jobs to purge expired session data?
- [ ] In Docker Compose, why can containers communicate via service names (`redis:6379`) while the host connects via `localhost:6379`?
- [ ] Why does vector similarity search for AI embeddings require `pgvector/pgvector:pg16` instead of standard `postgres:16`?

### pgvector & AI Embeddings (`database/pgvector/`)

#### [[database/pgvector/vector_embeddings_and_search|Vector Embeddings & Semantic Search: Production Cheat Sheet & Mental Model]]
- [ ] Why does traditional SQL keyword search (`ILIKE '%red%'`) fail when users search with natural, descriptive language?
- [ ] What is a vector embedding, and why does AI use 768 dimensions instead of 2D or 3D?
- [ ] Who actually performs the search: the Gemini AI model or PostgreSQL?
- [ ] What is the difference between an Embedding Model (produces numbers) and a Chat Model (produces text)?
- [ ] What does Cosine Distance measure, what does a lower vs higher distance value mean in practice, and why do we subtract it from 1 to get a similarity score?
- [ ] What is an HNSW index, and what happens to search performance without one on a large dataset?
- [ ] When a user sends a long prompt to ChatGPT, does the entire prompt become one vector or is it processed differently?

---

## 🐳 DevOps & Infrastructure

### Docker & Containers (`devops/docker/`)

#### [[devops/docker/docker-core-and-networking|Docker Core Architecture, Container Anatomy & Bridge Networking]]
- [ ] Why is a Docker container NOT a Virtual Machine (VM), and what two Linux kernel primitives (Namespaces vs. Cgroups) make containerization possible?
- [ ] How does port mapping (`-p 5000:5000` Host:Container) punch a hole through the container NAT boundary?
- [ ] Why does the default `bridge` network lack internal DNS resolution, and why must multi-container systems use a user-defined custom bridge network?
- [ ] Inside a Docker container, why does pointing your database client to `localhost:5432` fail with connection refused?

#### [[devops/docker/dockerfile-and-layer-caching|Production Dockerfiles, Layer Caching Optimization & Multi-Stage Builds]]
- [ ] Why does copying `package*.json` and running `npm install` BEFORE copying source code (`COPY . .`) dramatically accelerate build times?
- [ ] How do **Multi-Stage Builds** reduce production image sizes from 1.2 GB to under 30 MB for frontend applications?
- [ ] Why should production Node.js images run `npm ci --omit=dev` instead of standard `npm install`?
- [ ] What critical security and performance hazards occur if `.dockerignore` omits `node_modules` or `.env` files?

#### [[devops/docker/docker-compose-production-patterns|Docker Compose Production Patterns: The 4 Pillars, Healthchecks & Volumes]]
- [ ] What are the **4 Pillars** that define every Docker Compose architecture?
- [ ] Why does naive `depends_on: [postgres]` cause application startup crashes, and how does `condition: service_healthy` solve this race condition?
- [ ] What is the fundamental difference between a **Named Volume** (`pgdata:/var/lib/postgresql/data`) and a **Bind Mount** (`./src:/app/src`)?
- [ ] Why should database migrations run as a dedicated **One-Shot Task** before backend services boot?

### Nginx & Web Serving (`devops/nginx/`)

#### [[devops/nginx/nginx-reverse-proxy-and-spa-routing|Nginx Production Web Architecture: Reverse Proxying & SPA Routing]]
- [ ] Why does refreshing the browser on a React client-side route like `/movies/seat-selection` return a 404 Not Found error without Nginx's `try_files` directive?
- [ ] How does an Nginx **Reverse Proxy** eliminate browser Cross-Origin Resource Sharing (CORS) issues between frontend and backend?
- [ ] Why can static production assets (`dist/assets/*.js`) be cached with a 1-year `Cache-Control: max-age=31536000, immutable` header, while `index.html` must never be cached?
- [ ] What critical HTTP headers (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) must Nginx pass when forwarding `/api/` traffic to Express?

### Security & SSL/TLS (`devops/security/`)

#### [[devops/security/ssl-tls-and-certbot|SSL/TLS Architecture, Public-Key Cryptography & Certbot Deep-Dive]]
- [ ] What are the three foundational vulnerabilities of plain unencrypted HTTP that TLS (HTTPS) resolves?
- [ ] How does the TLS handshake combine **Asymmetric Encryption** (Public/Private Keys) with **Symmetric Encryption** (Session Keys) to balance security and performance?
- [ ] How does the Let's Encrypt **ACME HTTP-01 Challenge** mathematically prove domain ownership to issue a signed certificate?
- [ ] Why must Let's Encrypt certificates be renewed every 90 days, and how does a Certbot cron / systemd timer automate this?

### Cloud Deployment (`devops/deployment/`)

#### [[devops/deployment/self-hosted-vps-vs-paas|Production Cloud Deployment: Self-Hosted Linux VPS vs. Managed PaaS]]
- [ ] Why does a 1 GB RAM Linux VPS crash during `docker compose build`, and how does allocating a **2 GB Swap file with `vm.swappiness=10`** prevent Out-Of-Memory (OOM) kernel panics?
- [ ] What are the key architectural trade-offs between a **Self-Hosted VPS** (All-in-One Docker network) and a **Decoupled Managed PaaS** (Vercel + Render + Neon)?
- [ ] How does Uncomplicated Firewall (UFW) enforce the security perimeter on a production VPS, and why must database port 5432 remain closed?
- [ ] Why does Windows reject SSH private keys until permissions are stripped using `icacls`, and what is the Linux equivalent (`chmod 600`)?

### CI/CD Pipelines (`devops/ci-cd/`)

#### [[devops/ci-cd/github-actions-ci-pipeline|Continuous Integration (CI) with GitHub Actions: Sterile Runners & Service Containers]]
- [ ] If a developer already runs `npm test` locally on their laptop, what three critical failure modes (e.g. "Forgotten File" trap, Windows case-insensitivity) make cloud CI mandatory?
- [ ] How does GitHub Actions use **Service Containers** (`services: postgres`) to provision an ephemeral database instance during test execution?
- [ ] What is the difference between running tests against a "dirty" developer laptop vs. a 100% "sterile" cloud virtual machine runner?
- [ ] How does `actions/cache` optimize CI pipeline execution times by preserving `node_modules` based on `package-lock.json` hash keys?

---

## 🎨 Frontend Architecture

### React (`frontend/react/`)

#### [[frontend/react/react-router-v6-architecture|React Router v6 Architecture: Layouts, Outlets & Nested Routing]]
- [ ] How does React Router v6's `<Outlet />` component work internally to enable shared persistent layouts across sub-routes?
- [ ] What is the fundamental difference between declarative `<Link />` / `<Navigate />` components and imperative `useNavigate()` hook calls?
- [ ] Why did React Router v6 replace v5's regex-based route matching with a relative path ranking algorithm?
- [ ] How does the HTML5 History API (`pushState`, `replaceState`, `popstate`) enable client-side navigation without triggering full page reloads?

#### [[frontend/react/react-auth-guards-and-routing|Protected Routes, Role Guards & Redux State Synchronization]]
- [ ] Why does evaluating route protection during initial application boot without an `isLoading` gate cause a momentary "flash redirect" to `/login`?
- [ ] How does a declarative `<CheckAuth>` wrapper component enforce Role-Based Access Control (RBAC) across public, authenticated, and admin routes?
- [ ] What race condition happens when a user clicks "Login", Redux updates `isAuthenticated = true`, and both `useEffect` and `<CheckAuth>` attempt navigation simultaneously?
- [ ] Why should unauthenticated users be redirected with `state: { from: location }` so they return to their requested page after logging in?

#### [[frontend/react/react-hook-form-architecture|React Hook Form: Uncontrolled Inputs & Dynamic Schema Validation]]
- [ ] Why does traditional controlled React state (`value={state}` + `onChange={setState}`) degrade rendering performance on complex forms with 20+ fields?
- [ ] How does React Hook Form achieve isolated, zero-re-render typing using uncontrolled DOM inputs and `ref` registration?
- [ ] What is the role of `<Controller />`, and why is it necessary for custom UI components (Shadcn UI, Radix Select, DatePickers)?
- [ ] How does `useFieldArray` manage dynamic arrays of inputs (e.g. adding/removing product variants) without losing user input on re-render?

### Redux Toolkit (`frontend/redux/`)

#### [[frontend/redux/redux-toolkit-core-architecture-and-dataflow|Redux Toolkit (RTK) Core Architecture & Unidirectional Dataflow]]
- [ ] What is the exact sequence of events in the Redux unidirectional dataflow from user click to component re-render?
- [ ] How does Immer.js allow developers to write "mutating" logic (`state.value += 1`) while maintaining 100% immutable state snapshots under the hood?
- [ ] Why should you organize Redux code by **feature** (`features/counter/counterSlice.js`) rather than by technical type (`actions/`, `reducers/`)?
- [ ] What catastrophic re-render loop occurs if you return a newly instantiated object or array directly inside an unmemoized `useSelector`?
- [ ] Why is returning a new state value AND mutating the draft in the same `createSlice` reducer an illegal operation in Immer?

#### [[frontend/redux/javascript-reduce-to-redux-reducers|From JavaScript Array.reduce() to Redux Reducers]]
- [ ] Why is Redux named after JavaScript's native `Array.prototype.reduce()` function?
- [ ] In both `Array.reduce` and Redux, what are the exact definitions of the **Accumulator** and the **Current Item**?
- [ ] What catastrophic bug occurs in `Array.prototype.reduce()` if you omit the `initialValue` argument on an empty array?
- [ ] How does JavaScript's computed property syntax `(tally[fruit] || 0) + 1` dynamically build lookup dictionaries?
- [ ] How is an application's lifecycle over time mathematically equivalent to `actions.reduce(rootReducer, initialState)`?

#### [[frontend/redux/redux-toolkit-async-thunks|Redux Toolkit Async Thunks: Lifecycle, Unwrapping & API Envelopes]]
- [ ] What three distinct action types does `createAsyncThunk('auth/login', ...)` automatically generate, and how does the `extraReducers` builder handle them?
- [ ] Why does a rejected thunk NOT automatically trigger the `catch` block in a React component unless you call `.unwrap()`?
- [ ] What is the "Envelope Mismatch" bug when a backend returns `ApiResponse { statusCode, data, message }` and Axios wraps it in `response.data`?
- [ ] How does `thunkAPI.rejectWithValue` prevent non-serializable Axios error objects from polluting the Redux DevTools?

### Frontend Tooling (`frontend/tooling/`)

#### [[frontend/tooling/code-quality-linters-oxlint|Modern Frontend Code Quality: The 4 Layers & Rust-Based Linters (Oxlint)]]
- [ ] What are the **4 distinct layers** of frontend code quality, and what specific role does each tool (Compiler, Type Checker, Formatter, Linter) perform?
- [ ] Why is **Oxlint** 50x–100x faster than legacy ESLint, and how does Abstract Syntax Tree (AST) parsing in Rust achieve this?
- [ ] What catastrophic runtime bugs does the `react/rules-of-hooks` lint rule prevent?
- [ ] Why does exporting non-component constants from a React component file break Vite's Fast Refresh (Hot Module Reloading)?

---

## 🔷 TypeScript & Runtime Safety

### 1. TypeScript Core Fundamentals

#### Topics to Recall:
1. **Static Typing vs. Dynamic Typing:** Compile-time check (`tsc`) vs. runtime evaluation (V8 / Node.js).
2. **Type Erasure & Transpilation:** Why types occupy 0 bytes in compiled JavaScript and why types cannot be accessed as runtime values.
3. **`tsconfig.json` Core Directives:** What `target`, `module`, `strict`, `rootDir`, and `outDir` instruct the compiler to do.
4. **Primitive Types:** Annotating `string`, `number`, `boolean`, `null`, and `undefined`.
5. **Type Inference:** When TypeScript automatically detects the type vs. when to write explicit annotations.
6. **Arrays vs. Tuples:** Open-ended typed lists (`number[]`) vs. fixed-length and fixed-order pairs (`[number, string]`).
7. **Object Data Contracts (`interface` vs. `type`):** When to use each, extending interfaces, and defining contracts for entities.
8. **Immutability & Optionality:** `readonly` properties (preventing reassignment) and optional properties (`?`).
9. **Function Signatures:** Parameter typing, explicit return types, and the `void` return type.
10. **Typing Function Callbacks:** The `(param: Type) => ReturnType` signature and passing functions as arguments.
11. **Asynchronous TypeScript:** `Promise<T>`, why all `async` functions return a `Promise`, and checking for `null` before property access.
12. **Generics Demystified (`<T>`):** Type parameters as placeholders, avoiding `any`, and building reusable shapes like `ApiResponse<T>`.
13. **Literal Union Types:** Enforcing exact string or number options (`"HOME" | "AWAY"`) instead of broad types.
14. **Defensive Typing & Narrowing:** Why `any` is banned, using `unknown`, and narrowing with `typeof` and `in`.

👉 **Detailed Study Note:** [[typescript/core/typescript_basics|typescript_basics.md]]

---

### 2. TypeScript Compilation & Multi-tsconfig Architecture

#### Topics to Recall:
1. **No Runtime Execution:** Why neither Node.js nor web browsers execute TypeScript directly; all code must become plain JS.
2. **Backend vs. Frontend Conversion:** Why `tsc` emits `dist/*.js` for backend, while Vite bundles `dist/` and `tsc` only type-checks (`noEmit: true`) for frontend.
3. **`tsc` In-Memory Checking vs Type Erasure:** Why types stay active in AST memory during checking and are erased only during code emission.
4. **Why Multiple Configs:** Why backend needs only 1 `tsconfig.json`, while frontend requires 3 files (`tsconfig.json`, `tsconfig.app.json`, `tsconfig.node.json`) to isolate Browser DOM from Node.js build scripts.
5. **The 3 Error Watchers Timeline:** Editor Language Server (`tsserver`) while typing $\rightarrow$ Vite Dev Server (`esbuild`) during dev $\rightarrow$ `tsc -b` during production build.

👉 **Detailed Study Note:** [[typescript/core/tsconfig_conecpts|tsconfig_conecpts.md]]

### Zod (`typescript/zod/`)

#### [[typescript/zod/zod-schema-validation|Zod Runtime Schema Validation & Static Type Inference]]
- [ ] Why does TypeScript's static type system vanish at runtime (type erasure), and why is Zod required to validate external boundaries (API responses, forms)?
- [ ] How does `z.infer<typeof schema>` eliminate redundant interface declarations and keep TypeScript types in 100% sync with runtime validators?
- [ ] What is the difference between `.refine()` and `.transform()` in a Zod validation chain?
- [ ] Why should you prefer `schema.safeParse(data)` over `schema.parse(data)` when building production backend API middlewares or services?

---

## 🧭 Active Recall Deep-Dives & Roadmaps

- [[00-active-recall/backend-docker-qna|Backend, Database & Docker Mastery — Active Recall Q&A Guide (With Collapsible Answers)]]
- [[00-active-recall/system-design-scale-lab-roadmap|Future Project Blueprint: High-Throughput Distributed Systems & Scale Lab]]
