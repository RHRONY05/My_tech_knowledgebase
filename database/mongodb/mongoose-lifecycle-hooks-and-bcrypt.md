---
tags:
  - database
  - mongodb
  - mongoose
  - security
  - bcrypt
last_reviewed: 2026-10-02
related_notes:
  - "[[database/mongodb/mongoose-connection-and-fail-fast]]"
  - "[[backend/express/jwt-refresh-token-rotation]]"
  - "[[backend/express/auth-lifecycle-jwt-cookies]]"
---

# Mongoose Lifecycle Hooks, Schema Methods & Bcrypt Security

> [!CHECKLIST] Active Recall Self-Test
> - [ ] Why is `if (!this.isModified("password")) return next();` mandatory in a `pre("save")` hook?
> - [ ] What happens if you define a Mongoose middleware or schema method using an ES6 arrow function `() => {}`?
> - [ ] What is the architectural distinction between a Mongoose Model (Static methods) and a Document (Instance methods)?
> - [ ] Why must temporary reset/verification tokens be stored in the database as SHA-256 hashes instead of plain text?
> - [ ] What is the purpose of `user.save({ validateBeforeSave: false })` when updating internal tokens?

---

## 1. The Core Problem: The Password Double-Hashing Disaster

In poorly structured Node.js applications, security logic is scattered throughout route controllers or implemented with flawed Mongoose hooks:

1. **The Double-Hashing Catastrophe:** A developer hashes the password in `pre("save")`. When a user later updates their avatar (`user.avatar = newUrl; await user.save()`), the hook executes again, takes the *already hashed* string (`$2b$10$wK8v...`), and hashes it a second time. The password is permanently corrupted, locking the user out.
2. **Leaking Secrets via Missing Projections:** Calling `User.findById(id)` without negative projections returns the bcrypt hash and refresh tokens directly in the API payload, inadvertently exposing sensitive credentials over the wire.
3. **Leaked Reset Tokens in Data Breaches:** When generating email verification or password reset links, developers often store the raw token directly in MongoDB. If a database snapshot is leaked, attackers can immediately take over any account with an active reset token.
4. **Scattered Business Logic:** Writing `bcrypt.hash()` across registration, reset password, and update password controllers violates DRY and results in inconsistent salt rounds.

---

## 2. The Mental Model: "Fat Model, Skinny Controller" & Mongoose Lifecycle

Mongoose acts as the data guardian. By encapsulating hashing, comparisons, and token generation directly on the schema, controllers remain clean, declarative, and focused solely on HTTP coordination.

```
Document Mutation (Controller)
          │
          │ user.password = "newSecret123"
          ▼
┌─────────────────────────────────────────────────────────────┐
│ Mongoose .pre("save") Hook                                  │
│                                                             │
│   Is "password" modified? (this.isModified("password"))     │
│             │                                               │
│       NO ───┴───▶ Skip hashing ───┐                         │
│       YES                         │                         │
│        │                          │                         │
│        ▼                          │                         │
│   bcrypt.hash(password, 10)       │                         │
│        │                          │                         │
│        ▼                          ▼                         │
│   Replace plain text with hash    │                         │
└─────────────────────┬─────────────┴─────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ MongoDB Storage (BSON Document)                             │
│ Persists securely hashed password ($2b$10$...)               │
└─────────────────────────────────────────────────────────────┘
```

### Static Model Methods vs. Instance Document Methods

| Category | Target | Scope | Common Examples |
| :--- | :--- | :--- | :--- |
| **Model (Static)** | The Collection / Factory | Manages queries, multiple documents, or creation. | `User.findOne()`, `User.create()`, `User.findById()`, `User.exists()` |
| **Document (Instance)** | A Specific Record / Object | Has state and document-level behavior. | `user.save()`, `user.isPasswordCorrect()`, `user.generateTemporaryToken()` |

---

## 3. Production Code Breakdown

### A. Battle-Tested User Schema with Hooks & Methods

```javascript
// src/models/user.model.js
import mongoose, { Schema } from "mongoose";
import bcrypt from "bcrypt";
import crypto from "crypto";
import jwt from "jsonwebtoken";

const userSchema = new Schema(
  {
    username: {
      type: String,
      required: [true, "Username is required"],
      unique: true,
      lowercase: true,
      trim: true,
      index: true,
    },
    email: {
      type: String,
      required: [true, "Email is required"],
      unique: true,
      lowercase: true,
      trim: true,
      index: true,
    },
    password: {
      type: String,
      required: [true, "Password is required"],
    },
    avatar: {
      url: { type: String, required: true },
      localPath: { type: String }, // Path on disk before cleanup
    },
    refreshToken: {
      type: String,
    },
    emailVerificationToken: {
      type: String,
    },
    emailVerificationExpiry: {
      type: Date,
    },
  },
  {
    timestamps: true, // Automatically manages createdAt and updatedAt
  }
);

/**
 * Pre-Save Lifecycle Interceptor.
 * MUST use standard `function()` syntax to retain Mongoose `this` document context.
 */
userSchema.pre("save", async function (next) {
  // Guard Clause: Prevent double-hashing on non-password updates
  if (!this.isModified("password")) return next();

  // Salt rounds = 10 (Optimal balance of security & CPU overhead)
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

/**
 * Instance Method: Timing-safe bcrypt comparison.
 */
userSchema.methods.isPasswordCorrect = async function (plainPassword) {
  return await bcrypt.compare(plainPassword, this.password);
};

/**
 * Instance Method: Temporary Cryptographic Token Generator.
 * Used for Password Reset & Email Verification.
 * Returns raw token for email link, stores SHA-256 hash in DB.
 */
userSchema.methods.generateTemporaryToken = function () {
  // 1. Generate unguessable random hex string (40 chars)
  const unHashedToken = crypto.randomBytes(20).toString("hex");

  // 2. Hash using SHA-256 for persistent database storage
  const hashedToken = crypto
    .createHash("sha256")
    .update(unHashedToken)
    .digest("hex");

  // 3. Expiration window (20 minutes)
  const tokenExpiry = Date.now() + 20 * 60 * 1000;

  return { unHashedToken, hashedToken, tokenExpiry };
};

export const User = mongoose.model("User", userSchema);
```

### B. Controller Integration: Registration & Verification Flow

```javascript
// src/controllers/auth.controller.js
import { User } from "../models/user.model.js";
import { asyncHandler } from "../utils/asyncHandler.js";
import { ApiError } from "../utils/ApiError.js";
import { ApiResponse } from "../utils/ApiResponse.js";
import crypto from "crypto";

export const registerUser = asyncHandler(async (req, res) => {
  const { username, email, password } = req.body;

  // 1. Check for duplicates using MongoDB $or operator
  const existingUser = await User.findOne({
    $or: [{ username }, { email }],
  });

  if (existingUser) {
    throw new ApiError(409, "User with email or username already exists");
  }

  // 2. Create document (pre-save hook automatically hashes password)
  const user = await User.create({
    username,
    email,
    password,
    avatar: { url: "https://placehold.co/150", localPath: "" },
  });

  // 3. Generate cryptographic verification token
  const { unHashedToken, hashedToken, tokenExpiry } = user.generateTemporaryToken();
  user.emailVerificationToken = hashedToken;
  user.emailVerificationExpiry = tokenExpiry;

  // 4. Save token updates bypassing full schema validation
  await user.save({ validateBeforeSave: false });

  // 5. Exclude sensitive credentials in response payload
  const createdUser = await User.findById(user._id).select(
    "-password -refreshToken -emailVerificationToken"
  );

  return res.status(201).json(
    new ApiResponse(
      201,
      { user: createdUser, verificationToken: unHashedToken },
      "User registered successfully"
    )
  );
});

export const verifyEmail = asyncHandler(async (req, res) => {
  const { token } = req.params;

  // Hash incoming raw token to compare against stored hash
  const hashedToken = crypto.createHash("sha256").update(token).digest("hex");

  const user = await User.findOne({
    emailVerificationToken: hashedToken,
    emailVerificationExpiry: { $gt: Date.now() },
  });

  if (!user) {
    throw new ApiError(400, "Invalid or expired verification token");
  }

  user.emailVerificationToken = undefined;
  user.emailVerificationExpiry = undefined;
  user.isEmailVerified = true;
  await user.save({ validateBeforeSave: false });

  return res.status(200).json(new ApiResponse(200, {}, "Email verified successfully"));
});
```

---

## 4. Production Gotchas & Best Practices

### 1. Arrow Functions Destroy `this` Context
Never use arrow functions for Mongoose hooks (`pre`, `post`) or `methods`:
```javascript
// ❌ BROKEN: `this` points to global scope / undefined
userSchema.pre("save", async () => {
  this.password = await bcrypt.hash(this.password, 10);
});

// ✅ CORRECT: Standard function binds `this` to the Mongoose document
userSchema.pre("save", async function (next) {
  if (this.isModified("password")) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});
```

### 2. The Direct Query Bypass
Mongoose hooks like `pre("save")` **do NOT run** on direct update queries such as `User.findByIdAndUpdate()` or `User.updateOne()`.
- If you use `User.findByIdAndUpdate(id, { password: newPassword })`, the password will be written to MongoDB in **plain text**.
- **Rule:** For password changes, always retrieve the document, update the property, and call `await user.save()` so the hook executes.

### 3. Lean Queries for Read-Only Workloads
Mongoose wraps every returned document in a heavyweight prototype containing getters, setters, and change tracking.
- For high-throughput read-only queries (dashboards, lists), append `.lean()`:
  ```javascript
  const users = await User.find({ status: "active" }).select("-password").lean();
  ```
- `.lean()` returns plain JavaScript objects, improving query throughput by 3–5x.
