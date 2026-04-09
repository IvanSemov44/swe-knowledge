# Zustand Deep Dive

## What it is
Zustand is a minimal, unopinionated state management library for React. No boilerplate, no providers, no actions/reducers — just a store as a hook.

Used by many teams as a lighter alternative to Redux for UI state.

---

## Basic Store

```ts
// store/useCartStore.ts
import { create } from 'zustand';

interface CartItem {
  productId: string;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  isOpen: boolean;
  addItem: (item: CartItem) => void;
  removeItem: (productId: string) => void;
  toggleCart: () => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  isOpen: false,

  addItem: (item) =>
    set((state) => {
      const existing = state.items.find(i => i.productId === item.productId);
      if (existing) {
        return {
          items: state.items.map(i =>
            i.productId === item.productId
              ? { ...i, quantity: i.quantity + item.quantity }
              : i
          ),
        };
      }
      return { items: [...state.items, item] };
    }),

  removeItem: (productId) =>
    set((state) => ({
      items: state.items.filter(i => i.productId !== productId),
    })),

  toggleCart: () => set((state) => ({ isOpen: !state.isOpen })),

  clearCart: () => set({ items: [] }),
}));
```

---

## Usage in Components

No Provider needed — just call the hook:

```tsx
// Selecting specific values (only re-renders when those values change)
const items = useCartStore(state => state.items);
const addItem = useCartStore(state => state.addItem);

// Multiple values with shallow comparison
import { useShallow } from 'zustand/react/shallow';

const { items, isOpen } = useCartStore(
  useShallow(state => ({ items: state.items, isOpen: state.isOpen }))
);
```

**Important:** Selecting the whole store object (`useCartStore()`) causes a re-render on ANY state change. Always select only what you need.

---

## Derived State

Compute values from the store without storing them:

```ts
// In component
const totalItems = useCartStore(state =>
  state.items.reduce((sum, i) => sum + i.quantity, 0)
);
```

Or as a selector function defined alongside the store:

```ts
// store/useCartStore.ts
export const selectTotalItems = (state: CartStore) =>
  state.items.reduce((sum, i) => sum + i.quantity, 0);

// In component
const totalItems = useCartStore(selectTotalItems);
```

---

## Async Actions

Actions can be async — no thunk setup needed:

```ts
interface UserStore {
  user: User | null;
  isLoading: boolean;
  fetchUser: (id: string) => Promise<void>;
}

export const useUserStore = create<UserStore>((set) => ({
  user: null,
  isLoading: false,

  fetchUser: async (id) => {
    set({ isLoading: true });
    try {
      const user = await api.getUser(id);
      set({ user, isLoading: false });
    } catch {
      set({ isLoading: false });
    }
  },
}));
```

> **Note:** For server state (fetching, caching, revalidation), prefer RTK Query or TanStack Query over storing it in Zustand.

---

## Middleware: persist

Persist store state to localStorage:

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useCartStore = create<CartStore>()(
  persist(
    (set) => ({
      items: [],
      addItem: (item) => set(state => ({ items: [...state.items, item] })),
    }),
    { name: 'cart-storage' } // localStorage key
  )
);
```

---

## Middleware: devtools

Connect to Redux DevTools:

```ts
import { devtools } from 'zustand/middleware';

export const useCartStore = create<CartStore>()(
  devtools(
    (set) => ({ ... }),
    { name: 'CartStore' }
  )
);
```

---

## Zustand vs Redux Toolkit

| | Zustand | Redux Toolkit |
|---|---|---|
| **Boilerplate** | Minimal | More structured |
| **Setup** | `create()`, done | `configureStore`, slices, typed hooks |
| **DevTools** | Via middleware | Built-in |
| **Learning curve** | Very low | Medium |
| **Team conventions** | You define them | Enforced by RTK patterns |
| **Best for** | Smaller apps, UI state | Large apps, complex state, teams |
| **Server state** | Use with TanStack Query | RTK Query built in |

**When to choose Zustand:**
- Smaller project or prototype
- You want minimal setup
- Team already knows it

**When to choose Redux Toolkit:**
- Large team with strict conventions
- Complex state interactions across many features
- You're already using RTK Query (natural fit)

---

## Common Interview Questions

1. What is Zustand and how does it differ from Redux?
2. Why do you need to select specific slices of state instead of the whole store?
3. How would you handle async operations in Zustand?
4. What is `useShallow` and when do you need it?
5. Can Zustand replace RTK Query? Why or why not?

---

## Common Mistakes

- Selecting the whole store (`useCartStore()`) instead of specific fields → unnecessary re-renders
- Storing server state in Zustand (use RTK Query / TanStack Query)
- Not using `useShallow` when selecting multiple fields at once
- Putting everything in one giant store (split by domain)

---

## How It Connects

- Zustand is an alternative to Redux slices — NOT an alternative to RTK Query
- `useShallow` is the Zustand equivalent of `createSelector` in Redux (prevents re-renders on reference equality)
- `persist` middleware is similar to `redux-persist`
- Zustand's `set` function is conceptually like dispatching a Redux action that directly patches state

---

## My Confidence Level
- `[ ]` create() — basic store with state + actions
- `[ ]` Selecting specific state (avoiding full store subscription)
- `[ ]` useShallow for multiple fields
- `[ ]` Async actions
- `[ ]` persist middleware
- `[ ]` Zustand vs Redux Toolkit — when to choose which

## My Notes
<!-- Personal notes -->
