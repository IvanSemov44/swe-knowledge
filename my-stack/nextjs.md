# Next.js

## What it is
A React framework that adds server-side rendering, static generation, file-based routing, and a full-stack API layer on top of React. Built by Vercel.

## Why it matters
The majority of production React applications use Next.js. If you're applying for React roles, expect to be asked about SSR vs SSG, the App Router, and Server Components.

---

## Rendering Strategies

This is the most important concept. Next.js lets you choose per-page how content is rendered.

| Strategy | Acronym | When HTML is generated | Data freshness | Use case |
|---|---|---|---|---|
| Client-Side Rendering | CSR | In the browser | Always fresh | Dashboards, auth-gated pages |
| Server-Side Rendering | SSR | On each request (server) | Always fresh | Dynamic pages that need SEO |
| Static Site Generation | SSG | At build time | Stale until rebuild | Blog posts, marketing pages |
| Incremental Static Regen | ISR | At build + background revalidation | Revalidated on interval | Product catalog, news feed |

```typescript
// App Router (Next.js 13+) — rendering strategy is implicit

// SSR — just fetch inside a Server Component (default behavior, no cache)
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await fetch(`/api/products/${params.id}`, {
    cache: 'no-store'  // SSR — fresh on every request
  }).then(r => r.json());

  return <ProductDetail product={product} />;
}

// SSG — fetch with default caching (cached at build time)
async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetch(`/api/posts/${params.slug}`)
    .then(r => r.json()); // cached = static

  return <Article post={post} />;
}

// ISR — revalidate on an interval
async function ProductList() {
  const products = await fetch('/api/products', {
    next: { revalidate: 60 } // regenerate in background every 60 seconds
  }).then(r => r.json());

  return <List items={products} />;
}
```

---

## App Router vs Pages Router

Next.js has two routing systems. **App Router** (Next.js 13+) is the current standard.

| | Pages Router (`/pages`) | App Router (`/app`) |
|---|---|---|
| Directory | `pages/` | `app/` |
| Default rendering | Client components | Server Components |
| Data fetching | `getServerSideProps`, `getStaticProps` | `async/await` in component |
| Layouts | `_app.tsx` (global) | `layout.tsx` per segment |
| Loading states | Manual | `loading.tsx` per segment |
| Error handling | Manual | `error.tsx` per segment |

```
app/
  layout.tsx           ← root layout (shared nav, fonts, providers)
  page.tsx             ← /
  loading.tsx          ← shown while page.tsx is loading
  error.tsx            ← shown if page.tsx throws
  products/
    page.tsx           ← /products
    [id]/
      page.tsx         ← /products/123
  api/
    products/
      route.ts         ← GET /api/products (Route Handler)
```

---

## Server Components vs Client Components

The **core mental model** of the App Router.

| | Server Component | Client Component |
|---|---|---|
| Runs on | Server only | Browser (and server for hydration) |
| Can `async/await` | Yes | No |
| Can use hooks | No | Yes |
| Can access DB / secrets | Yes | No |
| Sent to browser | HTML only (no JS) | JS bundle included |
| Default in App Router | Yes | No — opt in with `'use client'` |

```typescript
// Server Component — default, no directive needed
// Runs on the server. DB access is fine. No hooks.
async function ProductList() {
  const products = await db.products.findMany(); // direct DB access

  return (
    <ul>
      {products.map(p => (
        <ProductCard key={p.id} product={p} /> // Server or Client component
      ))}
    </ul>
  );
}
```

```typescript
// Client Component — opt in with 'use client'
// Can use hooks, browser APIs, event handlers
'use client';

import { useState } from 'react';

function AddToCartButton({ productId }: { productId: string }) {
  const [loading, setLoading] = useState(false);

  return (
    <button onClick={() => addToCart(productId)}>
      Add to Cart
    </button>
  );
}
```

**Key rules:**
- Server Components can import Client Components — OK
- Client Components **cannot** import Server Components
- Pass Server Component output as `children` prop to Client Components — OK
- Put `'use client'` as high as needed but as low as possible (keep most of the tree as Server Components)

---

## Data Fetching in App Router

```typescript
// Server Component — fetch directly, no useEffect
async function UserProfile({ userId }: { userId: string }) {
  // Parallel fetches — both start simultaneously
  const [user, orders] = await Promise.all([
    getUser(userId),
    getOrders(userId),
  ]);

  return <Profile user={user} orders={orders} />;
}

// Client Component — use SWR or TanStack Query for client-side fetching
'use client';
import { useQuery } from '@tanstack/react-query';

function ProductSearch() {
  const [query, setQuery] = useState('');
  const { data } = useQuery({
    queryKey: ['products', query],
    queryFn: () => fetch(`/api/products?q=${query}`).then(r => r.json()),
  });
}
```

**Rule:** Fetch data in Server Components when possible. Only use client-side fetching for user-driven, dynamic interactions (search, filters, real-time).

---

## Route Handlers (API Routes)

Replace the `pages/api/` pattern.

```typescript
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const category = searchParams.get('category');

  const products = await db.products.findMany({
    where: category ? { categoryId: category } : undefined,
  });

  return NextResponse.json(products);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const product = await db.products.create({ data: body });
  return NextResponse.json(product, { status: 201 });
}

// app/api/products/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const product = await db.products.findUnique({ where: { id: params.id } });
  if (!product) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(product);
}
```

---

## Layouts and Loading

```typescript
// app/layout.tsx — root layout, wraps every page
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Navbar />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — nested layout for /dashboard/* routes
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />
      <div className="flex-1">{children}</div>
    </div>
  );
}

// app/products/loading.tsx — shown while page.tsx is fetching
export default function Loading() {
  return <ProductListSkeleton />;
}

// app/products/error.tsx — shown if page.tsx throws
'use client';
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Something went wrong: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

---

## Middleware

Runs before every request. Use for auth checks, redirects, locale routing.

```typescript
// middleware.ts — at root, next to app/
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/account/:path*'],
};
```

---

## Next.js vs Plain React (Vite)

| Aspect | Next.js | Vite + React |
|---|---|---|
| SEO | Built-in (SSR/SSG) | Needs extra config |
| Initial load | Faster (pre-rendered HTML) | Blank page until JS loads |
| API routes | Built-in | Separate backend needed |
| Complexity | Higher | Lower |
| Bundle size | Larger default | Leaner |
| Best for | Marketing sites, e-commerce, public apps | Dashboards, internal tools, SPAs |

**When to choose Next.js:** SEO matters, public-facing content, want SSR or SSG, want collocated API routes.
**When to choose Vite+React:** Auth-gated SPA, no SEO requirements, lighter footprint, or you have a separate backend.

---

## How It Connects to What You Know

- Server Components = async functions that return JSX — same React you know, no hooks
- Client Components = React components you already write, just add `'use client'`
- Route Handlers = equivalent to your ASP.NET Core controllers, but in Next.js
- Layouts = equivalent to `_app.tsx` but scoped per route segment
- `loading.tsx` = automatic Suspense boundary — same concept as `<Suspense>` in React
- ISR = like a cache with TTL — same idea as Redis TTL in your backend caching

---

## Common Interview Questions

1. What is the difference between SSR and SSG?
2. What is ISR and when would you use it?
3. What is a Server Component? What can it do that a Client Component can't?
4. When would you put `'use client'` in a component?
5. What is the App Router? How is it different from the Pages Router?
6. How do you fetch data in a Server Component?
7. When would you choose Next.js over a plain Vite + React setup?

---

## Common Mistakes

- Putting `'use client'` at the top of every file — defeats the purpose of Server Components
- Fetching data in Client Components with `useEffect` when a Server Component would be simpler
- Mixing App Router and Pages Router patterns in the same project
- Calling Route Handlers from Server Components (you can query DB directly — no need for a round trip)
- Not using `loading.tsx` — pages feel slow without it

---

## My Confidence Level
- `[ ]` SSR vs SSG vs ISR vs CSR — trade-offs and when to use each
- `[ ]` App Router directory structure (layout, page, loading, error)
- `[ ]` Server Components vs Client Components — rules, constraints
- `[ ]` Data fetching in Server Components (async/await, cache options)
- `[ ]` Route Handlers (API routes in App Router)
- `[ ]` Middleware — auth checks, redirects
- `[ ]` Next.js vs Vite + React — when to choose which

## My Notes
<!-- Personal notes -->
