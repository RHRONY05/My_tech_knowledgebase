---
tags:
  - frontend
  - react
  - redux
  - state-management
  - architecture
last_reviewed: 2026-10-02
related_notes:
  - "[[frontend/redux/redux-toolkit-async-thunks]]"
  - "[[frontend/redux/javascript-reduce-to-redux-reducers]]"
  - "[[frontend/react/react-auth-guards-and-routing]]"
---

# Redux Toolkit (RTK) Core Architecture & Unidirectional Dataflow

> [!CHECKLIST] Active Recall Self-Test
> - [ ] What is the exact sequence of events in the Redux unidirectional dataflow from user click to component re-render?
> - [ ] How does Immer.js allow developers to write "mutating" logic (`state.value += 1`) while maintaining 100% immutable state snapshots under the hood?
> - [ ] Why should you organize Redux code by **feature** (`features/counter/counterSlice.js`) rather than by technical type (`actions/`, `reducers/`)?
> - [ ] What catastrophic re-render loop occurs if you return a newly instantiated object or array directly inside an unmemoized `useSelector`?
> - [ ] Why is returning a new state value AND mutating the draft in the same `createSlice` reducer an illegal operation in Immer?

---

## 1. The Core Problem: The Prop-Drilling & Mutation Trap

As React applications scale beyond trivial component trees, local state management (`useState`) breaks down:

1. **Prop Drilling Hell:** Sharing state between sibling components sitting five levels deep in the DOM tree requires drilling props through dozens of intermediary components that do not care about the data.
2. **State Desynchronization:** Duplicating state across multiple components leads to stale closures and split-brain UI states where one header component shows the user as logged out while a sidebar displays an active session.
3. **Accidental Object Mutation:** In vanilla JavaScript, updating nested state requires painstaking spread syntax (`return { ...state, user: { ...state.user, profile: { ...state.user.profile, name: 'Alice' } } }`). Forgetting a single spread level mutates the previous state object directly, breaking React change detection and time-travel debugging.
4. **Legacy Redux Boilerplate Bloat:** Historically, building a single action in vanilla Redux required writing 4 separate files: action type constants (`COUNTER_INCREMENT`), action creator functions, reducer switch statements, and immutable copy logic. Redux Toolkit (RTK) was created to eliminate this boilerplate entirely.

---

## 2. The Mental Model: The Unidirectional Execution Cycle

State in Redux travels strictly in a single, closed loop. No component ever directly modifies the store; it dispatches an action describing an intent, and the store computes the new state via pure reducer functions.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   REDUX UNIDIRECTIONAL DATA FLOW                       │
└────────────────────────────────────────────────────────────────────────┘

    1. USER INTERACTION
          │
          │ User clicks "Add to Cart" button
          ▼
    2. EVENT HANDLER DISPATCHES ACTION
          │
          │ dispatch(addToCart({ id: "prod_1", qty: 1 }))
          │ Action Object: { type: "cart/addToCart", payload: { ... } }
          ▼
    3. REDUX STORE BROADCASTS ACTION
          │
          │ Store passes (currentState, action) to ALL registered slice reducers
          ▼
    4. REDUCER EVALUATES ACTION TYPE
          │
          ├─ userSlice: Type mismatch? ──▶ Ignore action (return unchanged state)
          │
          └─ cartSlice: Type matches! ───▶ EXECUTE REDUCER
                                                 │
                                                 ▼
                                           Immer Proxy Interceptor
                                           • Tracks draft mutations
                                           • Produces fresh immutable state snapshot
                                                 │
                                                 ▼
    5. STORE COMMITS NEW STATE SNAPSHOT
          │
          │ Central state updated: { cart: { items: [...], total: 45 } }
          ▼
    6. SUBSCRIBER NOTIFICATION (useSelector)
          │
          │ All `useSelector` hooks run their selector functions
          │ Performs strict reference equality check (`prevSelected === nextSelected`)
          │
          ├─ If selected value is IDENTICAL (===) ──▶ Skip component re-render
          │
          └─ If selected value CHANGED (!==) ───────▶ Trigger component re-render
                                                            │
                                                            ▼
    7. UI RE-RENDERS WITH FRESH DATA
          │
          └─▶ User sees updated cart badge: "1 item"
```

---

## 3. Production Code Breakdown

### A. Feature-First Folder Architecture

```
src/
├── app/
│   ├── store.js             # Central store configuration & middleware
│   └── hooks.js             # Typed hooks (useAppDispatch, useAppSelector)
├── features/
│   ├── cart/
│   │   ├── Cart.jsx         # UI Component
│   │   ├── cartSlice.js     # State, reducers, selectors, thunks in ONE file
│   │   └── cartAPI.js       # External network endpoints
│   └── auth/
│       ├── Auth.jsx
│       └── authSlice.js
└── index.jsx                # Root React Provider binding
```

### B. Slice Definition with Immer Mutability (`features/cart/cartSlice.js`)

```javascript
// src/features/cart/cartSlice.js
import { createSlice, createSelector } from "@reduxjs/toolkit";

const initialState = {
  items: [], // Array of { id, title, price, quantity }
  discountCode: null,
  status: "idle", // 'idle' | 'loading' | 'failed'
};

export const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    // Immer allows direct mutation syntax!
    addToCart: (state, action) => {
      const { id, title, price } = action.payload;
      const existingItem = state.items.find((item) => item.id === id);

      if (existingItem) {
        existingItem.quantity += 1; // Direct mutation safe under Immer proxy
      } else {
        state.items.push({ id, title, price, quantity: 1 });
      }
    },

    removeFromCart: (state, action) => {
      const idToRemove = action.payload;
      state.items = state.items.filter((item) => item.id !== idToRemove);
    },

    clearCart: (state) => {
      // Replacing state entirely requires assigning properties or returning new state
      state.items = [];
      state.discountCode = null;
    },
  },
});

// Auto-generated Action Creators
export const { addToCart, removeFromCart, clearCart } = cartSlice.actions;

// Base Selectors
export const selectCartItems = (state) => state.cart.items;
export const selectDiscountCode = (state) => state.cart.discountCode;

// Memoized Reselect Selector: Recalculates ONLY when `cart.items` array reference changes
export const selectCartTotals = createSelector([selectCartItems], (items) => {
  const itemCount = items.reduce((total, item) => total + item.quantity, 0);
  const subtotal = items.reduce((total, item) => total + item.price * item.quantity, 0);

  return { itemCount, subtotal };
});

export default cartSlice.reducer;
```

### C. Store Configuration (`app/store.js`)

```javascript
// src/app/store.js
import { configureStore } from "@reduxjs/toolkit";
import cartReducer from "../features/cart/cartSlice.js";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
  },
  // Redux Toolkit automatically enables:
  // 1. Redux DevTools Extension integration
  // 2. redux-thunk middleware
  // 3. Immutability & Serializability runtime invariant checks in development
  devTools: process.env.NODE_ENV !== "production",
});
```

### D. Component Consumption with Optimized Subscriptions

```jsx
// src/features/cart/CartBadge.jsx
import React from "react";
import { useSelector, useDispatch } from "react-redux";
import { selectCartTotals, clearCart } from "./cartSlice.js";

export const CartBadge = () => {
  const dispatch = useDispatch();

  // Subscribes ONLY to the memoized derived totals, avoiding re-renders when other state changes
  const { itemCount, subtotal } = useSelector(selectCartTotals);

  return (
    <div className="cart-badge">
      <span>Items: {itemCount}</span>
      <span>Subtotal: ${subtotal.toFixed(2)}</span>
      <button onClick={() => dispatch(clearCart())}>Reset</button>
    </div>
  );
};
```

---

## 4. Production Gotchas & Best Practices

### 1. The "Mutate AND Return" Immer Illegal Operation
Immer allows you to **either** mutate the draft state **or** return a brand new state, but never both in the same reducer:
```javascript
// ❌ CRASH: Uncaught Error: [Immer] An invariant failed, return and mutate at the same time
clearCart: (state) => {
  state.items = [];
  return { items: [], discountCode: null }; // ILLEGAL
}

// ✅ CORRECT: Either mutate directly
clearCart: (state) => {
  state.items = [];
  state.discountCode = null;
}

// ✅ OR return a replacement object without touching draft
clearCart: () => initialState;
```

### 2. The Unmemoized Selector Re-render Trap
If your selector returns a newly instantiated object or array on every execution, `useSelector` will think state changed every single time an action is dispatched, triggering infinite renders:
```javascript
// ❌ SEVERE PERFORMANCE BUG: Creates a brand-new object on EVERY selector run
const { itemCount } = useSelector((state) => ({
  itemCount: state.cart.items.length,
}));

// ✅ CORRECT: Select primitive values directly (primitives compare by value)
const itemCount = useSelector((state) => state.cart.items.length);

// ✅ OR use `createSelector` from RTK to memoize complex derived objects
const totals = useSelector(selectCartTotals);
```

### 3. Asynchronous Logic Belongs in Thunks, NOT Reducers
Reducers must remain **100% pure functions**. Calling `fetch()`, writing to `localStorage`, or generating `Math.random()` / `Date.now()` inside a reducer breaks time-travel debugging and determinism.
- Keep all I/O, timers, and async calls inside **Async Thunks** (`createAsyncThunk`), and handle their outcomes in `extraReducers`.

### 4. Non-Serializable Data in State or Actions
Never put non-serializable objects (Promises, class instances, Symbols, DOM elements, or functions) into the Redux store or action payloads. Redux Toolkit will throw development warnings because non-serializable data breaks DevTools state hydration and undo/redo capabilities.
