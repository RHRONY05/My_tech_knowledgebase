---
tags:
  - backend
  - authentication
  - jwt
  - security
last_reviewed: 2026-10-02
related_notes:
  - "[[react-auth-guards-and-routing]]"
  - "[[validation-defense-in-depth]]"
---

# Full-Stack Authentication Lifecycle (JWT + HttpOnly Cookies)

> **Active Recall Self-Test:**
> 1. Why is storing JWTs in browser `localStorage` vulnerable to XSS attacks, and how do `HttpOnly`, `SameSite`, and `Secure` cookie flags remediate this?
> 2. What is the dual-token architecture (Access Token vs Refresh Token), and why should Access Tokens have short lifespans (e.g. 15 mins)?
> 3. How does the initial client boot (`checkAuth` / `/auth/me`) prevent the "flash of unauthorized content" when a user refreshes the page?
> 4. Why must user verification (e.g. email confirmation) gate login rather than auto-authenticating users upon registration?

---

## 1. The Core Problem
### The Naive Approaches & Security Flaws

#### Anti-Pattern A: JWT in `localStorage`
The most common beginner approach is saving tokens returned from an API response into browser storage:
```javascript
localStorage.setItem('token', response.data.token);
```
- **XSS Vulnerability:** Any third-party dependency, malicious npm script, or Cross-Site Scripting (XSS) vulnerability can execute `localStorage.getItem('token')` and exfiltrate the token to an external attacker server.
- **CSRF vs XSS Tradeoff:** While `localStorage` avoids Cross-Site Request Forgery (CSRF), XSS token theft is catastrophic because tokens can be used from any machine indefinitely until expiry.

#### Anti-Pattern B: Long-Lived Stateless Tokens
Issuing a single JWT valid for 30 days without server-side revocation:
- If a user's token is compromised, the attacker retains full access for 30 days. Because JWTs are cryptographically self-contained, a stateless backend cannot invalidate the token without rotating secret keys (which logs out all users globally).

#### The Production Standard: Dual Token + `HttpOnly` Cookies
1. **Access Token (Short-lived: 15m):** Used to authorize API endpoints. Stored in memory or in a strict cookie.
2. **Refresh Token (Long-lived: 7d):** Stored inside an `HttpOnly`, `SameSite=Strict`, `Secure` cookie. Inaccessible to client JavaScript.
3. **Session Rehydration Endpoint (`/check-auth` / `/me`):** On application mount, the frontend calls the backend to verify the cookie session before rendering protected views.

---

## 2. The Mental Model

### Registration & Email Verification Lifecycle
```
[ User Browser ]                  [ Express Backend ]              [ Database / Mailer ]
       │                                   │                                 │
       │ 1. POST /register { email, pass } │                                 │
       ├──────────────────────────────────►│                                 │
       │                                   │ 2. Hash password (bcrypt)       │
       │                                   │ 3. Generate verificationToken   │
       │                                   │ 4. Save User (isVerified=false) │
       │                                   ├────────────────────────────────►│
       │                                   │ 5. Dispatch verification email  │
       │                                   │    with secure link + token     │
       │ 6. 201 Created { message }        │                                 │
       │◄──────────────────────────────────┤                                 │
       │ (User NOT authenticated yet)      │                                 │
       │                                   │                                 │
       │ 7. User clicks link in email:     │                                 │
       │    GET /verify-email?token=xyz    │                                 │
       ├──────────────────────────────────►│                                 │
       │                                   │ 8. Mark isVerified=true         │
       │                                   │    Invalidate token in DB       │
       │ 9. Redirects to /login (Ready)    │                                 │
       │◄──────────────────────────────────┤                                 │
```

### Login, Cookie Set & Rehydration (`checkAuth`)
```
[ User Browser ]                  [ Express Backend ]              [ Database / Auth ]
       │                                   │                                 │
       │ 1. POST /login { email, pass }    │                                 │
       ├──────────────────────────────────►│ 2. Find user & verify password  │
       │                                   │ 3. Check if isVerified === true │
       │                                   │ 4. Issue AccessToken + Refresh  │
       │ 5. HTTP 200 OK                    │                                 │
       │    Set-Cookie: token=jwt;         │                                 │
       │    HttpOnly; Secure; SameSite=Lax │                                 │
       │◄──────────────────────────────────┤                                 │
       │                                   │                                 │
   ─── User presses F5 (Page Refresh / Tab Reopened) ──────────────────────────
       │                                   │                                 │
       │ 6. Redux / App mounts:            │                                 │
       │    GET /auth/check-auth           │                                 │
       │    (Cookie sent automatically)    │                                 │
       ├──────────────────────────────────►│ 7. Verify cookie JWT signature  │
       │                                   │ 8. Fetch user profile from DB   │
       │ 9. HTTP 200 { user, role }        │                                 │
       │◄──────────────────────────────────┤                                 │
       │ 10. Redux: isAuthenticated=true   │                                 │
       │     Render protected layout       │                                 │
```

---

## 3. Production Code Breakdown

### A. Generating Secure Cookies in Backend Controller (`src/controllers/auth.controller.js`)
```javascript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';
import { User } from '../models/user.model.js';

// Centralized cookie configuration helper
const getCookieOptions = () => ({
  httpOnly: true, // Prevents client-side JS (XSS) from reading the cookie
  secure: process.env.NODE_ENV === 'production', // Transmitted ONLY over HTTPS
  sameSite: process.env.NODE_ENV === 'production' ? 'none' : 'lax', // Protects against CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days in milliseconds
});

export const loginUser = async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ success: false, message: 'Invalid email or password' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'Invalid email or password' });
    }

    // Gate login on email verification
    if (!user.isVerified) {
      return res.status(403).json({
        success: false,
        message: 'Account not verified. Please check your email for the verification link.',
      });
    }

    // Sign payload
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    // Remove password before sending user object back
    const userPayload = {
      _id: user._id,
      email: user.email,
      username: user.username,
      role: user.role,
    };

    return res
      .status(200)
      .cookie('token', token, getCookieOptions())
      .json({
        success: true,
        user: userPayload,
        message: 'Logged in successfully',
      });
  } catch (error) {
    next(error);
  }
};

export const logoutUser = async (req, res) => {
  return res
    .status(200)
    .clearCookie('token', getCookieOptions())
    .json({ success: true, message: 'Logged out successfully' });
};
```

### B. Session Verification Endpoint (`src/controllers/auth.controller.js`)
```javascript
export const checkAuth = async (req, res) => {
  // req.user was populated by authenticateToken middleware
  const user = await User.findById(req.user.id).select('-password');
  if (!user) {
    return res.status(404).json({ success: false, message: 'User not found' });
  }

  return res.status(200).json({
    success: true,
    user: {
      _id: user._id,
      email: user.email,
      username: user.username,
      role: user.role,
    },
  });
};
```

### C. JWT Verification Middleware (`src/middlewares/auth.middleware.js`)
```javascript
import jwt from 'jsonwebtoken';

export const authenticateToken = (req, res, next) => {
  // Extract cookie from cookie-parser
  const token = req.cookies?.token;

  if (!token) {
    return res.status(401).json({ success: false, message: 'Authentication required. No token provided.' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ success: false, message: 'Session expired. Please log in again.' });
    }
    return res.status(403).json({ success: false, message: 'Invalid or malformed token.' });
  }
};
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The `credentials: 'include'` / `withCredentials: true` Requirement
- **The Trap:** When making cross-origin requests (e.g. frontend on `http://localhost:5173` and backend on `http://localhost:3000`), the browser strips `HttpOnly` cookies unless explicitly configured on **both** sides.
- **The Fix:**
  - On Axios client: `axios.defaults.withCredentials = true;`
  - On Express CORS middleware:
    ```javascript
    app.use(cors({
      origin: process.env.CLIENT_ORIGIN, // e.g. "http://localhost:5173" (NEVER "*" with credentials)
      credentials: true,
    }));
    ```

### ⚠️ Gotcha 2: The `SameSite` & `Secure` Cookie Deployment Trap
- **The Trap:** In local development on HTTP (`http://localhost`), setting `secure: true` causes the browser to reject and ignore the cookie completely. Conversely, in production over cross-site HTTPS subdomains, setting `sameSite: 'lax'` can block cookies on cross-origin iframe/API contexts.
- **The Fix:** Conditionally configure cookie flags:
  ```javascript
  secure: process.env.NODE_ENV === 'production',
  sameSite: process.env.NODE_ENV === 'production' ? 'none' : 'lax'
  ```

### ⚠️ Gotcha 3: The Ghost Auth Flash on Initial Page Reload
- **The Trap:** A user logs in and hits refresh. On mount, the Redux store resets to its initial state (`isAuthenticated: false`). If your protected route guard renders immediately, it detects `isAuthenticated === false` and prematurely redirects the user to `/login`. A half-second later, `/check-auth` returns 200, causing an erratic UI jitter.
- **The Fix:** Maintain a three-state machine: `isCheckingAuth: true | false`. While `isCheckingAuth === true`, show a full-screen loader or skeleton. Never evaluate route redirection until `isCheckingAuth === false`.
