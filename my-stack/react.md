# React Deep Dive

## What it is
A JavaScript library for building user interfaces through composable components. React manages a virtual DOM and efficiently updates the real DOM when state changes.

---

## Components & Props

```tsx
// Functional component with typed props
interface ProductCardProps {
  product: ProductDto;
  onAddToCart: (productId: string) => void;
}

const ProductCard = ({ product, onAddToCart }: ProductCardProps) => (
  <div className="product-card">
    <h3>{product.name}</h3>
    <p>${product.price.toFixed(2)}</p>
    <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
  </div>
);
```

---

## useState

```tsx
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// Functional update (when new state depends on previous)
setCount(prev => prev + 1); // CORRECT
setCount(count + 1);        // WRONG if called multiple times in a row

// Object state — always spread, never mutate
setUser(prev => ({ ...prev!, name: 'Ivan' }));
```

---

## useEffect

```tsx
// Runs after every render (no dependency array)
useEffect(() => { document.title = `${count} items`; });

// Runs once on mount (empty dependency array)
useEffect(() => {
  fetchData();
  return () => cleanup(); // cleanup on unmount
}, []);

// Runs when dependencies change
useEffect(() => {
  if (userId) fetchUserData(userId);
}, [userId]);
```

**Rules:**
- Never call Hooks conditionally
- Dependencies must include everything used inside the effect that comes from React scope

---

## useCallback & useMemo

```tsx
// useMemo: memoize expensive computation
const sortedProducts = useMemo(
  () => [...products].sort((a, b) => a.price - b.price),
  [products] // only re-sort when products array changes
);

// useCallback: memoize a function reference (prevent child re-renders)
const handleAddToCart = useCallback(
  (productId: string) => dispatch(addToCart(productId)),
  [dispatch] // stable reference
);
```

**When to use:**
- `useMemo` for expensive computations (sorting large lists, heavy transformations)
- `useCallback` when passing callbacks to children wrapped in `React.memo`
- Don't over-optimize — premature memoization adds complexity without benefit

---

## useRef

```tsx
// DOM reference
const inputRef = useRef<HTMLInputElement>(null);
const focusInput = () => inputRef.current?.focus();
return <input ref={inputRef} />;

// Mutable value that doesn't trigger re-render
const timerRef = useRef<NodeJS.Timeout | null>(null);
timerRef.current = setTimeout(() => {}, 1000);
```

---

## Custom Hooks

Extract stateful logic into reusable functions:

```tsx
// hooks/useDebounce.ts
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
const debouncedSearch = useDebounce(searchTerm, 300);
```

---

## React.memo

Prevent re-rendering when props haven't changed:

```tsx
const ProductCard = React.memo(({ product, onAddToCart }: ProductCardProps) => {
  // Only re-renders when product or onAddToCart reference changes
  return <div>...</div>;
});
```

**Pair with `useCallback`:** If you pass an inline `() => {}` function to a memoized component, it creates a new reference every render, defeating `memo`.

---

## Reconciliation & Virtual DOM

React maintains a virtual DOM (in-memory JS object tree). On state change:
1. React re-renders the component (creates new virtual DOM)
2. React diffs old vs new virtual DOM (reconciliation)
3. Only the changed DOM nodes are updated (patching)

**Key optimization:** `key` prop on list items. Without it, React can't efficiently diff lists and may re-render all items on any change.

```tsx
// WRONG: using index as key (items can reorder)
{products.map((p, i) => <ProductCard key={i} product={p} />)}

// CORRECT: use stable unique ID
{products.map(p => <ProductCard key={p.id} product={p} />)}
```

---

## Context API

Share state without prop drilling. Use for global UI state (theme, locale, auth).

```tsx
interface AuthContextValue {
  user: User | null;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<User | null>(null);
  return (
    <AuthContext.Provider value={{ user, logout: () => setUser(null) }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
};
```

**Don't use Context for server state** — that's RTK Query's job.

---

## RTK Query (see rtk-query.md)

All API calls go through RTK Query endpoints, not in `useEffect` directly.

```tsx
// WRONG: fetch in component
useEffect(() => {
  fetch('/api/products').then(r => r.json()).then(setProducts);
}, []);

// CORRECT: RTK Query
const { data: products, isLoading } = useGetProductsQuery({ page: 1 });
```

---

## React Router

```tsx
// App.tsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<HomePage />} />
    <Route path="/products/:id" element={<ProductPage />} />
    <Route path="/cart" element={<ProtectedRoute><CartPage /></ProtectedRoute>} />
    <Route path="*" element={<NotFoundPage />} />
  </Routes>
</BrowserRouter>

// In component
const { id } = useParams<{ id: string }>();
const navigate = useNavigate();
navigate('/products');
```

---

## Forms

```tsx
// Controlled input (state drives input)
const [name, setName] = useState('');
<input value={name} onChange={e => setName(e.target.value)} />

// React Hook Form (recommended for complex forms)
const { register, handleSubmit, formState: { errors } } = useForm<FormData>();

<form onSubmit={handleSubmit(onSubmit)}>
  <input {...register('email', { required: true, pattern: /^\S+@\S+$/i })} />
  {errors.email && <span>Invalid email</span>}
</form>
```

---

## Performance Optimization

```tsx
// Lazy loading routes
const ProductPage = lazy(() => import('./pages/ProductPage'));
<Suspense fallback={<Spinner />}>
  <ProductPage />
</Suspense>

// Virtualize long lists (only render visible items)
import { FixedSizeList } from 'react-window';

// Split large bundles
// Vite code splitting: dynamic imports
```

---

## Common Interview Questions

1. What is reconciliation and how do keys help?
2. What is the difference between `useMemo` and `useCallback`?
3. When would you use `useRef` vs `useState`?
4. What is Context API and when should you not use it?
5. What causes unnecessary re-renders and how do you prevent them?
6. What is the Virtual DOM?

---

## Common Mistakes

- Missing dependency array in `useEffect` → infinite loop
- Mutating state directly (`state.items.push(item)` → use spread)
- Using array index as `key` in dynamic lists
- Fetching data in `useEffect` directly instead of RTK Query
- Over-memoizing with `useMemo`/`useCallback` (adds cost too)
- Putting server state in Redux slices instead of RTK Query

---

## How It Connects

- Redux Toolkit manages UI/client state; RTK Query manages server/cache state
- React Router integrates with Redux for protected routes
- `useCallback` stability is essential when callbacks are passed to `React.memo` children
- Context API is fine for auth state; RTK Query for everything from the server
- Reconciliation understanding explains why key props matter for list performance

---

## My Confidence Level
- `[c]` Components, props, state
- `[c]` useEffect, useMemo, useCallback
- `[c]` Custom hooks
- `[b]` Context API
- `[b]` React Router
- `[b]` Forms (controlled vs uncontrolled)
- `[~]` Performance (memo, lazy, Suspense)

## My Notes
<!-- Personal notes -->
