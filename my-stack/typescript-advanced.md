# TypeScript Advanced Types

## What it is
TypeScript's type system beyond the basics. These features let you express complex type relationships, write generic utilities, and make impossible states unrepresentable at compile time.

## Why it matters
Every mid-level React + TypeScript interview tests these. They eliminate entire classes of runtime bugs by encoding business rules in types.

---

## Utility Types

Built-in generic type transformations. Know all of these cold.

```typescript
type User = {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
};

// Partial<T> — all properties optional
type UpdateUserDto = Partial<User>;
// { id?: number; name?: string; email?: string; ... }

// Required<T> — all properties required (opposite of Partial)
type StrictUser = Required<Partial<User>>;

// Readonly<T> — all properties read-only, compile error if mutated
type ImmutableUser = Readonly<User>;

// Pick<T, K> — select a subset of properties
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: number; name: string }

// Omit<T, K> — remove specific properties
type PublicUser = Omit<User, 'password'>;
// { id: number; name: string; email: string; createdAt: Date }

// Record<K, V> — object type with specific key/value types
type RolePermissions = Record<'admin' | 'user' | 'guest', string[]>;

// Exclude<T, U> — remove members from a union
type Status = 'pending' | 'active' | 'deleted';
type ActiveStatus = Exclude<Status, 'deleted'>; // 'pending' | 'active'

// Extract<T, U> — keep only matching union members
type OnlyDeleted = Extract<Status, 'deleted' | 'archived'>; // 'deleted'

// NonNullable<T> — remove null and undefined
type SafeString = NonNullable<string | null | undefined>; // string

// ReturnType<T> — extract return type of a function
function getUser() { return { id: 1, name: 'Ivan' }; }
type UserType = ReturnType<typeof getUser>; // { id: number; name: string }

// Parameters<T> — extract parameter types as a tuple
type CreateParams = Parameters<(name: string, age: number) => void>; // [string, number]

// Awaited<T> — unwrap a Promise
type Resolved = Awaited<Promise<string>>; // string
```

---

## Discriminated Unions

A union where every member has a **shared literal type field**. TypeScript narrows automatically based on that field.

```typescript
// Without — hard to narrow, easy to forget cases
type Shape = { radius?: number; width?: number; height?: number };

// With discriminated union — TypeScript knows exactly which type at each branch
type Circle    = { kind: 'circle';    radius: number };
type Rectangle = { kind: 'rectangle'; width: number; height: number };
type Triangle  = { kind: 'triangle';  base: number; height: number };

type Shape = Circle | Rectangle | Triangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;   // TypeScript: shape is Circle
    case 'rectangle':
      return shape.width * shape.height;    // TypeScript: shape is Rectangle
    case 'triangle':
      return 0.5 * shape.base * shape.height;
    default:
      const _exhaustive: never = shape;     // compile error if a case is missing
      throw new Error('Unknown shape');
  }
}
```

**In your E-commerce app — async state machine:**
```typescript
type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function ProductList({ state }: { state: ApiState<Product[]> }) {
  switch (state.status) {
    case 'loading': return <Spinner />;
    case 'error':   return <ErrorMessage message={state.error} />;
    case 'success': return <List items={state.data} />;
    case 'idle':    return null;
  }
}
```

---

## Type Guards

Expressions that narrow a union type within a block.

```typescript
// typeof — for primitives
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase(); // string here
  }
  return value.toFixed(2); // number here
}

// instanceof — for classes
function handle(error: Error | ValidationError) {
  if (error instanceof ValidationError) {
    return error.fieldErrors; // ValidationError here
  }
  return error.message;
}

// in — check if a property exists on the object
type Cat = { meow(): void };
type Dog = { bark(): void };

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // Cat here
  } else {
    animal.bark(); // Dog here
  }
}

// User-defined type guard — the `is` keyword
function isProduct(item: Product | CartItem): item is Product {
  return 'stock' in item;
}

function displayItem(item: Product | CartItem) {
  if (isProduct(item)) {
    console.log(item.stock); // Product here — TypeScript trusts your guard
  }
}
```

**Rule:** Use `as` casts only as a last resort. Type guards are self-documenting and safe.

---

## Mapped Types

Create a new type by transforming each property of an existing type.

```typescript
// What Partial<T> does internally
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties nullable
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

// Real use case: form state derived from a model
type FormState<T> = {
  [K in keyof T]: {
    value: T[K];
    touched: boolean;
    error: string | null;
  };
};

type ProductFormState = FormState<Pick<Product, 'name' | 'price'>>;
// {
//   name:  { value: string;  touched: boolean; error: string | null };
//   price: { value: number; touched: boolean; error: string | null };
// }
```

---

## Conditional Types

Types that choose between alternatives based on whether `T extends U`.

```typescript
// Syntax: T extends U ? TrueType : FalseType
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Flatten arrays
type Flatten<T> = T extends Array<infer U> ? U : T;
type Num  = Flatten<number[]>; // number
type Str  = Flatten<string>;   // string (not an array, passes through)

// infer — extract a type from a conditional
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;
type MyRet = ReturnTypeOf<() => string>; // string

// Distribute over unions
type ToArray<T> = T extends any ? T[] : never;
type StringOrNumberArray = ToArray<string | number>; // string[] | number[]
```

---

## Template Literal Types

String literal types built with template literal syntax.

```typescript
type EventName = 'click' | 'focus' | 'blur';
type Handler = `on${Capitalize<EventName>}`; // 'onClick' | 'onFocus' | 'onBlur'

// Typed event names on a domain object
type DomainEvent = 'OrderPlaced' | 'OrderShipped' | 'OrderCancelled';
type EventListener = `on${DomainEvent}`;
// 'onOrderPlaced' | 'onOrderShipped' | 'onOrderCancelled'

// API route typing
type Resource = 'users' | 'products' | 'orders';
type GetRoute = `GET /api/${Resource}`;
```

---

## `satisfies` Operator (TypeScript 4.9+)

Validates a value against a type **without widening it**. You get both type safety AND preserved literal types.

```typescript
type Colors = Record<string, string | [number, number, number]>;

// With annotation — TypeScript widens to the annotation type
const palette: Colors = {
  red: [255, 0, 0],
  green: '#00ff00',
};
palette.red.map(v => v); // Error! TypeScript sees string | number[], not number[]

// With satisfies — validated against Colors AND literal types preserved
const palette2 = {
  red: [255, 0, 0],
  green: '#00ff00',
} satisfies Colors;

palette2.red.map(v => v);        // Works! TypeScript knows number[]
palette2.green.toUpperCase();    // Works! TypeScript knows string

// Real use case: config objects
type Config = {
  port: number;
  env: 'dev' | 'prod' | 'test';
};

const config = {
  port: 3000,
  env: 'dev',
} satisfies Config;
// config.env is 'dev' (literal), not the wide 'dev' | 'prod' | 'test'
```

---

## Generic Constraints

```typescript
// Constrain T to have specific shape
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'Ivan' };
const name = getProperty(user, 'name'); // TypeScript infers: string
// getProperty(user, 'age'); // Error: 'age' not a key of user

// Constrain to entity base type
function processEntity<T extends { id: string; createdAt: Date }>(entity: T): T {
  console.log(`Processing ${entity.id}`);
  return entity;
}

// Multiple constraints with &
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

---

## Intersection vs Union

```typescript
// Union (|) — EITHER one OR another, only shared members accessible without narrowing
type StringOrNumber = string | number;

// Intersection (&) — ALL types combined, must have ALL properties
type AdminUser = User & { permissions: string[] };

// Practical: building DTOs
type CreateProductRequest =
  Pick<Product, 'name' | 'price' | 'stock'> & {
    categoryId: string;
    images: string[];
  };
```

---

## Common Interview Questions

1. What is the difference between `Partial<T>` and `Required<T>`?
2. What is a discriminated union? When would you use one?
3. How does a user-defined type guard (`value is T`) work?
4. What is the difference between `Omit<T, K>` and `Exclude<T, U>`?
5. What does `satisfies` do differently from a type annotation?
6. How would you type an API response that can be `idle | loading | success | error`?
7. What is a mapped type? Give a real example.
8. What is `infer` used for?

---

## Common Mistakes

- Using `as` casts instead of type guards — hides type errors
- Not using `never` for exhaustive switches — silent missing case bug
- Confusing `Omit` (removes object keys) with `Exclude` (removes union members)
- Putting validation logic in types instead of runtime checks — types are erased at runtime

---

## How It Connects

- Discriminated unions replace `if (loading) / if (error)` with type-safe state machines
- `Partial<T>` is used for PATCH request DTOs in RTK Query mutations
- `Omit<T, 'password'>` creates `PublicUserDto` without sensitive fields
- `ReturnType<T>` is useful for inferring what RTK Query endpoint hooks return
- Type guards work with React conditional rendering — narrows inside JSX

---

## My Confidence Level
- `[ ]` Utility types (Partial, Required, Pick, Omit, Record, Exclude, Extract, NonNullable, ReturnType, Awaited)
- `[ ]` Discriminated unions + exhaustive narrowing with never
- `[ ]` Type guards (typeof, instanceof, in, user-defined is)
- `[ ]` Mapped types
- `[ ]` Conditional types + infer
- `[ ]` Template literal types
- `[ ]` satisfies operator
- `[ ]` Generic constraints

## My Notes
<!-- Personal notes -->
