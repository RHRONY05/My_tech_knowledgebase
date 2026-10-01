---
tags:
  - frontend
  - react
  - routing
  - architecture
last_reviewed: 2026-10-02
related_notes:
  - "[[react-auth-guards-and-routing]]"
  - "[[redux-toolkit-async-thunks]]"
---

# React Router v6 Architecture: Layouts, Outlets & Nested Routing

> **Active Recall Self-Test:**
> 1. How does React Router v6's `<Outlet />` component work internally to enable shared persistent layouts across sub-routes?
> 2. What is the fundamental difference between declarative `<Link />` / `<Navigate />` components and imperative `useNavigate()` hook calls?
> 3. Why did React Router v6 replace v5's regex-based route matching with a relative path ranking algorithm?
> 4. How does the HTML5 History API (`pushState`, `replaceState`, `popstate`) enable client-side navigation without triggering full page reloads?

---

## 1. The Core Problem
### The Flaws of Repetitive Boilerplate & Component Re-Mounting

#### The Naive v5 Pattern: Duplicating Layout Wrappers
In older or naive routing implementations, developers repeat navigation headers, sidebars, and footers across every single page component:
```jsx
// DashboardPage.jsx
export const DashboardPage = () => (
  <AdminLayout>
    <Sidebar />
    <DashboardContent />
  </AdminLayout>
);

// SettingsPage.jsx
export const SettingsPage = () => (
  <AdminLayout>
    <Sidebar />
    <SettingsContent />
  </AdminLayout>
);
```
- **Unnecessary Re-Mounts:** When navigating from `/admin/dashboard` to `/admin/settings`, the entire `<AdminLayout>` and `<Sidebar>` unmount and re-mount from scratch.
- **State Wiping:** Any local state inside the sidebar (e.g., collapsed accordions, scroll position, search queries) is instantly destroyed on every route change.
- **Visual Glitches:** Users experience flickers as common layout elements tear down and rebuild.

#### The Architectural Solution: Nested Layout Routing with `<Outlet />`
React Router v6 introduces first-class parent-child route nesting. Parent routes render a persistent shell containing common layout elements, while child views dynamically render inside a designated `<Outlet />` slot without triggering parent re-mounts.

---

## 2. The Mental Model

### Layout Hierarchy & The `<Outlet />` Slot
```
Browser URL: "/admin/products"

[ BrowserRouter (Listens to HTML5 History popstate) ]
       │
       ▼
[ <Routes> (Path Ranking Engine) ]
       │ Matches parent path: "/admin"
       ▼
[ <AdminLayout /> (Persistent Shell) ]
 ┌────────────────────────────────────────────────────────┐
 │ Header: Global Admin Nav                               │
 ├────────────────┬───────────────────────────────────────┤
 │ Sidebar        │ <Outlet /> Slot                       │
 │ ├── Dashboard  │                                       │
 │ ├── Products ──┼────────► Dynamically mounts:          │
 │ └── Orders     │          <AdminProductsPage />        │
 │                │          (Header & Sidebar stay warm; │
 │                │           never re-mount!)            │
 └────────────────┴───────────────────────────────────────┘
```

### Path Resolution Mechanics
In React Router v6, paths are relative:
- Parent: `<Route path="/shop" element={<ShopLayout />}>`
- Child A: `<Route path="home" element={<ShopHome />} />` $\implies$ Resolves to `/shop/home`
- Child B: `<Route path="products/:id" element={<ProductDetail />} />` $\implies$ Resolves to `/shop/products/123`
- Index Route: `<Route index element={<ShopHome />} />` $\implies$ Matches `/shop` directly when no child path is specified.

---

## 3. Production Code Breakdown

### A. Root Router Definition with Layouts (`src/App.jsx`)
```jsx
import React from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import AuthLayout from './layouts/AuthLayout';
import AdminLayout from './layouts/AdminLayout';
import ShoppingLayout from './layouts/ShoppingLayout';

import LoginPage from './pages/auth/LoginPage';
import RegisterPage from './pages/auth/RegisterPage';
import AdminDashboard from './pages/admin/AdminDashboard';
import AdminProducts from './pages/admin/AdminProducts';
import ShopHome from './pages/shop/ShopHome';
import ProductDetails from './pages/shop/ProductDetails';
import NotFoundPage from './pages/common/NotFoundPage';

export default function App() {
  return (
    <Routes>
      {/* Root redirect */}
      <Route path="/" element={<Navigate to="/shop" replace />} />

      {/* 1. Auth Layout (Centered Card Shell) */}
      <Route path="/auth" element={<AuthLayout />}>
        <Route path="login" element={<LoginPage />} />
        <Route path="register" element={<RegisterPage />} />
      </Route>

      {/* 2. Admin Layout (Sidebar + Header + Protected Content) */}
      <Route path="/admin" element={<AdminLayout />}>
        <Route index element={<AdminDashboard />} />
        <Route path="dashboard" element={<AdminDashboard />} />
        <Route path="products" element={<AdminProducts />} />
      </Route>

      {/* 3. Shopping Layout (Storefront Header + Catalog + Footer) */}
      <Route path="/shop" element={<ShoppingLayout />}>
        <Route index element={<ShopHome />} />
        <Route path="home" element={<ShopHome />} />
        <Route path="product/:productId" element={<ProductDetails />} />
      </Route>

      {/* Catch-all 404 Route */}
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
}
```

### B. Persistent Parent Layout (`src/layouts/AdminLayout.jsx`)
```jsx
import React from 'react';
import { Outlet, NavLink } from 'react-router-dom';
import AdminSidebar from '../components/admin/AdminSidebar';
import AdminHeader from '../components/admin/AdminHeader';

export default function AdminLayout() {
  return (
    <div className="flex h-screen bg-slate-900 text-white">
      {/* Persistent Sidebar */}
      <AdminSidebar />

      <div className="flex flex-1 flex-col overflow-hidden">
        {/* Persistent Header */}
        <AdminHeader />

        {/* Dynamic Page Content Renders Inside Outlet */}
        <main className="flex-1 overflow-y-auto p-6 bg-slate-950">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

### C. Dynamic Parameter Extraction (`src/pages/shop/ProductDetails.jsx`)
```jsx
import React from 'react';
import { useParams, useNavigate, useLocation } from 'react-router-dom';

export default function ProductDetails() {
  const { productId } = useParams(); // Extracts :productId from URL
  const navigate = useNavigate();
  const location = useLocation();

  const handleBack = () => {
    // Navigate backwards in browser history stack safely
    if (window.history.state && window.history.state.idx > 0) {
      navigate(-1);
    } else {
      navigate('/shop/home', { replace: true });
    }
  };

  return (
    <div>
      <h2>Inspecting Product ID: {productId}</h2>
      <button onClick={handleBack}>Go Back</button>
    </div>
  );
}
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Leading Slash `/` in Nested Routes
- **The Trap:** Defining child paths with a leading forward slash:
  ```jsx
  <Route path="/admin" element={<AdminLayout />}>
    <Route path="/products" element={<AdminProducts />} /> {/* ❌ Bug! */}
  </Route>
  ```
- **The Consequence:** A leading slash treats the path as root-relative. The route matches `/products`, NOT `/admin/products`, disconnecting it from the parent hierarchy.
- **The Fix:** Child routes must always use relative paths without a leading slash: `path="products"`.

### ⚠️ Gotcha 2: Missing `replace: true` on Redirects
- **The Trap:** Using `<Navigate to="/login" />` or `navigate('/login')` without `{ replace: true }`.
- **The Bug:** The redirect pushes a new entry onto the browser's history stack. If an unauthenticated user hits the browser's "Back" button, it returns to the protected page, which immediately triggers the redirect to `/login` again, trapping the user in an inescapable infinite navigation loop.
- **The Fix:** Always specify `replace: true` for programmatic redirects and route guards.

### ⚠️ Gotcha 3: The Active Link State (`NavLink` vs `Link`)
- **The Trap:** Using manual `location.pathname === item.path` comparisons to highlight active sidebar menu items.
- **The Fix:** Use React Router's built-in `<NavLink>` component, which passes an `isActive` boolean to its `className` callback:
  ```jsx
  <NavLink
    to="/admin/products"
    className={({ isActive }) =>
      `px-4 py-2 rounded ${isActive ? 'bg-blue-600 text-white' : 'text-slate-400'}`
    }
  >
    Products
  </NavLink>
  ```
