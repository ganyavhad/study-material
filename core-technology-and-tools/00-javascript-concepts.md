# JavaScript Concepts - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist

---

## 1. JavaScript Engine & Runtime

### 1.1 V8 Engine Internals

```
Source Code → Parser → AST → Ignition (Interpreter) → Bytecode
                                        ↓
                              TurboFan (Optimizing Compiler) → Machine Code
```

**Key Stages:**
- **Parser** — tokenizes and creates Abstract Syntax Tree (AST)
- **Ignition** — interprets AST into bytecode (fast startup)
- **TurboFan** — JIT-compiles hot functions into optimized machine code
- **Deoptimization** — falls back to bytecode if assumptions break (e.g., type changes)

### 1.2 Memory Management

```
Heap Memory
├── New Space (Young Generation) — short-lived objects, Scavenge GC
└── Old Space (Old Generation) — long-lived objects, Mark-Sweep-Compact GC
```

**Garbage Collection:**
- **Scavenge (Minor GC)** — copies live objects between semi-spaces in New Space
- **Mark-Sweep-Compact (Major GC)** — marks reachable objects, sweeps dead ones, compacts memory
- **Incremental Marking** — breaks marking into small steps to avoid long pauses

---

## 2. Data Types & Type Coercion

### 2.1 Primitive vs Reference Types

| Primitive (Stack) | Reference (Heap) |
|-------------------|-------------------|
| `string` | `Object` |
| `number` | `Array` |
| `boolean` | `Function` |
| `undefined` | `Date` |
| `null` | `RegExp` |
| `symbol` | `Map`, `Set` |
| `bigint` | `WeakMap`, `WeakSet` |

### 2.2 Type Coercion

```javascript
// Implicit coercion
'5' + 3        // '53' (string concatenation)
'5' - 3        // 2 (numeric subtraction)
true + 1       // 2
null + 5       // 5
undefined + 5  // NaN

// Falsy values
// false, 0, -0, 0n, '', null, undefined, NaN

// Equality pitfalls
0 == ''        // true (both coerced to 0)
0 == '0'       // true
'' == '0'      // false
null == undefined // true
NaN == NaN     // false
```

### 2.3 typeof & instanceof

```javascript
typeof 'hello'      // 'string'
typeof 42           // 'number'
typeof true         // 'boolean'
typeof undefined    // 'undefined'
typeof null         // 'object' (historical bug)
typeof Symbol()     // 'symbol'
typeof BigInt(10)   // 'bigint'
typeof {}           // 'object'
typeof []           // 'object'
typeof function(){} // 'function'

[] instanceof Array  // true
{} instanceof Object // true
```

---

## 3. Scope & Closures

### 3.1 Scope Types

```javascript
// Global Scope
var globalVar = 'accessible everywhere';

// Function Scope
function example() {
  var functionScoped = 'only inside function';
}

// Block Scope (let & const)
if (true) {
  let blockScoped = 'only inside block';
  const alsoBlockScoped = 'cannot be reassigned';
}

// Lexical Scope — inner function accesses outer variables
function outer() {
  const message = 'hello';
  function inner() {
    console.log(message); // 'hello' — lexical scope
  }
  inner();
}
```

### 3.2 var vs let vs const

| Feature | `var` | `let` | `const` |
|---------|-------|-------|---------|
| Scope | Function | Block | Block |
| Hoisting | Yes (initialized as `undefined`) | Yes (TDZ — not initialized) | Yes (TDZ — not initialized) |
| Re-declaration | Allowed | Not allowed | Not allowed |
| Re-assignment | Allowed | Allowed | Not allowed |

### 3.3 Closures

```javascript
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.getCount();  // 2
// count is enclosed — not accessible directly
```

**Common Use Cases:**
- Data privacy / encapsulation
- Factory functions
- Memoization
- Event handlers retaining state
- Module pattern (pre-ES6)

### 3.4 Temporal Dead Zone (TDZ)

```javascript
console.log(a); // undefined (var is hoisted)
var a = 10;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 20;
```

---

## 4. Execution Context & Hoisting

### 4.1 Execution Context

```
Global Execution Context
├── Creation Phase
│   ├── Create Global Object (window / global)
│   ├── Create `this` binding
│   ├── Allocate memory for variables (hoisting)
│   └── Functions stored entirely, var set to undefined
└── Execution Phase
    └── Code runs line by line
```

### 4.2 Call Stack

```javascript
function first() {
  console.log('first');
  second();
}
function second() {
  console.log('second');
  third();
}
function third() {
  console.log('third');
}
first();

// Call Stack:
// | third()  |
// | second() |
// | first()  |
// | global() |
```

### 4.3 Hoisting

```javascript
// Function declarations are fully hoisted
greet(); // 'Hello!'
function greet() { console.log('Hello!'); }

// Function expressions are NOT hoisted
sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log('Hi!'); };

// Arrow functions are NOT hoisted
hello(); // ReferenceError
const hello = () => console.log('Hello');
```

---

## 5. `this` Keyword

### 5.1 Binding Rules (Priority Order)

```javascript
// 1. new binding
function Person(name) { this.name = name; }
const p = new Person('Ganesh'); // this → new object

// 2. Explicit binding (call, apply, bind)
function greet() { console.log(this.name); }
greet.call({ name: 'Ganesh' });   // 'Ganesh'
greet.apply({ name: 'Ganesh' });  // 'Ganesh'
const bound = greet.bind({ name: 'Ganesh' });
bound(); // 'Ganesh'

// 3. Implicit binding (object method)
const obj = {
  name: 'Ganesh',
  greet() { console.log(this.name); }
};
obj.greet(); // 'Ganesh'

// 4. Default binding (global / undefined in strict mode)
function show() { console.log(this); }
show(); // window (browser) or global (Node.js)
```

### 5.2 Arrow Functions & `this`

```javascript
const obj = {
  name: 'Ganesh',
  // Arrow function does NOT have its own `this`
  // It inherits from the enclosing lexical scope
  greet: () => {
    console.log(this.name); // undefined — `this` is outer scope
  },
  delayedGreet() {
    setTimeout(() => {
      console.log(this.name); // 'Ganesh' — arrow inherits from delayedGreet
    }, 100);
  }
};
```

---

## 6. Prototypes & Inheritance

### 6.1 Prototype Chain

```
myObj → Object.prototype → null

myArr → Array.prototype → Object.prototype → null

myFunc → Function.prototype → Object.prototype → null
```

```javascript
const animal = {
  speak() { return `${this.name} makes a sound`; }
};

const dog = Object.create(animal);
dog.name = 'Rex';
dog.speak(); // 'Rex makes a sound'

// Checking prototype
Object.getPrototypeOf(dog) === animal; // true
dog.hasOwnProperty('name');   // true
dog.hasOwnProperty('speak');  // false (inherited)
```

### 6.2 ES6 Classes (Syntactic Sugar)

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }

  speak() {
    return `${this.name} barks`;
  }
}

const dog = new Dog('Rex', 'Labrador');
dog.speak();          // 'Rex barks'
dog instanceof Dog;   // true
dog instanceof Animal; // true
```

### 6.3 Static & Private Fields

```javascript
class Config {
  static instance = null;
  #secret; // private field

  constructor(secret) {
    this.#secret = secret;
  }

  static getInstance(secret) {
    if (!Config.instance) {
      Config.instance = new Config(secret);
    }
    return Config.instance;
  }

  getSecret() {
    return this.#secret;
  }
}
```

---

## 7. Asynchronous JavaScript

### 7.1 Event Loop & Task Queues

```
Call Stack        │  Web APIs / Node APIs
──────────────    │  ──────────────────
│ executing  │───>│  setTimeout, fetch,
│ sync code  │    │  DOM events, I/O
──────────────    │
      ↑           │
      │           ↓
      │    ┌─────────────────────┐
      │    │ Microtask Queue     │ ← Promise.then, queueMicrotask, MutationObserver
      │    └─────────────────────┘
      │    ┌─────────────────────┐
      └────│ Macrotask Queue     │ ← setTimeout, setInterval, I/O, setImmediate
           └─────────────────────┘
```

**Execution Order:** Call Stack → All Microtasks → One Macrotask → All Microtasks → ...

### 7.2 Promise Deep Dive

```javascript
// Promise states: pending → fulfilled | rejected

// Chaining
fetch('/api/user')
  .then(res => res.json())
  .then(user => fetch(`/api/posts/${user.id}`))
  .then(res => res.json())
  .catch(err => console.error(err))
  .finally(() => console.log('done'));

// Promise combinators
Promise.all([p1, p2, p3]);       // all must fulfill; rejects on first failure
Promise.allSettled([p1, p2, p3]); // waits for all; returns status of each
Promise.race([p1, p2, p3]);      // first to settle (fulfill or reject)
Promise.any([p1, p2, p3]);       // first to fulfill; ignores rejections
```

### 7.3 Async/Await Patterns

```javascript
// Sequential (slower — use when order matters)
async function sequential() {
  const user = await getUser();
  const posts = await getPosts(user.id);
  return { user, posts };
}

// Parallel (faster — use for independent operations)
async function parallel() {
  const [user, config] = await Promise.all([
    getUser(),
    getConfig()
  ]);
  return { user, config };
}

// Error handling
async function withErrorHandling() {
  try {
    const data = await riskyOperation();
    return data;
  } catch (error) {
    if (error instanceof NetworkError) {
      return fallbackData;
    }
    throw error; // re-throw unexpected errors
  }
}
```

### 7.4 Event Loop Output Questions

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// Output: 1, 4, 3, 2
// Sync first → Microtask (Promise) → Macrotask (setTimeout)
```

```javascript
async function foo() {
  console.log('A');
  await Promise.resolve();
  console.log('B');
}

console.log('C');
foo();
console.log('D');

// Output: C, A, D, B
// 'B' is a microtask scheduled after the await
```

---

## 8. Iterators & Generators

### 8.1 Iterables & Iterators

```javascript
// Custom iterable
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { done: true };
      }
    };
  }
};

for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

// Spread & destructuring work with iterables
const nums = [...range];      // [1, 2, 3, 4, 5]
const [a, b] = range;         // a=1, b=2
```

### 8.2 Generators

```javascript
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
fib.next(); // { value: 0, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 2, done: false }

// Practical: paginated API fetching
async function* fetchPages(url) {
  let page = 1;
  let hasMore = true;
  while (hasMore) {
    const res = await fetch(`${url}?page=${page}`);
    const data = await res.json();
    yield data.items;
    hasMore = data.hasNextPage;
    page++;
  }
}

for await (const items of fetchPages('/api/users')) {
  console.log(items);
}
```

---

## 9. ES6+ Features

### 9.1 Destructuring

```javascript
// Object destructuring
const { name, age, country = 'India' } = user;
const { address: { city } } = user; // nested
const { id, ...rest } = user;       // rest operator

// Array destructuring
const [first, , third] = [1, 2, 3]; // skip elements
const [head, ...tail] = [1, 2, 3];  // rest

// Function parameters
function createUser({ name, role = 'user' }) {
  return { name, role };
}
```

### 9.2 Spread Operator

```javascript
// Arrays
const merged = [...arr1, ...arr2];
const copy = [...original]; // shallow copy

// Objects
const updated = { ...user, name: 'Updated' }; // shallow merge
const merged = { ...defaults, ...overrides };
```

### 9.3 Optional Chaining & Nullish Coalescing

```javascript
// Optional chaining (?.) — returns undefined instead of throwing
const city = user?.address?.city;
const first = users?.[0];
const result = obj?.method?.();

// Nullish coalescing (??) — default only for null/undefined
const port = config.port ?? 3000;
// vs OR (||) which also defaults for 0, '', false
const port = config.port || 3000; // 0 would fallback to 3000
```

### 9.4 Template Literals & Tagged Templates

```javascript
// Template literals
const greeting = `Hello, ${name}! You are ${age} years old.`;

// Multi-line
const html = `
  <div>
    <h1>${title}</h1>
    <p>${body}</p>
  </div>
`;

// Tagged templates (used in libraries like styled-components)
function sql(strings, ...values) {
  // strings: ['SELECT * FROM users WHERE id = ', '']
  // values: [userId]
  return {
    text: strings.join('$'),
    values
  };
}

const query = sql`SELECT * FROM users WHERE id = ${userId}`;
```

### 9.5 Symbols

```javascript
const id = Symbol('id');
const user = { [id]: 123, name: 'Ganesh' };

user[id];       // 123
Object.keys(user); // ['name'] — symbols are not enumerable

// Well-known symbols
Symbol.iterator   // make objects iterable
Symbol.toPrimitive // custom type coercion
Symbol.hasInstance  // customize instanceof
```

### 9.6 Map, Set, WeakMap, WeakSet

```javascript
// Map — any key type, ordered, iterable
const map = new Map();
map.set({ id: 1 }, 'value');   // object as key
map.set('key', 'value');
map.get('key');   // 'value'
map.has('key');   // true
map.size;         // 2

// Set — unique values, ordered
const set = new Set([1, 2, 2, 3]);
set.size; // 3
set.add(4);
set.has(2); // true

// WeakMap — keys must be objects, allows GC
const cache = new WeakMap();
let obj = {};
cache.set(obj, 'metadata');
obj = null; // obj can be garbage collected

// WeakSet — objects only, allows GC
const visited = new WeakSet();
visited.add(node);
visited.has(node); // true
```

---

## 10. Functional Programming Patterns

### 10.1 Pure Functions & Immutability

```javascript
// Pure function — same input → same output, no side effects
const add = (a, b) => a + b;

// Immutable update
const addItem = (arr, item) => [...arr, item];
const updateUser = (user, updates) => ({ ...user, ...updates });
```

### 10.2 Higher-Order Functions

```javascript
// Functions that take or return functions
const withLogging = (fn) => (...args) => {
  console.log(`Calling with: ${args}`);
  const result = fn(...args);
  console.log(`Result: ${result}`);
  return result;
};

const multiply = (a, b) => a * b;
const loggedMultiply = withLogging(multiply);
loggedMultiply(3, 4); // logs and returns 12
```

### 10.3 Currying & Partial Application

```javascript
// Currying — transform f(a, b, c) into f(a)(b)(c)
const curry = (fn) => {
  const arity = fn.length;
  return function curried(...args) {
    if (args.length >= arity) return fn(...args);
    return (...more) => curried(...args, ...more);
  };
};

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);   // 6
add(1, 2)(3);   // 6

// Practical: reusable API fetcher
const fetchFrom = curry((baseUrl, endpoint, id) =>
  fetch(`${baseUrl}${endpoint}/${id}`).then(r => r.json())
);

const fetchUser = fetchFrom('https://api.example.com')('/users');
fetchUser(42); // fetches user 42
```

### 10.4 Composition

```javascript
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

const processUser = pipe(
  normalize,
  validate,
  save
);
```

### 10.5 Array Methods

```javascript
const users = [
  { name: 'Alice', age: 30 },
  { name: 'Bob', age: 25 },
  { name: 'Charlie', age: 35 },
];

// map — transform each element
const names = users.map(u => u.name);

// filter — keep elements matching condition
const seniors = users.filter(u => u.age >= 30);

// reduce — accumulate into single value
const totalAge = users.reduce((sum, u) => sum + u.age, 0);

// find / findIndex
const bob = users.find(u => u.name === 'Bob');

// some / every
const hasMinor = users.some(u => u.age < 18);
const allAdults = users.every(u => u.age >= 18);

// flat / flatMap
[[1, 2], [3, 4]].flat();             // [1, 2, 3, 4]
users.flatMap(u => u.tags || []);     // flatten nested arrays

// Chaining
const result = users
  .filter(u => u.age >= 30)
  .map(u => u.name)
  .sort();
```

---

## 11. Error Handling

### 11.1 Error Types

```javascript
// Built-in error types
new Error('generic error');
new TypeError('expected a string');
new RangeError('index out of bounds');
new ReferenceError('x is not defined');
new SyntaxError('unexpected token');
```

### 11.2 Custom Errors

```javascript
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.name = 'AppError';
    this.statusCode = statusCode;
    this.code = code;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

// Usage
throw new NotFoundError('User');
```

### 11.3 try/catch/finally

```javascript
try {
  const data = JSON.parse(input);
  processData(data);
} catch (error) {
  if (error instanceof SyntaxError) {
    console.error('Invalid JSON:', error.message);
  } else {
    throw error; // re-throw unexpected errors
  }
} finally {
  cleanup(); // always runs
}
```

---

## 12. Design Patterns in JavaScript

### 12.1 Module Pattern

```javascript
const UserModule = (() => {
  const _users = []; // private

  return {
    add(user) { _users.push(user); },
    getAll() { return [..._users]; },
    getCount() { return _users.length; },
  };
})();
```

### 12.2 Singleton

```javascript
class Database {
  static #instance;

  constructor(config) {
    if (Database.#instance) return Database.#instance;
    this.config = config;
    Database.#instance = this;
  }

  static getInstance(config) {
    return Database.#instance || new Database(config);
  }
}
```

### 12.3 Observer Pattern

```javascript
class EventEmitter {
  #listeners = new Map();

  on(event, callback) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, []);
    }
    this.#listeners.get(event).push(callback);
    return this;
  }

  emit(event, ...args) {
    const callbacks = this.#listeners.get(event) || [];
    callbacks.forEach(cb => cb(...args));
  }

  off(event, callback) {
    const callbacks = this.#listeners.get(event) || [];
    this.#listeners.set(event, callbacks.filter(cb => cb !== callback));
  }
}
```

### 12.4 Factory Pattern

```javascript
class NotificationFactory {
  static create(type, config) {
    switch (type) {
      case 'email': return new EmailNotification(config);
      case 'sms': return new SmsNotification(config);
      case 'push': return new PushNotification(config);
      default: throw new Error(`Unknown type: ${type}`);
    }
  }
}

const notification = NotificationFactory.create('email', { to: 'user@test.com' });
```

---

## 13. Proxy & Reflect

```javascript
const handler = {
  get(target, prop) {
    console.log(`Accessing ${prop}`);
    return Reflect.get(target, prop);
  },
  set(target, prop, value) {
    if (prop === 'age' && typeof value !== 'number') {
      throw new TypeError('Age must be a number');
    }
    return Reflect.set(target, prop, value);
  }
};

const user = new Proxy({}, handler);
user.name = 'Ganesh'; // logs: Accessing name (on read)
user.age = 'abc';     // throws TypeError
```

**Use Cases:**
- Validation
- Logging / debugging
- Reactive systems (Vue.js uses Proxy)
- Default values
- Access control

---

## 14. Modules (ESM vs CJS)

| Feature | CommonJS (CJS) | ES Modules (ESM) |
|---------|----------------|-------------------|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | Synchronous | Asynchronous |
| Evaluation | Runtime | Compile-time (static) |
| Top-level `this` | `module.exports` | `undefined` |
| Tree-shaking | Not supported | Supported |
| File extension | `.js` (default in Node) | `.mjs` or `"type": "module"` |
| Dynamic import | `require(path)` | `import(path)` returns Promise |

```javascript
// Dynamic imports (ESM)
const module = await import('./heavy-module.js');

// Conditional loading
if (needsChart) {
  const { Chart } = await import('./chart.js');
}
```

---

## 15. Web APIs & Browser Concepts

### 15.1 DOM Manipulation

```javascript
// Selecting elements
document.getElementById('app');
document.querySelector('.card');
document.querySelectorAll('li');

// Creating elements
const el = document.createElement('div');
el.textContent = 'Hello';
el.classList.add('active');
document.body.appendChild(el);

// Event delegation (efficient for lists)
document.querySelector('ul').addEventListener('click', (e) => {
  if (e.target.matches('li')) {
    console.log('Clicked:', e.target.textContent);
  }
});
```

### 15.2 Event Propagation

```
Capturing Phase (top → target)
       ↓
   Target Phase
       ↓
Bubbling Phase (target → top)
```

```javascript
// Bubbling (default)
element.addEventListener('click', handler);

// Capturing
element.addEventListener('click', handler, true);

// Stop propagation
element.addEventListener('click', (e) => {
  e.stopPropagation();  // stops bubbling/capturing
});

// Prevent default
link.addEventListener('click', (e) => {
  e.preventDefault();  // prevents navigation
});
```

### 15.3 Storage APIs

```javascript
// localStorage — persists across sessions
localStorage.setItem('theme', 'dark');
localStorage.getItem('theme'); // 'dark'
localStorage.removeItem('theme');

// sessionStorage — cleared when tab closes
sessionStorage.setItem('token', 'abc123');

// Cookies (via document.cookie)
document.cookie = 'name=Ganesh; max-age=3600; path=/; secure; samesite=strict';
```

| Feature | localStorage | sessionStorage | Cookies |
|---------|-------------|----------------|---------|
| Capacity | ~5-10 MB | ~5-10 MB | ~4 KB |
| Expiry | Manual | Tab close | Configurable |
| Sent to server | No | No | Yes (every request) |
| Access | Same origin | Same origin + tab | Configurable |

---

## 16. Common Interview Questions

### Q1: What is the difference between `==` and `===`?
- `==` performs **type coercion** before comparison
- `===` checks **value and type** without coercion
- Always use `===` to avoid unexpected behavior

### Q2: Explain closures with a practical example.
A closure is a function that remembers the variables from its outer scope even after the outer function has returned. Used for data privacy, memoization, and maintaining state in callbacks.

### Q3: What is the event loop?
The event loop continuously checks the call stack and task queues. When the stack is empty, it processes all microtasks (Promises), then one macrotask (setTimeout), and repeats.

### Q4: What is the difference between `null` and `undefined`?
- `undefined` — variable declared but not assigned; default return value
- `null` — explicitly assigned to indicate "no value"
- `typeof undefined === 'undefined'`, `typeof null === 'object'`

### Q5: Explain prototypal inheritance.
Every object has an internal `[[Prototype]]` link. When a property is accessed and not found on the object, JS walks up the prototype chain until it finds it or reaches `null`.

### Q6: What is debouncing vs throttling?

```javascript
// Debounce — execute after delay since last call (search input)
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle — execute at most once per interval (scroll/resize)
function throttle(fn, interval) {
  let lastTime = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn(...args);
    }
  };
}
```

### Q7: What is a memory leak? Common causes?
- Forgotten timers (`setInterval` without `clearInterval`)
- Detached DOM nodes still referenced
- Closures holding large objects
- Global variables
- Event listeners not removed

---

## 17. Quick Reference Cheat Sheet

```
Scope:         var → function    let/const → block
Hoisting:      var → undefined   let/const → TDZ
this:          arrow → lexical   function → dynamic
Equality:      == → coercion    === → strict
Falsy:         false, 0, '', null, undefined, NaN
Copy:          spread → shallow  structuredClone → deep
Async Order:   sync → microtask (Promise) → macrotask (setTimeout)
Modules:       CJS → sync/runtime     ESM → async/static
```

---

*Study Material — Ganesh Avhad | [ganeshavhad.com](https://ganeshavhad.com)*
