---
tags:
  - backend
  - node
  - architecture
  - best-practices
last_reviewed: 2026-10-02
related_notes:
  - "[[backend/express/express-middleware-architecture]]"
  - "[[backend/express/centralized-error-handling-and-api-contracts]]"
  - "[[database/mongodb/mongoose-connection-and-fail-fast]]"
---

# Production Node.js Architecture & Clean Enterprise Structure

> [!CHECKLIST] Active Recall Self-Test
> - [ ] Why should you strictly distinguish `dependencies` from `devDependencies` in containerized production deployments?
> - [ ] What is the architectural role of `Object.freeze()` when declaring enums or constant maps in JavaScript?
> - [ ] Why does an API follow MVC where the "View" is replaced by JSON API Contracts and Validators?
> - [ ] Where does `public/temp` fit into the file upload lifecycle, and why is `localPath` tracked in models?
> - [ ] Why must `process.exit(1)` ONLY run during bootstrap pre-flight checks and never inside request handlers?

---

## 1. The Core Problem: Architectural Rot & Magic Strings

In early-stage backend projects, developers frequently make architectural compromises that cause severe production regressions:
1. **The Spaghetti Monolith:** Routing, database queries, business validation, and response formatting are all packed into a single controller file or inside `index.js`.
2. **Magic Strings & Unsynchronized State:** Statuses like `"pending"`, `"in_progress"`, and `"completed"` are hardcoded as raw strings across models, routes, and controllers. A typo like `"in-progress"` fails silently at runtime without compile-time warnings.
3. **Bloated Production Containers:** Installing `devDependencies` (like `nodemon`, `jest`, `prettier`) inside production Docker images balloons image size by 300MB+ and introduces security vulnerabilities from untracked dev tooling.
4. **Environment Secret Exposure:** Committing API keys and connection strings to Git or scattering `process.env` calls across fifty files without centralized validation leads to crashes at runtime when an environment variable is omitted.

---

## 2. The Mental Model: Enterprise Modular MVC for APIs

In RESTful and JSON-driven backend services, the classic MVC pattern is adapted: the "View" is not HTML, but **standardized JSON contracts** validated by strict DTO/Validator layers.

```
Incoming Request (HTTP / HTTPS)
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                       Express App                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Global Middlewares: cors(), helmet(), express.json()  │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             │                               │
│                             ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Route Matcher (`src/routes/user.routes.js`)           │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             │                               │
│                             ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Gatekeepers: authMiddleware, validate(registerSchema) │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             │                               │
│                             ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Controller Logic (`src/controllers/user.controller.js`)│  │
│  │   • Unwraps sanitized body                            │  │
│  │   • Orchestrates services/models                      │  │
│  └─────────────┬───────────────────────────┬─────────────┘  │
│                │                           │                │
│                ▼                           ▼                │
│   ┌─────────────────────────┐ ┌─────────────────────────┐  │
│   │ Mongoose Models         │ │ Utils / Helpers         │  │
│   │ (`src/models/*.js`)     │ │ (`src/utils/*.js`)      │  │
│   │ Schema & Business Rules │ │ asyncHandler, ApiError  │  │
│   └────────────┬────────────┘ └─────────────────────────┘  │
│                │                                            │
│                ▼                                            │
│   ┌─────────────────────────┐                               │
│   │ Database (MongoDB/SQL)  │                               │
│   └─────────────────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

### Directory Taxonomy

```
project-root/
├── public/
│   └── temp/              # Ephemeral disk cache for incoming uploads (cleaned post-upload)
├── src/
│   ├── config/            # Centralized, typed environment config
│   ├── constants/         # Frozen Enums & domain constants (Single Source of Truth)
│   ├── controllers/       # HTTP Request Orchestration (Skinny controllers)
│   ├── db/                # Database bootstrap & connection lifecycle
│   ├── middlewares/       # Auth guards, rate limiting, request context
│   ├── models/            # Database schemas, hooks, domain methods
│   ├── routes/            # Route declarations & middleware bindings
│   ├── utils/             # Reusable primitives (ApiError, ApiResponse, asyncHandler)
│   ├── validators/        # Request payload schemas (Zod or express-validator)
│   ├── app.js             # Express app setup, global middleware binding
│   └── index.js           # Server entry point & fail-fast bootstrap
├── .env.example           # Documented template of all required environment variables
├── .gitignore             # Excludes node_modules, .env, public/temp
└── package.json
```

---

## 3. Production Code Breakdown

### A. Immutable Enums & Constants Pattern

Avoid hardcoding strings. Use `Object.freeze()` to lock the enum at runtime and `Object.values()` to generate validation arrays dynamically.

```javascript
// src/constants/roles.constant.js
/**
 * Single source of truth for user authorization roles.
 * Frozen to prevent accidental runtime mutation.
 */
export const UserRolesEnum = Object.freeze({
  ADMIN: "admin",
  PROJECT_MANAGER: "project_manager",
  MEMBER: "member",
});

/**
 * Array of valid roles for Mongoose enum validators & Zod schemas.
 * Automatically stays synchronized when UserRolesEnum changes.
 */
export const AvailableUserRoles = Object.freeze(Object.values(UserRolesEnum));

export const TaskStatusEnum = Object.freeze({
  TODO: "todo",
  IN_PROGRESS: "in_progress",
  UNDER_REVIEW: "under_review",
  COMPLETED: "completed",
});

export const AvailableTaskStatuses = Object.freeze(Object.values(TaskStatusEnum));
```

### B. Fail-Fast Bootstrapping Separation (`app.js` vs `index.js`)

Decouple the Express app definition (`app.js`) from the network listener and database connection (`index.js`). This enables isolated integration testing with `supertest` without starting real port listeners.

```javascript
// src/app.js
import express from "express";
import cors from "cors";
import { errorHandler } from "./middlewares/error.middleware.js";
import userRouter from "./routes/user.routes.js";

const app = express();

app.use(cors({ origin: process.env.CORS_ORIGIN, credentials: true }));
app.use(express.json({ limit: "16kb" }));
app.use(express.urlencoded({ extended: true, limit: "16kb" }));
app.use(express.static("public"));

// Route declarations
app.use("/api/v1/users", userRouter);

// Global Error Handler (MUST have 4 arguments: (err, req, res, next))
app.use(errorHandler);

export default app;
```

```javascript
// src/index.js
import dotenv from "dotenv";
import app from "./app.js";
import connectDB from "./db/index.js";

dotenv.config({ path: "./.env" });

const PORT = process.env.PORT || 8000;

// Fail-Fast: Never start the HTTP listener if the DB cannot connect
connectDB()
  .then(() => {
    const server = app.listen(PORT, () => {
      console.log(`🚀 Server running on port: ${PORT}`);
    });

    const shutdown = () => {
      console.log("Shutting down gracefully...");
      server.close(() => {
        console.log("HTTP server closed.");
        process.exit(0);
      });
    };

    process.on("SIGINT", shutdown);
    process.on("SIGTERM", shutdown);
  })
  .catch((err) => {
    console.error("💥 Fatal DB connection failure:", err);
    process.exit(1);
  });
```

---

## 4. Production Gotchas & Best Practices

### 1. `dependencies` vs. `devDependencies` in Docker
When packaging for production in Docker, never run a bare `npm install`. Use multi-stage builds and install only production dependencies in the final image:
```dockerfile
# Builder stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Production runner stage
FROM node:20-alpine AS runner
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev  # Strips nodemon, prettier, testing tools
COPY --from=builder /app/src ./src
CMD ["node", "src/index.js"]
```

### 2. The `public/temp` Zombie File Problem
When users upload avatars via Multer, files land in `public/temp` with a unique disk path (`localPath`). If your Cloudinary or S3 upload fails, or if you forget to run `fs.unlinkSync(localPath)` in a `finally` block, your server will silently run out of disk space (`ENOSPC`).
- **Rule:** Always delete `localPath` in a `try/finally` block after cloud upload or upon error.

### 3. Mutating Enums
Without `Object.freeze()`, any team member can inadvertently write `UserRolesEnum.ADMIN = "superadmin"` in a test or controller, corrupting authentication globally for all subsequent requests.

### 4. Process Exits
Never call `process.exit()` within route handlers or middleware. An unhandled exception or bad input from a single user will terminate the process for thousands of active users. Reserve `process.exit(1)` solely for **unrecoverable bootstrap pre-flight errors** (missing DB connection, invalid port, missing critical environment variables).
