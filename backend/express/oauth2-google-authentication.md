---
tags:
  - backend
  - express
  - authentication
  - oauth
  - security
last_reviewed: 2026-10-02
related_notes:
  - "[[auth-lifecycle-jwt-cookies]]"
  - "[[validation-defense-in-depth]]"
---

# Google OAuth 2.0 Backend Architecture: ID Token Verification & Session Minting

> **Active Recall Self-Test:**
> 1. In Google Sign-In, what is the critical architectural difference between an **ID Token (`id_token`)**, an **Access Token (`access_token`)**, and a **Refresh Token (`refresh_token`)**?
> 2. Why does the backend verify Google ID Tokens cryptographically using `google-auth-library` rather than trusting the user profile payload sent from the frontend?
> 3. Why must the backend mint its **own custom JWT** after verifying Google credentials instead of using the Google ID Token for subsequent API requests?
> 4. How can you test Google OAuth backend endpoints using the **Google OAuth 2.0 Playground** without building a frontend UI?

---

## 1. The Core Problem
### The Flaws of Naive Social Login

When implementing "Sign in with Google", developers often make catastrophic security assumptions:

#### Anti-Pattern A: Trusting Frontend User Data
The frontend signs the user in via Google, extracts the user's email and name, and sends it directly to the backend:
```javascript
// ❌ CRITICAL SECURITY VULNERABILITY:
POST /api/auth/google
{
  "email": "ceo@google.com",
  "name": "Sundar Pichai"
}
```
- **Why this is fatal:** Anyone with Postman or `curl` can send arbitrary emails to this endpoint. An attacker can log into any user account, admin profile, or CEO account simply by typing their email into a JSON payload!

#### Anti-Pattern B: Using Google's ID Token for App Sessions
The frontend passes Google's `id_token` on every subsequent API call (e.g. `GET /api/movies`):
- **High Latency & Rate Limits:** The backend must contact Google's servers to verify the token on every single HTTP request, adding 100ms–300ms of external latency and hitting Google API rate limits.
- **Expiry Chaos:** Google ID tokens expire strictly after 1 hour with no automatic application-level session refresh.

#### The Architectural Solution: Cryptographic Verification & Local Session Minting
1. Frontend receives an immutable, cryptographically signed **ID Token** from Google and forwards it to the backend.
2. Backend verifies Google's cryptographic RSA signature locally using Google's public keys via `google-auth-library` (zero network calls after key caching).
3. Backend finds or creates the user in the database.
4. Backend issues its **own custom, secure JWT** stored in an `HttpOnly` cookie.

---

## 2. The Mental Model

### The OAuth 2.0 Credential Exchange Lifecycle
```
[ User Browser / React ]          [ Google Cloud Auth ]          [ Express Backend ]
          │                                 │                             │
          │ 1. Clicks "Sign in with Google" │                             │
          ├────────────────────────────────►│                             │
          │ 2. Prompts consent & verifies   │                             │
          │    User identity                │                             │
          │ 3. Returns signed `id_token`    │                             │
          │◄────────────────────────────────┤                             │
          │                                 │                             │
          │ 4. POST /api/auth/google        │                             │
          │    { idToken: "eyJhbGciOi..." } │                             │
          ├─────────────────────────────────┼────────────────────────────►│
          │                                 │                             │ 5. Verifies signature
          │                                 │                             │    using google-auth-library
          │                                 │                             │    & AUDIENCE (Client ID)
          │                                 │                             │ 6. Extracts { email, sub }
          │                                 │                             │ 7. Finds or creates User in DB
          │                                 │                             │ 8. Mints Custom App JWT
          │ 9. HTTP 200 OK                  │                             │
          │    Set-Cookie: token=AppJWT;    │                             │
          │    HttpOnly; Secure             │                             │
          │◄────────────────────────────────┼─────────────────────────────┤
          │                                 │                             │
   ─── Subsequent API Requests (e.g. GET /api/bookings) ───────────────────
          │                                                               │
          │ 10. GET /api/bookings (Cookie sent automatically)             │
          ├──────────────────────────────────────────────────────────────►│
          │                                                               │ 11. Backend verifies its
          │                                                               │     own internal JWT!
          │                                                               │     (Google is never contacted!)
```

### The Three Token Types
| Token | Purpose | Analogy | Used For |
|---|---|---|---|
| **ID Token (`id_token`)** | Proof of Identity | Digital Passport | Proves *who* the user is (contains email, name, avatar). Used to authenticate. |
| **Access Token (`access_token`)** | Permission / Scopes | Hotel Keycard | Gives permission to access Google APIs (Google Drive, Calendar). |
| **Refresh Token (`refresh_token`)** | Renewal | Master Key in Vault | Long-lived token used to obtain fresh Access Tokens when they expire. |

---

## 3. Production Code Breakdown

### A. Environment Configuration (`.env`)
```bash
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
JWT_SECRET=super_secret_production_app_key_84729
```

### B. Controller Verification & Session Minting (`src/controllers/auth.controller.js`)
```javascript
import { OAuth2Client } from 'google-auth-library';
import jwt from 'jsonwebtoken';
import pool from '../config/db.js';

const googleClient = new OAuth2Client(process.env.GOOGLE_CLIENT_ID);

export const googleAuthCallback = async (req, res, next) => {
  try {
    const { idToken } = req.body;

    if (!idToken) {
      return res.status(400).json({ success: false, message: 'Google ID Token is required' });
    }

    // 1. Cryptographically verify the Google ID Token
    const ticket = await googleClient.verifyIdToken({
      idToken,
      audience: process.env.GOOGLE_CLIENT_ID, // ⚠️ Rejects tokens generated for other apps!
    });

    const payload = ticket.getPayload();
    if (!payload || !payload.email) {
      return res.status(400).json({ success: false, message: 'Invalid token payload' });
    }

    const { email, name, sub: googleId } = payload;

    // 2. Find or create user in PostgreSQL
    let userQuery = 'SELECT id, email, name, role FROM users WHERE google_id = $1 OR email = $2;';
    let userResult = await pool.query(userQuery, [googleId, email]);

    let user;

    if (userResult.rows.length === 0) {
      // First-time user -> Register account
      const insertQuery = `
        INSERT INTO users (email, name, google_id)
        VALUES ($1, $2, $3)
        RETURNING id, email, name, role;
      `;
      const insertResult = await pool.query(insertQuery, [email, name, googleId]);
      user = insertResult.rows[0];
    } else {
      user = userResult.rows[0];
      // Link Google ID if user previously registered via email
      if (!user.google_id) {
        await pool.query('UPDATE users SET google_id = $1 WHERE id = $2;', [googleId, user.id]);
      }
    }

    // 3. Mint internal application session JWT
    const appToken = jwt.sign(
      { id: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    // 4. Return secure HttpOnly cookie
    return res
      .status(200)
      .cookie('token', appToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: process.env.NODE_ENV === 'production' ? 'none' : 'lax',
        maxAge: 7 * 24 * 60 * 60 * 1000,
      })
      .json({
        success: true,
        user: { id: user.id, email: user.email, name: user.name, role: user.role },
        message: 'Authenticated successfully with Google',
      });
  } catch (error) {
    next(error);
  }
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Audience Verification Hole
- **The Trap:** Calling `verifyIdToken({ idToken })` without passing `audience: process.env.GOOGLE_CLIENT_ID`.
- **The Vulnerability:** An attacker generates a valid ID token using their own malicious app on Google Cloud, then submits it to your API. Google signs it, so the signature is valid. Without the `audience` check, your backend accepts the foreign token and logs the attacker in!
- **The Rule:** Always specify `audience: process.env.GOOGLE_CLIENT_ID` so tokens intended for other applications are strictly rejected.

### ⚠️ Gotcha 2: Testing via Google OAuth Playground
- **The Trap:** Believing you must build an entire React frontend just to test your backend Google OAuth route.
- **The Solution:** Use the [Google OAuth 2.0 Playground](https://developers.google.com/oauthplayground/):
  1. Click Gear Icon $\rightarrow$ Check *"Use your own OAuth credentials"*. Enter your Client ID and Secret.
  2. Authorize `email`, `profile`, and `openid` scopes.
  3. Exchange authorization code for tokens.
  4. Copy the raw `id_token` string from the JSON response and POST it to your backend via Postman!

### ⚠️ Gotcha 3: The Account Linking Trap
- **The Trap:** Creating a duplicate user account if a customer previously registered with email/password and later clicks "Sign in with Google".
- **The Fix:** Query by `google_id OR email`. If a user exists with matching email, link their `google_id` to the existing account rather than throwing an error or creating a duplicate row.
