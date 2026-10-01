---
tags:
  - frontend
  - react
  - auth
  - routing
  - redux
last_reviewed: 2026-10-02
related_notes:
  - "[[react-router-v6-architecture]]"
  - "[[auth-lifecycle-jwt-cookies]]"
  - "[[redux-toolkit-async-thunks]]"
---

# Protected Routes, Role Guards & Redux State Synchronization

> **Active Recall Self-Test:**
> 1. Why does evaluating route protection during initial application boot without an `isLoading` gate cause a momentary "flash redirect" to `/login`?
> 2. How does a declarative `<CheckAuth>` wrapper component enforce Role-Based Access Control (RBAC) across public, authenticated, and admin routes?
> 3. What race condition happens when a user clicks "Login", Redux updates `isAuthenticated = true`, and both `useEffect` and `<CheckAuth>` attempt navigation simultaneously?
> 4. Why should unauthenticated users be redirected with `state: { from: location }` so they return to their requested page after logging in?

---

## 1. The Core Problem
### The Authentication Race Condition & Flash Bug

When integrating client-side routing (React Router) with global state (Redux Toolkit) and cookie authentication:

#### Flaw 1: The Initial State Flash
When a logged-in user refreshes the page (`F5`), Redux reinitializes its in-memory store from scratch:
```javascript
const initialState = { isAuthenticated: false, user: null }; // Default boot state
```
If your router checks `if (!isAuthenticated) return <Navigate to="/login" />` immediately on mount, the user is violently kicked to `/login` before the background `/auth/check-auth` API request has even completed!

#### Flaw 2: The Double Navigation War
Upon submitting a login form, two separate entities often try to navigate at the exact same moment:
1. The `onSubmit` handler in `LoginPage.jsx` calls `navigate('/shop/home')`.
2. A `useEffect([isAuthenticated])` inside `App.jsx` detects the state change and calls `navigate(...)`.
3. The `<CheckAuth>` guard evaluates and fires `<Navigate to="..." replace />`.
This causes conflicting history pushes, cancelled transitions, or redirect loops.

#### The Architectural Solution: Unified Declarative Route Guard
1. Enforce a **3-State Machine** in Redux: `isLoading: true | false`, `isAuthenticated: true | false`, `user: object | null`.
2. Block routing evaluation with a full-screen loading skeleton while `isLoading === true`.
3. Use a single, centralized `<CheckAuth>` guard wrapping route hierarchies to handle all redirects declaratively.

---

## 2. The Mental Model

### The 3-State Routing Gatekeeper
```
                 [ User Requests Route (e.g. /admin/products) ]
                                      │
                                      ▼
                        Is Redux `isLoading === true`?
                        (Background session rehydration)
                                 /          \
                              YES            NO
                              /                \
               [ Render Skeleton / Loader ]   [ Evaluate <CheckAuth> Matrix ]
```

### The Declarative Redirection Matrix
```
User State             Target Route            Action Taken
────────────────────────────────────────────────────────────────────────
Unauthenticated        Protected Route         Redirect ──► /auth/login
                       (/shop/checkout)        (Preserves return URL in location.state)

Authenticated          Auth Route              Redirect ──► Role Home
(Regular User)         (/auth/login)           (/shop/home)

Authenticated          Admin Route             Redirect ──► /unauthorized
(Regular User)         (/admin/dashboard)      (or /shop/home)

Authenticated          Admin Route             Allow ────► Render <AdminLayout />
(Admin Role)           (/admin/dashboard)

Authenticated          Storefront Route        Redirect ──► /admin/dashboard
(Admin Role)           (/shop/home)            (Keeps admin in admin workspace)
```

---

## 3. Production Code Breakdown

### A. Centralized `<CheckAuth>` Guard (`src/components/common/CheckAuth.jsx`)
```jsx
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';

/**
 * Universal Route Guard Component
 * Enforces authentication and Role-Based Access Control (RBAC)
 */
export default function CheckAuth({ isAuthenticated, user, children }) {
  const location = useLocation();
  const currentPath = location.pathname;

  // 1. Unauthenticated users trying to access protected routes
  if (!isAuthenticated) {
    if (!currentPath.includes('/login') && !currentPath.includes('/register')) {
      // Save intended destination so login can redirect them back!
      return <Navigate to="/auth/login" state={{ from: location }} replace />;
    }
  }

  // 2. Authenticated users trying to access login/register pages
  if (isAuthenticated) {
    if (currentPath.includes('/login') || currentPath.includes('/register')) {
      if (user?.role === 'admin') {
        return <Navigate to="/admin/dashboard" replace />;
      }
      return <Navigate to="/shop/home" replace />;
    }
  }

  // 3. Regular users trying to access Admin routes
  if (isAuthenticated && user?.role !== 'admin' && currentPath.startsWith('/admin')) {
    return <Navigate to="/unauthorized" replace />;
  }

  // 4. Admin users browsing normal customer storefront
  if (isAuthenticated && user?.role === 'admin' && currentPath.startsWith('/shop')) {
    return <Navigate to="/admin/dashboard" replace />;
  }

  // All checks passed -> render child layout / component
  return <>{children}</>;
}
```

### B. Integrating the Guard into App Routes (`src/App.jsx`)
```jsx
import React, { useEffect } from 'react';
import { Routes, Route } from 'react-router-dom';
import { useSelector, useDispatch } from 'react-redux';
import { checkAuth } from './store/auth-slice';

import CheckAuth from './components/common/CheckAuth';
import FullScreenLoader from './components/common/FullScreenLoader';

import AuthLayout from './layouts/AuthLayout';
import AdminLayout from './layouts/AdminLayout';
import ShoppingLayout from './layouts/ShoppingLayout';
import LoginPage from './pages/auth/LoginPage';
import AdminDashboard from './pages/admin/AdminDashboard';
import ShopHome from './pages/shop/ShopHome';

export default function App() {
  const dispatch = useDispatch();
  const { user, isAuthenticated, isLoading } = useSelector((state) => state.auth);

  // Rehydrate auth cookie session on boot
  useEffect(() => {
    dispatch(checkAuth());
  }, [dispatch]);

  // ⚠️ CRITICAL: Block rendering while checking session cookie!
  if (isLoading) {
    return <FullScreenLoader message="Authenticating session..." />;
  }

  return (
    <Routes>
      {/* Auth routes guarded */}
      <Route
        path="/auth"
        element={
          <CheckAuth isAuthenticated={isAuthenticated} user={user}>
            <AuthLayout />
          </CheckAuth>
        }
      >
        <Route path="login" element={<LoginPage />} />
      </Route>

      {/* Admin routes guarded */}
      <Route
        path="/admin"
        element={
          <CheckAuth isAuthenticated={isAuthenticated} user={user}>
            <AdminLayout />
          </CheckAuth>
        }
      >
        <Route path="dashboard" element={<AdminDashboard />} />
      </Route>

      {/* Shop routes guarded */}
      <Route
        path="/shop"
        element={
          <CheckAuth isAuthenticated={isAuthenticated} user={user}>
            <ShoppingLayout />
          </CheckAuth>
        }
      >
        <Route path="home" element={<ShopHome />} />
      </Route>
    </Routes>
  );
}
```

### C. Redirecting Back to Original Destination on Login (`src/pages/auth/LoginPage.jsx`)
```jsx
import React from 'react';
import { useLocation, useNavigate } from 'react-router-dom';
import { useDispatch } from 'react-redux';
import { loginUser } from '../../store/auth-slice';

export default function LoginPage() {
  const dispatch = useDispatch();
  const navigate = useNavigate();
  const location = useLocation();

  // Retrieve previous URL if user was redirected here by CheckAuth
  const fromDestination = location.state?.from?.pathname || '/shop/home';

  const handleLoginSubmit = async (formData) => {
    try {
      const result = await dispatch(loginUser(formData)).unwrap();
      
      // Navigate to saved destination or role default
      if (result.user.role === 'admin') {
        navigate('/admin/dashboard', { replace: true });
      } else {
        navigate(fromDestination, { replace: true });
      }
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  // ... render form
}
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Props Destructuring Bug in Wrapper Components
- **The Trap:** Writing `const CheckAuth = (isAuthenticated, user, children) => { ... }`.
- **The Bug:** In React, components receive a single `props` object. Here, `isAuthenticated` becomes the entire props object (`{ isAuthenticated: false, user: null, children: ... }`), which is truthy in JavaScript! As a result, unauthenticated visitors are treated as authenticated!
- **The Fix:** Always destructure inside parentheses:
  ```jsx
  const CheckAuth = ({ isAuthenticated, user, children }) => { ... }
  ```

### ⚠️ Gotcha 2: The Redirection Bounce Loop
- **The Trap:** A rule directs `/shop` to `/admin/dashboard`, and another rule directs `/admin/dashboard` back to `/shop`.
- **The Consequence:** The browser crashes or locks up with `ERR_TOO_MANY_REDIRECTS`.
- **The Fix:** Ensure each role has a single deterministic landing route, and verify that the destination route doesn't match any exclusion condition.

### ⚠️ Gotcha 3: Stale User Permissions in Redux
- **The Trap:** A user's role is demoted from `admin` to `user` in the database, but their client Redux store retains `{ role: 'admin' }` until they log out.
- **The Fix:** Every sensitive admin API call on the backend must verify the user's role directly against the database or verified JWT payload via backend authorization middleware. Client guards are strictly for UX navigation; **the backend remains the sole authority on permissions**.
