# React Patterns

## Why This Matters for Interviews
Mid-level React interviews regularly ask about patterns beyond basic hooks. These come up as "how would you design X" or "what pattern is this code using" questions.

---

## Error Boundaries

Class component that catches JavaScript errors in its child tree and renders a fallback UI instead of crashing the whole app.

```tsx
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  fallback: ReactNode;
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error('Error boundary caught:', error, info.componentStack);
    // Send to error monitoring (Sentry, etc.)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <ProductList />
</ErrorBoundary>
```

**Key facts:**
- Must be a class component (no hook equivalent yet)
- Does NOT catch errors in event handlers or async code (only render-time errors)
- `react-error-boundary` library gives a hook-friendly API

---

## Higher-Order Components (HOC)

A function that takes a component and returns a new component with enhanced behavior.

```tsx
// withAuth.tsx — protect routes/components
function withAuth<T extends object>(WrappedComponent: React.ComponentType<T>) {
  return function AuthGuard(props: T) {
    const { user } = useAuth();

    if (!user) {
      return <Navigate to="/login" replace />;
    }

    return <WrappedComponent {...props} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(DashboardPage);
```

**When to use HOCs:**
- Cross-cutting concerns (auth, logging, permissions) applied to many components
- You need to wrap a third-party component you can't modify

**Downsides:**
- Prop drilling / prop collisions
- Hard to trace in DevTools (wrapper hell)
- Modern preference: custom hooks solve most of the same problems more cleanly

---

## Render Props

A component that receives a function as a prop and calls it to determine what to render. Shares stateful logic without HOCs.

```tsx
interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  render: (position: MousePosition) => ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });

  return (
    <div onMouseMove={e => setPosition({ x: e.clientX, y: e.clientY })}>
      {render(position)}
    </div>
  );
}

// Usage
<MouseTracker render={({ x, y }) => <p>Mouse: {x}, {y}</p>} />
```

**Modern note:** Custom hooks have largely replaced render props for logic reuse. Render props still appear in libraries (React Router's `<Route render={...}>` old API, react-table, etc.).

---

## Compound Components

Break a complex component into multiple sub-components that share implicit state via Context. Gives users a flexible, expressive API.

```tsx
// Classic example: a custom Select/Tabs component

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | undefined>(undefined);

function Tabs({ children, defaultTab }: { children: ReactNode; defaultTab: string }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab, setActiveTab } = useContext(TabsContext)!;
  return (
    <button
      className={activeTab === id ? 'active' : ''}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab } = useContext(TabsContext)!;
  return activeTab === id ? <div>{children}</div> : null;
}

// Attach sub-components to parent
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — clean, declarative API
<Tabs defaultTab="overview">
  <Tabs.Tab id="overview">Overview</Tabs.Tab>
  <Tabs.Tab id="reviews">Reviews</Tabs.Tab>

  <Tabs.Panel id="overview"><ProductOverview /></Tabs.Panel>
  <Tabs.Panel id="reviews"><ProductReviews /></Tabs.Panel>
</Tabs>
```

**Why it's powerful:** The parent manages state, children are declarative, user of the component controls composition.

---

## Portals

Render a child component into a different DOM node than its parent — useful for modals, tooltips, and dropdowns that need to escape overflow/z-index constraints.

```tsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')! // outside the main React root
  );
}
```

**Key facts:**
- The portal is rendered outside the parent DOM tree, but still inside the React component tree
- Events bubble up through the React tree (not the DOM tree) — `onClick` on a parent of `Modal` still fires

---

## Custom Hook Pattern (Logic Extraction)

Not a "pattern" per se, but the modern replacement for HOCs and render props:

```tsx
// Replaces the MouseTracker render prop example above
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  return position;
}

// Usage — no wrapper components
function MyComponent() {
  const { x, y } = useMousePosition();
  return <p>Mouse: {x}, {y}</p>;
}
```

---

## Pattern Evolution (Interview Context)

| Problem | Old solution | Modern solution |
|---|---|---|
| Share stateful logic | HOC / Render Props | Custom Hook |
| Flexible component API | Props explosion | Compound Components |
| Escape DOM constraints | - | Portals |
| Handle render errors | - | Error Boundary |

---

## Common Interview Questions

1. What is an Error Boundary? What errors does it NOT catch?
2. What is an HOC? Give an example use case and a downside.
3. What is the render props pattern?
4. How do compound components use Context internally?
5. What is a Portal and when would you use one?
6. How have custom hooks changed the need for HOCs and render props?

---

## Common Mistakes

- Expecting Error Boundaries to catch async errors (they don't)
- HOC prop name collisions (wrapping overrides a prop the inner component needs)
- Using Compound Components without protecting against `useContext` returning `undefined` (always throw if outside provider)
- Portals: forgetting that DOM events bubble through the DOM tree, not React tree

---

## How It Connects

- Error Boundaries often wrap Suspense boundaries (loading + error handled together)
- Compound Components use Context API internally — same mechanism, different purpose
- Custom hooks replaced most HOC/render-prop use cases post React 16.8 (hooks release)
- Portals are how most UI component libraries implement modals, tooltips, and select dropdowns

---

## My Confidence Level
- `[ ]` Error Boundaries — when to use, what they don't catch
- `[ ]` HOC — implement one, explain tradeoffs
- `[ ]` Render Props — recognize and implement the pattern
- `[ ]` Compound Components — Context-based sub-component API
- `[ ]` Portals — when and how to use
- `[ ]` Custom hooks as the modern replacement for HOC/render props

## My Notes
<!-- Personal notes -->
