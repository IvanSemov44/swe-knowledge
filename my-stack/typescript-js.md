# TypeScript / JavaScript Deep Dive

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: c -->

---

## Closures

A function that retains references to variables from its outer (lexical) scope even after that scope has returned.

```typescript
function createCounter() {
    let count = 0;  // private — not accessible from outside

    return {
        increment: () => ++count,
        decrement: () => --count,
        value: () => count
    };
}

const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.value();     // 2
// counter.count    // undefined — no direct access
```

`count` stays alive because the returned methods still reference it.

### Classic bug: var in a loop

```typescript
// var — one shared reference, logs 3, 3, 3
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}

// let — block-scoped, new binding each iteration, logs 0, 1, 2
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}
```

### Practical uses

```typescript
// Debounce — captures timer reference across calls
function debounce(fn: Function, ms: number) {
    let timer: ReturnType<typeof setTimeout>;
    return (...args: any[]) => {
        clearTimeout(timer);
        timer = setTimeout(() => fn(...args), ms);
    };
}

// Memoization — captures cache
function memoize<T>(fn: (arg: number) => T) {
    const cache = new Map<number, T>();
    return (arg: number): T => {
        if (cache.has(arg)) return cache.get(arg)!;
        const result = fn(arg);
        cache.set(arg, result);
        return result;
    };
}
```

---

## Prototype Chain

Every object has a hidden `[[Prototype]]` link. Property lookups walk up the chain until found or null.

```typescript
const animal = { breathes: true };
const dog = Object.create(animal);
dog.barks = true;

dog.barks;    // found on dog
dog.breathes; // not on dog → walks up to animal → found
dog.flies;    // not on dog, not on animal, not on Object.prototype → undefined
```

Classes in JS/TS are syntactic sugar over prototype chains:

```typescript
class Animal { breathes = true; }
class Dog extends Animal { barks = true; }

// Under the hood:
// Dog.prototype.__proto__ === Animal.prototype
```

---

## Event Loop

JavaScript is single-threaded. Async work is handled by the event loop.

```
Call Stack → runs synchronous code
Web APIs   → handles async (setTimeout, fetch, etc.)
Task Queue → macrotasks (setTimeout callbacks)
Microtask Queue → promises, queueMicrotask (runs before next macrotask)
```

```typescript
console.log('1');

setTimeout(() => console.log('2'), 0);  // macrotask

Promise.resolve().then(() => console.log('3'));  // microtask

console.log('4');

// Output: 1, 4, 3, 2
// Microtasks drain before next macrotask
```

---

## async/await & Promises

```typescript
// Promise chain
fetch('/api/orders')
    .then(res => res.json())
    .then(data => console.log(data))
    .catch(err => console.error(err));

// async/await (same thing, cleaner)
async function getOrders() {
    try {
        const res = await fetch('/api/orders');
        const data = await res.json();
        return data;
    } catch (err) {
        console.error(err);
    }
}

// Parallel — don't await in sequence when independent
const [orders, products] = await Promise.all([
    getOrders(),
    getProducts()
]);
```

---

## TypeScript Generics

```typescript
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}

// Constrained generics
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

// Generic interfaces
interface ApiResponse<T> {
    data: T;
    success: boolean;
    errors: string[];
}
```

---

## Useful Type Utilities

```typescript
type Partial<T>  // all props optional
type Required<T> // all props required
type Pick<T, K>  // subset of props
type Omit<T, K>  // all except K
type Record<K, V> // object type with keys K and values V

// Practical
type CreateOrderRequest = Omit<Order, 'id' | 'createdAt'>;
type OrderSummary = Pick<Order, 'id' | 'status' | 'total'>;
```

---

## Currying

Transform a function that takes multiple args into a chain of single-arg functions.

```typescript
// Normal
const add = (a: number, b: number) => a + b;
add(2, 3); // 5

// Curried
const curriedAdd = (a: number) => (b: number) => a + b;
const add5 = curriedAdd(5);
add5(3);   // 8
add5(10);  // 15
```

Useful for partial application — pre-filling arguments.

---

## Hoisting

`var` declarations are hoisted to the top of their scope (but not their value). `let`/`const` are hoisted but not initialized (temporal dead zone).

```typescript
console.log(x); // undefined (hoisted, not initialized)
var x = 5;

console.log(y); // ReferenceError: Cannot access before initialization
let y = 5;

// Functions are fully hoisted (declaration + body)
greet(); // works
function greet() { console.log('hello'); }

// Function expressions are NOT
sayHi(); // TypeError: sayHi is not a function
var sayHi = () => console.log('hi');
```

---

## Connected To

- `my-stack/react.md` — closures are everywhere in hooks (useCallback, useEffect dependencies)
- `junior/algorithms.md` — memoize pattern is a closure
