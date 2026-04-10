# Redux Toolkit Deep Dive

## What it is
Redux Toolkit (RTK) is the official, opinionated toolset for Redux. It eliminates boilerplate (no more action type constants, switch reducers, spread hell) while keeping Redux's predictable state model.

---

## Core Concepts

### Store
```ts
// app/store.ts
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from '../features/cart/cartSlice';
import uiReducer from '../features/ui/uiSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    ui: uiReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

## Slice

A slice owns one piece of state. It collocates the reducer + action creators.

```ts
// features/cart/cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CartItem {
  productId: string;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  isOpen: boolean;
}

const initialState: CartState = { items: [], isOpen: false };

const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    addItem(state, action: PayloadAction<CartItem>) {
      const existing = state.items.find(i => i.productId === action.payload.productId);
      if (existing) {
        existing.quantity += action.payload.quantity; // Immer allows mutation here
      } else {
        state.items.push(action.payload);
      }
    },
    removeItem(state, action: PayloadAction<string>) {
      state.items = state.items.filter(i => i.productId !== action.payload);
    },
    toggleCart(state) {
      state.isOpen = !state.isOpen;
    },
    clearCart(state) {
      state.items = [];
    },
  },
});

export const { addItem, removeItem, toggleCart, clearCart } = cartSlice.actions;
export default cartSlice.reducer;
```

**Key point:** RTK uses Immer under the hood. You can write "mutating" code inside reducers — Immer produces a new immutable state. Do NOT return AND mutate.

---

## Typed Hooks

Always use typed versions to avoid casting everywhere:

```ts
// app/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

```tsx
// In component
const cartItems = useAppSelector(state => state.cart.items);
const dispatch = useAppDispatch();

dispatch(addItem({ productId: '123', quantity: 1 }));
```

---

## Selectors

Extract derived data from state. Keeps components clean and logic testable.

```ts
// Simple selector (inline)
const itemCount = useAppSelector(state =>
  state.cart.items.reduce((sum, item) => sum + item.quantity, 0)
);

// Memoized selector with createSelector (reselect)
import { createSelector } from '@reduxjs/toolkit';

const selectCartItems = (state: RootState) => state.cart.items;
const selectProducts = (state: RootState) => state.products.list;

export const selectCartTotal = createSelector(
  [selectCartItems, selectProducts],
  (items, products) =>
    items.reduce((sum, item) => {
      const product = products.find(p => p.id === item.productId);
      return sum + (product?.price ?? 0) * item.quantity;
    }, 0)
);
```

**When to use `createSelector`:** when the selector involves computation (filter, reduce, map). Avoids recomputing on every render.

---

## createAsyncThunk

For async side effects that need to update Redux state (e.g., submit a form, then update UI state).

```ts
// Only use this for things that need to live in Redux state.
// If it's just a server fetch → use RTK Query instead.

export const submitOrder = createAsyncThunk(
  'order/submit',
  async (payload: CreateOrderRequest, { rejectWithValue }) => {
    try {
      const response = await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify(payload),
      });
      if (!response.ok) return rejectWithValue('Order failed');
      return await response.json();
    } catch (err) {
      return rejectWithValue('Network error');
    }
  }
);

// Handle in slice with extraReducers
const orderSlice = createSlice({
  name: 'order',
  initialState: { status: 'idle' as 'idle' | 'loading' | 'succeeded' | 'failed' },
  reducers: {},
  extraReducers: builder => {
    builder
      .addCase(submitOrder.pending, state => { state.status = 'loading'; })
      .addCase(submitOrder.fulfilled, state => { state.status = 'succeeded'; })
      .addCase(submitOrder.rejected, state => { state.status = 'failed'; });
  },
});
```

---

## Redux DevTools

Works out of the box with `configureStore`. In browser DevTools:
- Time-travel debugging (step through past actions)
- Inspect state at any point
- Dispatch actions manually

---

## Redux vs RTK Query — What Goes Where

| | Redux Toolkit (slices) | RTK Query |
|---|---|---|
| **Purpose** | UI state, client-only state | Server state (fetching, caching, sync) |
| **Examples** | Cart open/closed, modal state, selected tab, theme | Products list, user profile, order history |
| **Source of truth** | Your app | The server |
| **Invalidation** | You decide when to clear | Cache tags drive it |

**Rule of thumb:** If it came from an API, it belongs in RTK Query. If it's pure UI or computed from user interaction, it belongs in a slice.

---

## Common Interview Questions

1. What problem does Redux solve and when would you NOT use it?
2. What is Immer and why does RTK use it?
3. What is the difference between `useSelector` and `createSelector`?
4. When would you use `createAsyncThunk` vs RTK Query?
5. What is a Redux action? What is a reducer?
6. How does Redux DevTools help with debugging?

---

## Common Mistakes

- Mutating state outside a reducer
- Returning AND mutating state in the same reducer case (Immer: do one or the other)
- Storing server data in Redux slices instead of RTK Query
- Not using typed hooks (leads to `any` types everywhere)
- Overusing Redux for simple local state (use `useState` instead)

---

## How It Connects

- RTK Query is built on top of Redux Toolkit — both live in the same store
- `createSelector` uses the Reselect library under the hood
- Immer is what makes the "mutation syntax" inside reducers safe
- Redux slices are the RTK equivalent of classic `combineReducers` + action creators

---

## My Confidence Level
- `[ ]` Slices (actions, reducers, Immer)
- `[ ]` Typed hooks (useAppSelector, useAppDispatch)
- `[ ]` createSelector / memoized selectors
- `[ ]` createAsyncThunk + extraReducers
- `[ ]` Redux DevTools
- `[ ]` Redux vs RTK Query — what goes where

## My Notes
<!-- Personal notes -->
