# React Performance

## What it is
Understanding when and why React re-renders, and the tools to prevent unnecessary renders: `React.memo`, `useMemo`, `useCallback`, lazy loading, and the React Profiler.

## Why it matters
Performance questions come up in every mid-level React interview. More importantly, misusing `useMemo` and `useCallback` is one of the most common mistakes — adding them everywhere without understanding when they help.

---

## When React Re-Renders

A component re-renders when:
1. Its **state** changes (`useState`, `useReducer`)
2. Its **parent re-renders** (by default, all children re-render too)
3. Its **context value** changes
4. Its **props** change (reference equality for objects/functions)

```typescript
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
      <Child /> {/* re-renders every time Parent re-renders, even if no props changed */}
    </div>
  );
}

function Child() {
  console.log('Child rendered');
  return <div>I am a child</div>;
}
```

---

## React.memo

Wraps a component and skips re-rendering if props haven't changed (shallow comparison).

```typescript
// Without memo — re-renders every time parent re-renders
function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
}

// With memo — only re-renders if product reference changes
const ProductCard = React.memo(function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
});

// Custom comparison — for when you need deep equality on part of props
const ProductCard = React.memo(
  function ProductCard({ product, onAddToCart }: ProductCardProps) {
    return <div>{product.name}</div>;
  },
  (prev, next) => prev.product.id === next.product.id // custom comparator
);
```

**When `React.memo` helps:**
- Component renders often but props don't change
- Component is expensive to render (large lists, complex UI)

**When it doesn't help:**
- Props are objects/functions created inline in the parent — new reference every render = always re-renders anyway
- Component is cheap to render — the comparison overhead may cost more than the render

---

## useMemo

Memoizes an **expensive computed value** — only recomputes when dependencies change.

```typescript
// WITHOUT useMemo — totalPrice recomputed on every render
function CartSummary({ items }: { items: CartItem[] }) {
  const totalPrice = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  return <div>Total: ${totalPrice}</div>;
}

// WITH useMemo — only recomputed when items changes
function CartSummary({ items }: { items: CartItem[] }) {
  const totalPrice = useMemo(
    () => items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );
  return <div>Total: ${totalPrice}</div>;
}
```

**When `useMemo` helps:**
- Expensive calculations (filtering/sorting large lists, complex math)
- Creating stable object references to pass as props to `React.memo` children

**When it doesn't help (don't overuse):**
```typescript
// UNNECESSARY — primitive operations don't benefit from memoization
const doubled = useMemo(() => count * 2, [count]); // just write: count * 2

// UNNECESSARY — useMemo itself has overhead; not worth it for cheap ops
const label = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// just write: `${firstName} ${lastName}`
```

---

## useCallback

Memoizes a **function reference** — returns the same function instance across renders unless dependencies change.

```typescript
// WITHOUT useCallback — new function reference on every render
function ProductList({ products }: { products: Product[] }) {
  const handleAddToCart = (productId: string) => {
    dispatch(addItem(productId));
  };

  return products.map(p =>
    <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
    // ProductCard with React.memo still re-renders every time — new function reference!
  );
}

// WITH useCallback — stable function reference
function ProductList({ products }: { products: Product[] }) {
  const handleAddToCart = useCallback((productId: string) => {
    dispatch(addItem(productId));
  }, []); // empty deps — dispatch is stable from Redux

  return products.map(p =>
    <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
    // Now React.memo works — same function reference
  );
}
```

**`useCallback` only makes sense when the function is passed to:**
- A `React.memo` wrapped child
- A `useEffect` dependency array
- Another `useMemo`/`useCallback` dependency

**The trio works together:**
```
React.memo → skips re-render if props are same references
useCallback → keeps function props at stable references
useMemo → keeps object/array props at stable references
```

---

## The Dependency Array — Getting It Right

```typescript
// stale closure — reads outdated count value
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // always 0! count is captured from first render
  }, 1000);
  return () => clearInterval(id);
}, []); // missing count in deps

// correct — include count, but interval re-creates every render
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // always latest value
  }, 1000);
  return () => clearInterval(id);
}, [count]);

// better — use functional update to avoid stale closure
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1); // doesn't read count — no stale closure
  }, 1000);
  return () => clearInterval(id);
}, []); // empty deps is now correct
```

---

## Key Prop

React uses `key` to identify list items. Wrong keys cause subtle bugs and poor performance.

```typescript
// WRONG — index as key: causes state bugs when list reorders
{products.map((p, index) => <ProductCard key={index} product={p} />)}

// CORRECT — stable unique ID
{products.map(p => <ProductCard key={p.id} product={p} />)}

// Key forces remount — use to reset component state
<UserProfile key={userId} userId={userId} />
// Changing userId causes UserProfile to fully unmount and remount — state reset
```

---

## Lazy Loading — Code Splitting

Split your bundle so users only download code for the current page.

```typescript
// Without lazy — entire app JS loaded upfront
import { AdminDashboard } from './pages/AdminDashboard';

// With lazy — AdminDashboard JS only loaded when first rendered
const AdminDashboard = React.lazy(() => import('./pages/AdminDashboard'));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/admin" element={<AdminDashboard />} />
      </Routes>
    </Suspense>
  );
}
```

```typescript
// Lazy load heavy components (rich text editors, charts, maps)
const RichTextEditor = React.lazy(() => import('./RichTextEditor'));

function ProductDescriptionEditor({ onChange }: { onChange: (value: string) => void }) {
  return (
    <Suspense fallback={<TextareaSkeleton />}>
      <RichTextEditor onChange={onChange} />
    </Suspense>
  );
}
```

---

## React Profiler

Built into React DevTools. Shows which components rendered, how long they took, and why.

```
How to use:
1. Open React DevTools → Profiler tab
2. Click Record
3. Interact with your app
4. Stop recording
5. Click any bar to see: component name, render time, why it rendered
   (changed props, state, hooks, parent)
```

**In code — programmatic profiling:**
```typescript
import { Profiler } from 'react';

function onRender(
  id: string,          // component name
  phase: 'mount' | 'update',
  actualDuration: number,  // render time in ms
  baseDuration: number,    // estimated time without memoization
) {
  if (actualDuration > 16) // flag renders taking longer than one frame
    console.warn(`Slow render: ${id} took ${actualDuration}ms`);
}

<Profiler id="ProductList" onRender={onRender}>
  <ProductList products={products} />
</Profiler>
```

---

## Quick Decision Guide

```
Is the component re-rendering unnecessarily?
  → Wrap with React.memo

Are you passing a function as prop to a memoized child?
  → Wrap with useCallback

Are you passing an object/array as prop to a memoized child, or doing expensive computation?
  → Wrap with useMemo

Is the initial JS bundle too large / page load slow?
  → Use React.lazy + Suspense for route-level code splitting

Is a specific component visibly slow?
  → Profile with React DevTools Profiler first, then optimize
```

**Rule: Measure before you optimize. Don't add `useMemo`/`useCallback` everywhere "just in case" — it adds cognitive overhead and its own (small) performance cost.**

---

## Common Interview Questions

1. When does React re-render a component?
2. What does `React.memo` do? When does it NOT help?
3. What is the difference between `useMemo` and `useCallback`?
4. Why does `React.memo` fail to prevent re-renders when the parent passes an inline function?
5. What is a stale closure in `useEffect`?
6. Why should you avoid using array index as a `key`?
7. What is `React.lazy` and what problem does it solve?

---

## Common Mistakes

- Adding `useMemo`/`useCallback` everywhere without measuring — premature optimization
- Using `React.memo` but passing new object/function references every render — memo does nothing
- Missing dependencies in `useEffect` — stale closures, bugs that only appear sometimes
- Using array index as `key` in a reorderable list — React reuses wrong DOM nodes
- Not lazy-loading heavy third-party libraries (chart libraries, rich text editors)
- Profiling in development mode — always profile production builds (dev mode is intentionally slower)

---

## How It Connects

- `useCallback` + `React.memo` is the pair used to prevent RTK Query from triggering unnecessary re-fetches via unstable callbacks
- Code splitting with `React.lazy` pairs with Next.js's automatic code splitting per page
- React Profiler output tells you which components to target — the `baseDuration` vs `actualDuration` gap shows how much memoization saved
- Redux `useSelector` already uses shallow equality by default; `createSelector` (reselect) is the `useMemo` equivalent for derived Redux state

---

## My Confidence Level
- `[ ]` When React re-renders — the four triggers
- `[b]` React.memo — what it does, when it fails
- `[b]` useMemo — expensive values, when NOT to use
- `[b]` useCallback — stable function references, the trio
- `[ ]` Dependency array — stale closures, functional updates
- `[b]` Key prop — stable IDs, remount trick
- `[ ]` React.lazy + Suspense — code splitting
- `[ ]` React Profiler — reading results, finding bottlenecks

## My Notes
<!-- Personal notes -->
