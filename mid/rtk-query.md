# RTK Query

## What it is
RTK Query is a data fetching and caching layer built into Redux Toolkit. It eliminates the need to write thunks, reducers, and loading/error state for every API call. Think of it as a declarative API client with automatic caching.

## Why it matters
Without RTK Query, you'd write dozens of thunks, manually manage loading/error state, and duplicate cache logic. RTK Query does all of this for you with minimal code.

---

## Setup

```typescript
// app/baseApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const baseApi = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({
    baseUrl: import.meta.env.VITE_API_URL,
    prepareHeaders: (headers, { getState }) => {
      const token = (getState() as RootState).auth.accessToken;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Product', 'Order', 'Cart'],
  endpoints: () => ({}),
});

// store.ts
export const store = configureStore({
  reducer: {
    [baseApi.reducerPath]: baseApi.reducer,
    auth: authSlice.reducer,
    cart: cartSlice.reducer,
  },
  middleware: (getDefault) => getDefault().concat(baseApi.middleware),
});
```

---

## Defining Endpoints

```typescript
// features/products/productsApi.ts
export const productsApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    // Query (GET)
    getProducts: builder.query<PaginatedResponse<ProductDto>, GetProductsParams>({
      query: ({ page = 1, pageSize = 20, category }) => ({
        url: '/products',
        params: { page, pageSize, category },
      }),
      providesTags: (result) =>
        result
          ? [
              ...result.items.map(({ id }) => ({ type: 'Product' as const, id })),
              { type: 'Product', id: 'LIST' },
            ]
          : [{ type: 'Product', id: 'LIST' }],
    }),

    getProductById: builder.query<ProductDto, string>({
      query: (id) => `/products/${id}`,
      providesTags: (result, error, id) => [{ type: 'Product', id }],
    }),

    // Mutation (POST/PUT/DELETE)
    createProduct: builder.mutation<ProductDto, CreateProductRequest>({
      query: (body) => ({
        url: '/products',
        method: 'POST',
        body,
      }),
      invalidatesTags: [{ type: 'Product', id: 'LIST' }],
    }),

    updateProduct: builder.mutation<ProductDto, { id: string; body: UpdateProductRequest }>({
      query: ({ id, body }) => ({
        url: `/products/${id}`,
        method: 'PUT',
        body,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'Product', id }],
    }),

    deleteProduct: builder.mutation<void, string>({
      query: (id) => ({
        url: `/products/${id}`,
        method: 'DELETE',
      }),
      invalidatesTags: (result, error, id) => [
        { type: 'Product', id },
        { type: 'Product', id: 'LIST' },
      ],
    }),
  }),
});

export const {
  useGetProductsQuery,
  useGetProductByIdQuery,
  useCreateProductMutation,
  useUpdateProductMutation,
  useDeleteProductMutation,
} = productsApi;
```

---

## Using in Components

```tsx
const ProductsPage = () => {
  const [page, setPage] = useState(1);
  const { data, isLoading, isError, error } = useGetProductsQuery({ page, pageSize: 20 });

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;

  return (
    <>
      {data?.items.map(product => <ProductCard key={product.id} product={product} />)}
      <Pagination total={data?.total} page={page} onChange={setPage} />
    </>
  );
};

// Mutation
const AddProductForm = () => {
  const [createProduct, { isLoading }] = useCreateProductMutation();

  const handleSubmit = async (data: CreateProductRequest) => {
    try {
      await createProduct(data).unwrap(); // .unwrap() throws on error
      showSuccessToast('Product created');
    } catch (err) {
      showErrorToast('Failed to create product');
    }
  };
};
```

---

## Cache Tags (Automatic Cache Invalidation)

Cache tags are the core of RTK Query's caching strategy.

**`providesTags`:** What data this endpoint "owns"
**`invalidatesTags`:** What data this mutation "breaks" → triggers refetch of matching queries

```
getProducts → providesTags: [{ type: 'Product', id: 'LIST' }]
getProductById(id) → providesTags: [{ type: 'Product', id }]

createProduct → invalidatesTags: [{ type: 'Product', id: 'LIST' }]
  → triggers refetch of getProducts (list is now stale)

updateProduct(id) → invalidatesTags: [{ type: 'Product', id }]
  → triggers refetch of getProductById(id)

deleteProduct(id) → invalidatesTags: [{ type: 'Product', id }, { type: 'Product', id: 'LIST' }]
  → triggers refetch of both
```

---

## Optimistic Updates

Update UI immediately before the server responds. Roll back on failure.

```typescript
updateProduct: builder.mutation<ProductDto, { id: string; body: UpdateProductRequest }>({
  query: ({ id, body }) => ({ url: `/products/${id}`, method: 'PUT', body }),

  async onQueryStarted({ id, body }, { dispatch, queryFulfilled }) {
    // Optimistically update the cache
    const patchResult = dispatch(
      productsApi.util.updateQueryData('getProductById', id, (draft) => {
        Object.assign(draft, body);
      })
    );

    try {
      await queryFulfilled;
    } catch {
      patchResult.undo(); // Roll back on failure
    }
  },
}),
```

---

## Polling

Automatically refetch on an interval:

```tsx
const { data } = useGetOrderStatusQuery(orderId, {
  pollingInterval: 5000, // refetch every 5 seconds
  skip: order?.status === 'delivered', // stop when complete
});
```

---

## Conditional Fetching

```tsx
// Only fetch if userId exists
const { data } = useGetUserProfileQuery(userId, {
  skip: !userId,
});

// Or: skip until condition is met
const { data } = useGetOrderByIdQuery(orderId ?? skipToken);
```

---

## selectFromResult (Performance)

Avoid re-renders when only part of the data changes:

```tsx
const productName = useGetProductsQuery(undefined, {
  selectFromResult: ({ data }) => ({
    productName: data?.items.find(p => p.id === id)?.name,
  }),
});
// Only re-renders if productName specifically changes
```

---

## RTK Query vs Slices

| | RTK Query | Redux Slice |
|---|---|---|
| Purpose | Server / async state | UI / client-only state |
| Examples | Products list, orders, user profile | Modal open/closed, filter state, form draft |
| Manages | Loading, error, caching, refetching | Local UI interactions |
| Source of truth | API / backend | Component interaction |

**Rule:** If it comes from the server, it lives in RTK Query. If it's purely UI state, it lives in a slice.

```typescript
// Slice (UI state only)
const cartSlice = createSlice({
  name: 'cart',
  initialState: { isOpen: false },
  reducers: {
    openCart: (state) => { state.isOpen = true; },
    closeCart: (state) => { state.isOpen = false; },
  },
});
```

---

## Common Interview Questions

1. What is the difference between RTK Query and a Redux thunk?
2. What are cache tags and how do they work?
3. When would you use a Redux slice vs RTK Query?
4. What is `.unwrap()` and why use it?
5. How do optimistic updates work in RTK Query?

---

## Common Mistakes

- Putting server state in slices instead of RTK Query
- Missing `tagTypes` declaration in `createApi`
- Forgetting `{ type: 'Entity', id: 'LIST' }` tag on list queries (mutations won't invalidate them)
- Not using `.unwrap()` and missing errors in mutation handlers
- Fetching data in `useEffect` instead of using a query hook

---

## How It Connects

- RTK Query is the frontend equivalent of your backend's `IUnitOfWork` — single place to manage data access
- Cache tags are the frontend equivalent of cache invalidation keys in Redis
- `providesTags` / `invalidatesTags` = cache invalidation strategy
- `skip` / `skipToken` = conditional dependency injection

---

## My Confidence Level
- `[c]` injectEndpoints, baseApi
- `[c]` Cache tags, invalidation
- `[b]` Optimistic updates
- `[b]` Polling
- `[~]` Streaming / subscriptions

## My Notes
<!-- Personal notes -->
