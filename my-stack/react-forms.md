# React Forms — React Hook Form + Zod

## What it is
React Hook Form (RHF) is the standard library for managing form state and validation. Zod is a TypeScript-first schema validation library. Together they give you type-safe, performant forms with minimal boilerplate.

## Why it matters
Forms appear in virtually every app. Interviewers ask about controlled vs uncontrolled, validation strategies, and how you handle async submission. RHF + Zod is the current industry standard for React + TypeScript.

---

## Why Not Controlled Components for Forms?

```typescript
// Pure React controlled form — re-renders on EVERY keystroke
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  // Every character typed → setState → re-render entire form
  // For 10 fields this gets messy fast
}
```

**RHF uses uncontrolled inputs under the hood** — no re-render on each keystroke. Only re-renders when validation state changes or on submit.

---

## Basic Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Define the schema — single source of truth for shape + validation
const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

// 2. Infer the TypeScript type from the schema — no duplication
type LoginFormData = z.infer<typeof loginSchema>;

// 3. Wire up the form
function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (data: LoginFormData) => {
    // data is fully typed: { email: string; password: string }
    await login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input type="email" {...register('email')} />
        {errors.email && <span>{errors.email.message}</span>}
      </div>
      <div>
        <input type="password" {...register('password')} />
        {errors.password && <span>{errors.password.message}</span>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in...' : 'Sign in'}
      </button>
    </form>
  );
}
```

---

## Zod Schema Patterns

```typescript
// String validations
z.string()
  .min(1, 'Required')
  .max(100, 'Too long')
  .email('Invalid email')
  .url('Invalid URL')
  .regex(/^[A-Z]/, 'Must start with uppercase')
  .trim()            // strip whitespace before validation
  .toLowerCase()     // normalize

// Number validations
z.number()
  .min(0, 'Must be positive')
  .max(1000, 'Too large')
  .int('Must be a whole number')
  .positive()

// Optional vs nullable
z.string().optional()          // string | undefined
z.string().nullable()          // string | null
z.string().nullish()           // string | null | undefined

// Enums
z.enum(['admin', 'user', 'guest'])

// Objects
const addressSchema = z.object({
  street: z.string().min(1, 'Required'),
  city: z.string().min(1, 'Required'),
  postCode: z.string().regex(/^\d{4}$/, 'Must be 4 digits'),
  country: z.string().optional(),
});

// Arrays
const tagsSchema = z.array(z.string()).min(1, 'Add at least one tag').max(10);

// Cross-field validation with refine
const registerSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: "Passwords don't match",
    path: ['confirmPassword'], // attach error to this field
  }
);

// Conditional validation with superRefine
const checkoutSchema = z.object({
  paymentMethod: z.enum(['card', 'bank']),
  cardNumber: z.string().optional(),
}).superRefine((data, ctx) => {
  if (data.paymentMethod === 'card' && !data.cardNumber) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Card number is required',
      path: ['cardNumber'],
    });
  }
});
```

---

## Full Product Form Example

```typescript
const createProductSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  price: z.coerce.number().positive('Price must be positive'), // coerce: input string → number
  stock: z.coerce.number().int().min(0, 'Stock cannot be negative'),
  categoryId: z.string().uuid('Select a category'),
  description: z.string().max(500).optional(),
  isActive: z.boolean().default(true),
});

type CreateProductFormData = z.infer<typeof createProductSchema>;

function CreateProductForm() {
  const [createProduct, { isLoading }] = useCreateProductMutation();

  const {
    register,
    handleSubmit,
    reset,
    setError,
    formState: { errors, isDirty, isValid },
  } = useForm<CreateProductFormData>({
    resolver: zodResolver(createProductSchema),
    defaultValues: { isActive: true },
    mode: 'onBlur', // validate on blur, not on every keystroke
  });

  const onSubmit = async (data: CreateProductFormData) => {
    try {
      await createProduct(data).unwrap();
      reset(); // clear form on success
    } catch (error) {
      // Map server errors back to fields
      if (isApiError(error) && error.data.errorCode === 'DUPLICATE_NAME') {
        setError('name', { message: 'A product with this name already exists' });
      }
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} placeholder="Product name" />
      {errors.name && <p>{errors.name.message}</p>}

      <input {...register('price')} type="number" step="0.01" />
      {errors.price && <p>{errors.price.message}</p>}

      <input {...register('stock')} type="number" />
      {errors.stock && <p>{errors.stock.message}</p>}

      <select {...register('categoryId')}>
        <option value="">Select category...</option>
        {categories.map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
      </select>
      {errors.categoryId && <p>{errors.categoryId.message}</p>}

      <textarea {...register('description')} />

      <label>
        <input type="checkbox" {...register('isActive')} />
        Active
      </label>

      <button type="submit" disabled={isLoading || !isDirty || !isValid}>
        {isLoading ? 'Creating...' : 'Create Product'}
      </button>
    </form>
  );
}
```

---

## useController — For Custom Input Components

When you can't use `register` directly (custom component, UI library):

```typescript
import { useController, Control } from 'react-hook-form';

interface PriceInputProps {
  name: string;
  control: Control<any>;
  label: string;
}

function PriceInput({ name, control, label }: PriceInputProps) {
  const {
    field: { value, onChange, onBlur, ref },
    fieldState: { error },
  } = useController({ name, control });

  return (
    <div>
      <label>{label}</label>
      <input
        ref={ref}
        value={value}
        onChange={e => onChange(parseFloat(e.target.value))}
        onBlur={onBlur}
        type="number"
      />
      {error && <span>{error.message}</span>}
    </div>
  );
}

// Usage
<PriceInput name="price" control={control} label="Price (USD)" />
```

---

## Field Arrays — Dynamic Lists

```typescript
import { useFieldArray } from 'react-hook-form';

const orderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.coerce.number().int().min(1),
  })).min(1, 'Add at least one item'),
});

function OrderForm() {
  const { control, register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(orderSchema),
    defaultValues: { items: [{ productId: '', quantity: 1 }] },
  });

  const { fields, append, remove } = useFieldArray({ control, name: 'items' });

  return (
    <form>
      {fields.map((field, index) => (
        <div key={field.id}> {/* use field.id NOT index as key */}
          <select {...register(`items.${index}.productId`)}>
            {products.map(p => <option key={p.id} value={p.id}>{p.name}</option>)}
          </select>
          <input type="number" {...register(`items.${index}.quantity`)} />
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ productId: '', quantity: 1 })}>
        Add Item
      </button>
    </form>
  );
}
```

---

## validation mode options

```typescript
useForm({
  mode: 'onSubmit',  // default — validate only on submit
  mode: 'onBlur',    // validate when field loses focus
  mode: 'onChange',  // validate on every keystroke (expensive)
  mode: 'onTouched', // validate on first blur, then on change
  mode: 'all',       // validate on both blur and change
});
```

**Recommendation:** Use `'onBlur'` for a good UX balance — no red errors while typing, but validation happens before submit.

---

## Common Interview Questions

1. Why does React Hook Form use uncontrolled inputs?
2. What is `z.infer<typeof schema>` and why use it?
3. How do you validate that two password fields match in Zod?
4. What is `z.coerce` and when do you need it?
5. How do you map a server validation error back to a specific field?
6. What is `useFieldArray` for?
7. What is the difference between `register` and `useController`?

---

## Common Mistakes

- Using `onChange` mode — validates every keystroke, causes excessive re-renders
- Forgetting `z.coerce.number()` for numeric inputs — HTML inputs always return strings
- Using `index` as key in `useFieldArray` — use `field.id` instead
- Not calling `.unwrap()` on RTK Query mutations inside forms — can't catch errors
- Forgetting `reset()` after successful submit — form stays dirty with old data

---

## How It Connects

- `z.infer<typeof schema>` produces the same DTO type used in your RTK Query mutation argument
- Server validation errors from ASP.NET Core's `ValidationFilter` can be mapped back via `setError`
- `useController` is how you integrate UI component libraries (shadcn/ui, MUI) with RHF
- `z.object()` schema mirrors your `CreateProductRequest` DTO on the backend — same shape, same constraints

---

## My Confidence Level
- `[ ]` RHF basic setup — useForm, register, handleSubmit, errors
- `[ ]` Zod schemas — string/number/object validations
- `[ ]` z.infer — type from schema, no duplication
- `[ ]` z.refine / superRefine — cross-field validation
- `[ ]` z.coerce — string to number for HTML inputs
- `[ ]` setError — mapping server errors to fields
- `[ ]` useController — custom input components
- `[ ]` useFieldArray — dynamic lists

## My Notes
<!-- Personal notes -->
