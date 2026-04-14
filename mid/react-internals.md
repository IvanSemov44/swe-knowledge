# React Internals

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: b -->

---

## How Re-renders Work

When a parent re-renders, **all its children re-render by default** — regardless of whether their props changed. React re-runs the function top to bottom.

Re-render is triggered by:
- `setState` / `useState` setter called
- Context value changes
- Parent re-renders

Re-render does **not** mean DOM update — React diffs the Virtual DOM first.

---

## Virtual DOM & Reconciliation

React keeps a Virtual DOM (plain JS objects). On re-render:

1. React builds a new Virtual DOM tree
2. Diffs it against the previous one (reconciliation)
3. Only applies the **minimal set of real DOM changes**

This is why re-renders are usually cheap — most of the time nothing changes in the real DOM.

---

## Preventing Unnecessary Re-renders

### React.memo

Wraps a component. React skips re-rendering if props haven't changed (shallow comparison).

```tsx
const ProductCard = React.memo(({ product }: { product: Product }) => {
    return <div>{product.name}</div>;
});
```

**Shallow comparison** means: primitive values compared by value, objects/functions compared by reference.

### useCallback — stabilize function props

Without it, `React.memo` is useless for components that receive function props — every render creates a new function reference:

```tsx
// BAD — new reference every render, memo never skips
<ProductCard onAdd={() => dispatch(addToCart(id))} />

// GOOD — stable reference across renders
const handleAdd = useCallback(() => dispatch(addToCart(id)), [id]);
<ProductCard onAdd={handleAdd} />
```

### useMemo — stabilize computed values

```tsx
const sorted = useMemo(() => [...items].sort((a, b) => a.price - b.price), [items]);
```

### How they work together

- `React.memo` — wraps the **component**, skips render if props unchanged
- `useCallback` — stabilizes **function props** so memo's comparison actually works
- `useMemo` — stabilizes **computed values** passed as props

`React.memo` alone is often not enough. If you pass a function prop without `useCallback`, memo always sees a new reference and never skips.

---

## When NOT to Use React.memo

- Component renders are cheap (simple markup, no heavy computation)
- Props always change anyway — memo check adds overhead with no benefit
- Component almost always needs to re-render legitimately

**Rule:** measure first, optimize second. Don't add memo everywhere upfront.

---

## Keys in Lists

React uses `key` to identify list items during reconciliation. Wrong keys = unnecessary re-renders or broken state.

```tsx
// BAD — index as key causes re-renders and state bugs when list reorders
{items.map((item, i) => <ProductCard key={i} product={item} />)}

// GOOD — stable unique ID
{items.map(item => <ProductCard key={item.id} product={item} />)}
```

---

## Concurrent Features (React 18)

### Suspense

Lets a component "wait" for async data before rendering. Shows a fallback in the meantime.

```tsx
<Suspense fallback={<Spinner />}>
    <ProductList />  {/* can throw a Promise to suspend */}
</Suspense>
```

### useTransition

Marks a state update as non-urgent — React can interrupt it to handle higher-priority updates (e.g. user input).

```tsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
    setSearchQuery(value);  // non-urgent — won't block typing
});
```

### useDeferredValue

Defers re-rendering a value until the browser is idle. Similar to debounce but React-aware.

```tsx
const deferredQuery = useDeferredValue(searchQuery);
// deferredQuery lags behind searchQuery during fast typing
```

---

## Gaps to Fill Next

- [ ] React Fiber architecture (how concurrent mode works internally)
- [ ] Strict Mode double-render behavior
- [ ] Server Components (React 19)

---

## Connected To

- `mid/rtk-query.md` — RTK Query triggers re-renders via cache updates; understanding memo helps optimize list components
- `my-stack/react.md` — hooks reference
