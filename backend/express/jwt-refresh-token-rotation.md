---
tags:
  - backend
  - express
  - auth
  - jwt
  - security
last_reviewed: 2026-10-02
related_notes:
  - "[[backend/express/auth-lifecycle-jwt-cookies]]"
  - "[[database/mongodb/mongoose-lifecycle-hooks-and-bcrypt]]"
  - "[[backend/express/centralized-error-handling-and-api-contracts]]"
---

# JWT Dual-Token Lifecycle & Refresh Token Rotation

> [!CHECKLIST] Active Recall Self-Test
> - [ ] What is the fundamental difference between an Opaque Token (Stateful) and a JWT (Stateless)?
> - [ ] Why should Access Tokens have an extremely short lifespan (e.g., 10–15 minutes) while Refresh Tokens live for days?
> - [ ] Why is storing tokens in `localStorage` an anti-pattern, and how do `HttpOnly`, `SameSite=Strict` cookies prevent token theft?
> - [ ] What is Refresh Token Rotation, and how does it detect token replay attacks by malicious actors?
> - [ ] Why does verifying a refresh token still require querying the database if the signature is mathematically valid?

---

## 1. The Core Problem: The Stateless Authentication Dilemma

Modern distributed applications cannot afford a database lookup on every single API request. However, purely stateless tokens introduce a severe security flaw:

1. **The Inability to Revoke:** Once an asymmetric or HMAC-signed JWT is minted, it remains valid until its `exp` timestamp. If a token is stolen via XSS, the attacker has unrestricted access until expiration; the server has no kill switch.
2. **The Session Bottleneck:** Storing sessions in a database (opaque tokens) requires an I/O query on every incoming HTTP call (`SELECT * FROM sessions WHERE id = ?`), creating a database bottleneck under heavy traffic.
3. **The XSS Vulnerability:** Developers frequently store JWTs in browser `localStorage` or `sessionStorage`. Any third-party npm package or malicious script injected via XSS can immediately read `window.localStorage.getItem('token')` and exfiltrate credentials.
4. **Token Replay Vulnerability:** If a long-lived refresh token is intercepted, the attacker can indefinitely generate access tokens unless the server implements rotation with reuse detection.

---

## 2. The Mental Model: Dual-Token Architecture & Silent Renewal

The industry standard combines the **speed of stateless JWTs** with the **revocation security of stateful persistence**:

1. **Access Token (Short-Lived, Stateless, 15m):** Carries claims (`_id`, `role`, `email`). Verified strictly via CPU mathematical signature verification—zero DB queries.
2. **Refresh Token (Long-Lived, Hybrid, 7–30d):** Stored in the database on the user document (or dedicated session collection) and transmitted strictly via `HttpOnly`, `Secure` cookies. Used exclusively at `/api/v1/auth/refresh-token`.

```
Client (Browser)                           Server API                        Database
      │                                        │                                │
      │──── 1. POST /login (credentials) ────▶│                                │
      │                                        │─── Verify bcrypt hash ────────▶│
      │                                        │◀── User Valid ─────────────────│
      │                                        │─── Save Hashed RefreshToken ──▶│
      │◀─── Set HttpOnly Cookie (Refresh) ─────│                                │
      │     Return JSON AccessToken (15m)      │                                │
      │                                        │                                │
      │==== Normal API Operation (0 - 15m) =====================================│
      │                                        │                                │
      │──── 2. GET /api/data (Bearer Token) ──▶│ Verify Signature (Fast, no DB) │
      │◀─── 200 OK (Data payload) ─────────────│                                │
      │                                        │                                │
      │==== Token Expiry Event (Minute 16) =====================================│
      │                                        │                                │
      │──── 3. GET /api/data (Expired Token) ─▶│ Signature Expired              │
      │◀─── 401 Unauthorized (TokenExpired) ───│                                │
      │                                        │                                │
      │==== Silent Rotation Protocol ===========================================│
      │                                        │                                │
      │──── 4. POST /auth/refresh-token ──────▶│                                │
      │        (Cookie: refreshToken)          │─── Verify JWT Signature        │
      │                                        │─── Compare with DB Token ─────▶│
      │                                        │◀── Token Matched ──────────────│
      │                                        │─── Issue NEW Access + Refresh  │
      │                                        │─── Update DB with NEW Refresh ─▶│
      │◀─── 200 OK: New Access Token ──────────│                                │
      │     Set-Cookie: NEW Refresh Token      │                                │
      │                                        │                                │
      │──── 5. Retry GET /api/data ───────────▶│ Verified with NEW Token        │
      │◀─── 200 OK (Seamless to User) ─────────│                                │
```

---

## 3. Production Code Breakdown

### A. Secure Cookie Configuration

```javascript
// src/constants/cookie.constant.js
export const REFRESH_COOKIE_OPTIONS = Object.freeze({
  httpOnly: true, // Prevents access from client-side JavaScript (Mitigates XSS)
  secure: process.env.NODE_ENV === "production", // Transmit only over HTTPS in production
  sameSite: "strict", // Prevents transmission on cross-site requests (Mitigates CSRF)
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days in milliseconds
  path: "/api/v1/auth", // Restricts cookie transmission strictly to auth endpoints
});

export const ACCESS_COOKIE_OPTIONS = Object.freeze({
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "strict",
  maxAge: 15 * 60 * 1000, // 15 minutes
});
```

### B. Dual Token Generation Helpers

```javascript
// src/models/user.model.js
import mongoose, { Schema } from "mongoose";
import jwt from "jsonwebtoken";

const userSchema = new Schema(
  {
    username: { type: String, required: true, unique: true, lowercase: true, trim: true },
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true },
    refreshToken: { type: String }, // Persisted for revocation checks
  },
  { timestamps: true }
);

// Short-lived Access Token (Fast verification)
userSchema.methods.generateAccessToken = function () {
  return jwt.sign(
    {
      _id: this._id,
      email: this.email,
      username: this.username,
    },
    process.env.ACCESS_TOKEN_SECRET,
    { expiresIn: process.env.ACCESS_TOKEN_EXPIRY || "15m" }
  );
};

// Long-lived Refresh Token (Minimal payload)
userSchema.methods.generateRefreshToken = function () {
  return jwt.sign(
    { _id: this._id },
    process.env.REFRESH_TOKEN_SECRET,
    { expiresIn: process.env.REFRESH_TOKEN_EXPIRY || "7d" }
  );
};

export const User = mongoose.model("User", userSchema);
```

### C. Refresh Token Rotation Handler

```javascript
// src/controllers/auth.controller.js
import jwt from "jsonwebtoken";
import { User } from "../models/user.model.js";
import { asyncHandler } from "../utils/asyncHandler.js";
import { ApiError } from "../utils/ApiError.js";
import { ApiResponse } from "../utils/ApiResponse.js";
import { REFRESH_COOKIE_OPTIONS, ACCESS_COOKIE_OPTIONS } from "../constants/cookie.constant.js";

/**
 * Rotates the refresh token and mints a new access token.
 * Detects token reuse to invalidate compromised sessions.
 */
export const refreshAccessToken = asyncHandler(async (req, res) => {
  const incomingRefreshToken = req.cookies?.refreshToken || req.body?.refreshToken;

  if (!incomingRefreshToken) {
    throw new ApiError(401, "Unauthorized: Refresh token missing");
  }

  // 1. Verify token signature
  let decodedToken;
  try {
    decodedToken = jwt.verify(incomingRefreshToken, process.env.REFRESH_TOKEN_SECRET);
  } catch (error) {
    throw new ApiError(401, "Unauthorized: Invalid or expired refresh token");
  }

  // 2. Fetch user from DB
  const user = await User.findById(decodedToken._id);
  if (!user) {
    throw new ApiError(401, "Unauthorized: User session not found");
  }

  // 3. REUSE DETECTION: Ensure incoming token matches the one stored in DB
  if (incomingRefreshToken !== user.refreshToken) {
    // SECURITY ALERT: Token reuse detected! Someone used an old/revoked token.
    // Invalidate stored token immediately to lock out attacker.
    user.refreshToken = undefined;
    await user.save({ validateBeforeSave: false });
    throw new ApiError(403, "Forbidden: Compromised token detected. Please login again.");
  }

  // 4. Generate NEW Access & Refresh Token (Rotation)
  const newAccessToken = user.generateAccessToken();
  const newRefreshToken = user.generateRefreshToken();

  // 5. Persist the rotated token
  user.refreshToken = newRefreshToken;
  await user.save({ validateBeforeSave: false });

  // 6. Return response with fresh cookies
  return res
    .status(200)
    .cookie("accessToken", newAccessToken, ACCESS_COOKIE_OPTIONS)
    .cookie("refreshToken", newRefreshToken, REFRESH_COOKIE_OPTIONS)
    .json(
      new ApiResponse(
        200,
        { accessToken: newAccessToken },
        "Access token refreshed successfully"
      )
    );
});
```

---

## 4. Production Gotchas & Best Practices

### 1. Token Reuse Detection
If an attacker intercepts a Refresh Token and uses it before the legitimate user does, the legitimate user will eventually attempt to refresh with an old token. By checking `incomingRefreshToken !== user.refreshToken`, your server identifies this collision, wipes `user.refreshToken`, and forces a re-login, terminating the stolen session.

### 2. The `validateBeforeSave: false` Optimization
When updating tokens (`user.refreshToken = newRefreshToken; await user.save({ validateBeforeSave: false });`), bypass schema validation. Mongoose validation is designed for user-submitted form data. Re-validating complex password or subdocument rules during high-frequency token rotation wastes CPU and fails if unrelated fields fail validation.

### 3. Asymmetric Keys (RS256 vs. HS256)
- **HS256 (Symmetric):** The same secret key signs and verifies. If an auth service and an external analytics microservice both verify tokens, you must share the private secret with both services.
- **RS256 (Asymmetric):** The Auth service holds the **Private Key** (signs tokens). All downstream microservices only possess the **Public Key** (verifies signature). Downstream microservices can never forge tokens even if breached.

### 4. Clock Skew Handling
Server clocks can drift by a few seconds. When checking token expiration, libraries like `jsonwebtoken` allow setting `clockTolerance: 5` (seconds) to prevent transient network rejections during clock synchronization.
