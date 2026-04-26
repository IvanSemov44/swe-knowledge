# TypeScript with React

## What it is
How to type React-specific patterns: component props, events, refs, hooks, context, and generics in JSX. TypeScript + React together have their own idioms that differ from plain TypeScript.

## Why it matters
Every React interview with TypeScript tests these. Typing props incorrectly is the most common beginner mistake. Generic components and properly typed hooks separate mid-level from junior.

---

## Typing Props

```typescript
// Interface for props (preferred for component contracts — extendable)
interface ProductCardProps {
  product: Product;
  onAddToCart: (productId: string) => void;
  variant?: 'compact' | 'full'; // optional with literal union
  className?: string;
}

// Type alias also fine — use when you need unions or intersections
type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  variant: 'primary' | 'secondary' | 'ghost';
  loading?: boolean;
};

function ProductCard({ product, onAddToCart, variant = 'full' }: ProductCardProps) {
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
}
```

---

## Children Props

```typescript
// React.ReactNode — anything renderable (most permissive, use for layout components)
interface CardProps {
  children: React.ReactNode;
  title: string;
}

// React.ReactElement — a single React element (not string/number/null)
interface WrapperProps {
  children: React.ReactElement;
}

// Specific children (compound components)
interface FormProps {
  children: React.ReactElement<typeof FormField> | React.ReactElement<typeof FormField>[];
}

// PropsWithChildren<T> — shortcut to add children to an existing props type
type CardProps = React.PropsWithChildren<{ title: string }>;
```

---

## Event Types

```typescript
// Input events
function SearchBar() {
  const [query, setQuery] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    search(query);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={query} onChange={handleChange} />
    </form>
  );
}

// Button / mouse events
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  e.stopPropagation();
};

// Keyboard events
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === 'Enter') submit();
};

// Drag events
const handleDrop = (e: React.DragEvent<HTMLDivElement>) => {
  const data = e.dataTransfer.getData('text/plain');
};

// Focus events
const handleFocus = (e: React.FocusEvent<HTMLInputElement>) => {};

// Shorthand — React.ChangeEventHandler<HTMLInputElement> = (e: ChangeEvent<HTMLInputElement>) => void
const handleChange: React.ChangeEventHandler<HTMLInputElement> = (e) => setQuery(e.target.value);
```

---

## Typing Hooks

```typescript
// useState — inferred from initial value
const [count, setCount] = useState(0);          // number
const [name, setName] = useState('');            // string
const [user, setUser] = useState<User | null>(null); // explicit when starting null

// useState with complex types
const [items, setItems] = useState<CartItem[]>([]);

// useRef — two forms
// 1. DOM ref — starts null, assign to element
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus(); // ?. because null before mount

// 2. Mutable ref — holds a value across renders, not null
const timerRef = useRef<NodeJS.Timeout | undefined>(undefined);
timerRef.current = setTimeout(() => {}, 1000);

// useReducer
type CartAction =
  | { type: 'ADD_ITEM'; payload: CartItem }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'CLEAR' };

function cartReducer(state: CartItem[], action: CartAction): CartItem[] {
  switch (action.type) {
    case 'ADD_ITEM':    return [...state, action.payload];
    case 'REMOVE_ITEM': return state.filter(i => i.id !== action.payload);
    case 'CLEAR':       return [];
  }
}

const [cart, dispatch] = useReducer(cartReducer, []);
dispatch({ type: 'ADD_ITEM', payload: item });
```

---

## Typing Context

```typescript
// 1. Define the shape
interface AuthContextValue {
  user: User | null;
  login: (credentials: LoginDto) => Promise<void>;
  logout: () => void;
}

// 2. Create context with undefined default (force consumers to check)
const AuthContext = React.createContext<AuthContextValue | undefined>(undefined);

// 3. Custom hook — throws if used outside provider
function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (context === undefined)
    throw new Error('useAuth must be used within AuthProvider');
  return context;
}

// 4. Provider
function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (credentials: LoginDto) => {
    const user = await authApi.login(credentials);
    setUser(user);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout: () => setUser(null) }}>
      {children}
    </AuthContext.Provider>
  );
}

// 5. Usage
function UserMenu() {
  const { user, logout } = useAuth(); // typed, throws if outside provider
}
```

---

## Generic Components

```typescript
// Generic list component — works with any item type
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage — T is inferred from items
<List
  items={products}
  keyExtractor={p => p.id}
  renderItem={p => <ProductCard product={p} onAddToCart={addToCart} />}
/>

// Generic select component
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string;
}

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select
      value={value ? getValue(value) : ''}
      onChange={e => {
        const selected = options.find(o => getValue(o) === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {options.map(opt => (
        <option key={getValue(opt)} value={getValue(opt)}>{getLabel(opt)}</option>
      ))}
    </select>
  );
}
```

---

## forwardRef

```typescript
// Typing a component that forwards its ref to a DOM element
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const TextInput = React.forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...props }, ref) => (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
      {error && <span>{error}</span>}
    </div>
  )
);

TextInput.displayName = 'TextInput'; // for React DevTools

// Usage
const inputRef = useRef<HTMLInputElement>(null);
<TextInput ref={inputRef} label="Email" type="email" />
inputRef.current?.focus();
```

---

## ComponentProps Utility

Extract props from an existing component without re-defining the type.

```typescript
// Get props of a component
type ButtonProps = React.ComponentProps<'button'>; // native button props
type CardProps = React.ComponentProps<typeof Card>; // your custom component's props

// Extend native element props
type TextInputProps = React.ComponentProps<'input'> & {
  label: string;
  error?: string;
};
```

---

## Typing RTK Query Hooks

```typescript
// RTK Query infers types from the endpoint definition
const { data: products, isLoading, error } = useGetProductsQuery({ page: 1 });
// products is Product[] | undefined
// isLoading is boolean
// error is FetchBaseQueryError | SerializedError | undefined

// Mutation
const [createOrder, { isLoading: isCreating }] = useCreateOrderMutation();

const handleSubmit = async () => {
  try {
    const result = await createOrder(orderData).unwrap(); // throws on error
    navigate(`/orders/${result.id}`);
  } catch (error) {
    // error is typed
  }
};
```

---

## Common Interview Questions

1. What is the difference between `React.ReactNode` and `React.ReactElement`?
2. How do you type an event handler for an input's `onChange`?
3. How do you type `useState` when the initial value is null?
4. What is the difference between the two forms of `useRef`?
5. How do you create a generic React component?
6. What is `forwardRef` and when do you need it?
7. How do you type a `useContext` to avoid null checks everywhere?

---

## Common Mistakes

- Using `any` for event types instead of `React.ChangeEvent<HTMLInputElement>`
- Typing `useRef` as `useRef<HTMLInputElement>()` without `null` (should be `useRef<HTMLInputElement>(null)` for DOM refs)
- Not creating a custom hook for context (forces consumers to check for undefined)
- Using `React.FC` — it implicitly adds `children` and has other quirks; just use function with explicit props type
- Forgetting `displayName` on `forwardRef` components (unnamed in React DevTools)

---

## How It Connects

- Discriminated unions from `typescript-advanced.md` are how you type `CartAction` in `useReducer`
- Generic components are the TypeScript version of the Render Props pattern from `react-patterns.md`
- Context typing with a custom hook is how you expose Redux store slices in a testable way
- `ComponentProps<'button'>` is how you extend native element types without repeating every HTML attribute
- RTK Query hook types come from your endpoint definitions — TypeScript catches mismatched response types at compile time

---

## My Confidence Level
- `[ ]` Typing props — interface vs type, optional, literal unions
- `[ ]` Children props — ReactNode vs ReactElement, PropsWithChildren
- `[ ]` Event types — ChangeEvent, FormEvent, MouseEvent, KeyboardEvent
- `[ ]` Typing hooks — useState<T>, useRef forms, useReducer with discriminated union
- `[ ]` Typing Context — undefined default + custom hook pattern
- `[ ]` Generic components — List<T>, Select<T>
- `[ ]` forwardRef — typing + displayName
- `[ ]` ComponentProps utility

## My Notes
<!-- Personal notes -->
