# JS Mastery — Arun's Holy Grail
_Built by Boron | Started: 2026-06-05 | Updated: 2026-06-05_
_Architecture: concept → mental model → gotchas → interview answer → code_

> This file is the only JS reference you'll ever need.
> Each topic = enough to recall everything + not get trapped in interviews.

---

## TABLE OF CONTENTS
1. [Event Loop](#1-event-loop)
2. [Closures](#2-closures)
3. [var / let / const + Hoisting + TDZ](#3-var--let--const--hoisting--tdz)
4. [this keyword](#4-this-keyword)
5. [Prototypal Inheritance](#5-prototypal-inheritance)
6. [Promises + Async/Await](#6-promises--asyncawait)
7. [Debounce vs Throttle](#7-debounce-vs-throttle)
8. [Array Methods](#8-array-methods)
9. [Destructuring + Spread + Rest](#9-destructuring--spread--rest)
10. [Memoization](#10-memoization)

_More added after each study session._

---

## 1. Event Loop

### Mental Model
> JS is single-threaded. One chef, one kitchen. The Event Loop is the system that manages the order tickets (callbacks) so the chef never stands around waiting.

### The Architecture
```
Your Code (V8 engine)
      │
  Call Stack          ← what's currently executing
      │
  Event Loop          ← checks queues when stack is empty
      │
  ┌───┴────────────────────────────────────┐
  │  MICROTASK QUEUE (highest priority)    │
  │  1. process.nextTick()                 │
  │  2. Promise.then() / catch / finally   │
  └───┬────────────────────────────────────┘
      │
  ┌───┴────────────────────────────────────┐
  │  MACROTASK QUEUE (phases, in order)    │
  │  1. timers       → setTimeout          │
  │  2. pending I/O  → I/O error callbacks │
  │  3. poll         → new I/O events      │  ← Node spends most time here
  │  4. check        → setImmediate        │
  │  5. close        → socket.on('close')  │
  └────────────────────────────────────────┘
      │
  libuv thread pool   ← actual async work (file I/O, DNS, crypto)
```

### The Golden Rule
> After EVERY phase completes, the microtask queue drains FULLY before moving to the next phase.
> Microtasks always cut in between phases.

### Priority Order (memorize this)
```
1. process.nextTick()    ← highest
2. Promise.then()
3. setTimeout(fn, 0)
4. setImmediate()        ← lowest (but beats setTimeout inside I/O callbacks)
```

### The Classic Question
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
process.nextTick(() => console.log('4'));
console.log('5');

// Output: 1, 5, 4, 3, 2
// Why:
// 1, 5 → sync, runs immediately
// 4 → nextTick (highest microtask priority)
// 3 → Promise (second microtask priority)
// 2 → setTimeout (macrotask, runs last)
```

### Gotcha #1 — Nested nextTick
```js
process.nextTick(() => {
  process.nextTick(() => console.log('inner'));
  console.log('outer');
});
Promise.resolve().then(() => console.log('promise'));

// Output: outer, inner, promise
// WHY: nextTick queue drains COMPLETELY including anything added during drain.
// inner was added while draining, so it still runs before Promise.
```

### Gotcha #2 — setImmediate vs setTimeout (non-deterministic at top level)
```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Output: UNPREDICTABLE at top level (OS timing uncertainty)
```

### Gotcha #3 — setImmediate ALWAYS wins inside I/O callback
```js
fs.readFile('file', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Output: ALWAYS "immediate" then "timeout"
// Why: inside I/O callback = we're in poll phase.
// Next stop is check phase (setImmediate). Timers phase already passed.
```

### Gotcha #4 — nextTick starvation (DANGER in production)
```js
function runForever() {
  process.nextTick(runForever); // NEVER stop
}
runForever();
// Promises never resolve. I/O never fires. App freezes.
// nextTick keeps draining itself, event loop never moves forward.
// Use setImmediate if you need recursive scheduling.
```

### Interview Answer (say this out loud)
> "JavaScript is single-threaded, so it uses the Event Loop to handle async operations without blocking. When you make an async call — like a file read or setTimeout — it gets handed off to libuv or the OS. The JS thread moves on. When the async work completes, the callback goes into a queue. The Event Loop runs in phases: timers, poll where I/O callbacks land, check for setImmediate, and close callbacks. Between every phase, it drains the microtask queue — process.nextTick first, then Promises. So microtasks always run before the next phase starts. That's why Promises resolve before setTimeout even with zero delay."

---

## 2. Closures

### Mental Model
> A closure is a function that carries its backpack. Even after the outer function is gone, the inner function remembers everything in that backpack (outer scope).

### The Core
```js
function makeCounter() {
  let count = 0;              // this lives in the backpack
  return function() {
    return ++count;           // can always access count
  };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2
// makeCounter() is done — but count is still alive because counter holds a reference
```

### Gotcha #1 — var in loops (the classic trap)
```js
// BAD — all print 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// WHY: var is function-scoped. All 3 callbacks share the SAME i.
// By the time they run, the loop is done and i = 3.

// FIX 1: use let (block-scoped, new i per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}

// FIX 2: IIFE to capture i
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i); // 0, 1, 2
}
```

### Practical Uses
```js
// 1. Private variables (module pattern)
function createBank() {
  let balance = 0;                    // private
  return {
    deposit: (n) => balance += n,
    withdraw: (n) => balance -= n,
    getBalance: () => balance,
  };
}

// 2. Memoization
function memoize(fn) {
  const cache = {};                   // private cache
  return function(n) {
    return cache[n] ?? (cache[n] = fn(n));
  };
}

// 3. Partial application / currying
function multiply(a) {
  return function(b) {
    return a * b;                     // a lives in closure
  };
}
const double = multiply(2);
double(5); // 10
```

### Interview Answer
> "A closure is when a function retains access to its outer scope even after the outer function has returned. In JS, every function creates a closure. The practical uses are: private variables via the module pattern, memoization with a cached scope, and partial application / currying. The classic gotcha is var in loops — all callbacks share the same var reference, so you fix it with let or an IIFE."

---

## 3. var / let / const + Hoisting + TDZ

### The Table
| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (initialized to `undefined`) | Yes (NOT initialized — TDZ) | Yes (NOT initialized — TDZ) |
| Re-declare | Yes | No | No |
| Re-assign | Yes | Yes | No (binding, not value) |
| On window | Yes (`window.x`) | No | No |

### Hoisting — What It Really Means
> Hoisting = the JS engine moves declarations to the top of their scope BEFORE execution. Only declarations, not assignments.

```js
console.log(x); // undefined (not ReferenceError)
var x = 5;
// JS sees it as:
// var x;           ← declaration hoisted
// console.log(x);  ← x is undefined here
// x = 5;           ← assignment stays

// Function declarations are fully hoisted (body + all)
greet(); // works!
function greet() { return 'hi'; }

// Function expressions are NOT fully hoisted
greet(); // TypeError: greet is not a function
var greet = function() { return 'hi'; };
```

### TDZ — Temporal Dead Zone
> let/const are hoisted but NOT initialized. The time between hoisting and initialization = TDZ. Accessing during TDZ = ReferenceError.

```js
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// const must be initialized at declaration
const y; // SyntaxError: Missing initializer

// const binding can't change — but the VALUE can mutate
const arr = [1, 2, 3];
arr.push(4);       // fine — mutating the array
arr = [1, 2, 3, 4]; // TypeError — can't reassign the binding
```

### Gotcha — var leaks out of blocks
```js
if (true) {
  var x = 10;  // leaks to function scope
  let y = 20;  // stays in block
}
console.log(x); // 10
console.log(y); // ReferenceError
```

### Interview Answer
> "var is function-scoped and hoisted with an initial value of undefined. let and const are block-scoped and hoisted but enter the Temporal Dead Zone — accessing them before declaration throws a ReferenceError. const prevents reassignment of the binding, but doesn't make the value immutable — you can still mutate an object or array. Use const by default, let when you need to reassign, never var in modern code."

---

## 4. this Keyword

### The Rule
> `this` is not about where a function is defined. It's about HOW it is called.

### The 4 Rules (in priority order)
```js
// 1. new binding — this = new object
function Person(name) { this.name = name; }
const p = new Person('Arun'); // this = new Person object

// 2. Explicit binding — call/apply/bind
function greet() { return this.name; }
greet.call({ name: 'Arun' });   // 'Arun'
greet.apply({ name: 'Arun' });  // 'Arun' (args as array)
const bound = greet.bind({ name: 'Arun' }); bound(); // 'Arun'

// 3. Implicit binding — method call, this = object before the dot
const obj = {
  name: 'Arun',
  greet() { return this.name; }
};
obj.greet(); // 'Arun'

// 4. Default binding — standalone call
function greet() { return this; }
greet(); // window (browser) or global (Node) — undefined in strict mode
```

### Arrow Functions — Lexical this
> Arrow functions don't have their own this. They inherit it from where they were DEFINED (lexical scope).

```js
const obj = {
  name: 'Arun',
  greet: function() {
    const inner = () => this.name; // arrow inherits this from greet's context
    return inner();
  },
  greetArrow: () => this.name, // this = outer scope (window/undefined) — NOT obj
};
obj.greet();      // 'Arun'
obj.greetArrow(); // undefined
```

### Gotcha — losing this
```js
const obj = {
  name: 'Arun',
  greet() { return this.name; }
};
const fn = obj.greet;
fn(); // undefined — detached from obj, default binding applies

// Fix: bind
const fn = obj.greet.bind(obj);
fn(); // 'Arun'

// Fix: arrow in class (common React pattern)
class Comp {
  handleClick = () => { console.log(this); } // arrow = always bound to instance
}
```

### Interview Answer
> "this depends on how a function is called, not where it's defined. There are 4 rules in priority order: new binding creates a fresh object, explicit binding via call/apply/bind sets it manually, implicit binding uses the object before the dot, and default binding falls back to global or undefined in strict mode. Arrow functions don't have their own this — they inherit it from their enclosing scope at definition time. That's why React class components use arrow functions for event handlers, to avoid losing this when the method is passed as a callback."

---

## 5. Prototypal Inheritance

### Mental Model
> Every object in JS has a hidden link to another object called its prototype. If a property isn't found on the object, JS walks up the chain until it finds it or hits null.

```
dog → Dog.prototype → Animal.prototype → Object.prototype → null
```

### The Chain
```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  speak() { return `${this.name} barks`; }
}

const dog = new Dog('Rex');
dog.speak();         // 'Rex barks' (found on Dog.prototype)
dog.hasOwnProperty; // found on Object.prototype (walked the chain)

// Check the chain
Object.getPrototypeOf(dog) === Dog.prototype;     // true
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true
dog instanceof Dog;    // true
dog instanceof Animal; // true
```

### Gotcha — class is just syntax sugar
```js
// These are identical:
class Dog extends Animal {}

// Under the hood:
function Dog(name) { Animal.call(this, name); }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
```

### Gotcha — own vs inherited properties
```js
const dog = new Dog('Rex');
dog.hasOwnProperty('name');  // true — set in constructor
dog.hasOwnProperty('speak'); // false — lives on Dog.prototype
```

### Interview Answer
> "JavaScript uses prototypal inheritance — every object has a prototype chain. When you access a property, JS looks on the object first, then walks up the chain through prototypes until it finds it or hits null. ES6 classes are syntactic sugar over this prototype system. The extends keyword sets up the prototype chain, and super() calls the parent constructor. It's important to know the difference between own properties (on the object itself) and inherited properties (on the prototype)."

---

## 6. Promises + Async/Await

### Mental Model
> A Promise is a placeholder for a value that doesn't exist yet. It can be pending, fulfilled, or rejected. async/await is just cleaner syntax over Promises — under the hood it's the same thing.

### Promise Basics
```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve('done'), 1000);
});

p.then(val => console.log(val))  // 'done'
 .catch(err => console.error(err))
 .finally(() => console.log('always runs'));
```

### Async/Await
```js
async function getData() {
  try {
    const res = await fetch(url);   // pauses here, doesn't block thread
    const data = await res.json();
    return data;
  } catch (err) {
    console.error(err);
  }
}
```

### Parallel vs Sequential (the big gotcha)
```js
// SEQUENTIAL — slow (waits for each one)
const a = await fetchA();  // 1s
const b = await fetchB();  // 1s after A
// total: 2s

// PARALLEL — fast (both fire at once)
const [a, b] = await Promise.all([fetchA(), fetchB()]);
// total: 1s (whichever is slower)
```

### The Promise Combinators
```js
// All must succeed — rejects if ANY rejects
await Promise.all([p1, p2, p3]);

// Resolves/rejects with FIRST settled (win or fail)
await Promise.race([p1, p2, p3]);

// Waits for all, never rejects, gives status of each
await Promise.allSettled([p1, p2, p3]);
// returns: [{ status: 'fulfilled', value: ... }, { status: 'rejected', reason: ... }]

// Resolves with FIRST fulfilled (ignores rejections unless ALL reject)
await Promise.any([p1, p2, p3]);
```

### Gotcha — async functions always return a Promise
```js
async function greet() { return 'hi'; }
greet(); // Promise { 'hi' } — not 'hi'
greet().then(console.log); // 'hi'
```

### Gotcha — await in a forEach doesn't work
```js
// WRONG — forEach doesn't await
items.forEach(async (item) => {
  await processItem(item); // fires all at once, not sequential
});

// RIGHT — use for...of for sequential
for (const item of items) {
  await processItem(item);
}

// RIGHT — use Promise.all for parallel
await Promise.all(items.map(item => processItem(item)));
```

### Interview Answer
> "A Promise represents a value that may not be available yet. It has three states: pending, fulfilled, or rejected. async/await is syntactic sugar over Promises — an async function always returns a Promise, and await pauses execution inside the function without blocking the thread. The key gotcha is parallelism: using await in sequence is slow. Use Promise.all to fire requests in parallel. Another trap is using async/await inside forEach — it doesn't work as expected because forEach ignores returned Promises. Use for...of for sequential or Promise.all with map for parallel."

---

## 7. Debounce vs Throttle

### Mental Model
> **Debounce:** "Wait until the person stops talking, then respond."
> **Throttle:** "Respond at most once every X seconds, no matter how much they talk."

### Implementation
```js
// Debounce — fires after X ms of silence
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle — fires at most once per X ms
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```

### When to Use Which
| Use Case | Use |
|---|---|
| Search input / autocomplete | Debounce |
| Window resize handler | Debounce |
| Form validation on type | Debounce |
| Scroll event | Throttle |
| mousemove / drag | Throttle |
| Button spam prevention | Throttle |
| API rate limiting | Throttle |

### Interview Answer
> "Debounce delays the function call until after a period of inactivity — useful for search inputs where you don't want to fire an API call on every keystroke. Throttle limits how often a function can fire — useful for scroll or mousemove events where you want to run code at most once per frame. Both are closure-based — they return a new function that manages a timer internally."

---

## 8. Array Methods

### The Big Six (must know cold)
```js
const nums = [1, 2, 3, 4, 5];

// map — transform each item, returns new array (same length)
nums.map(x => x * 2);                  // [2, 4, 6, 8, 10]

// filter — keep items that pass test, returns new array
nums.filter(x => x % 2 === 0);         // [2, 4]

// reduce — accumulate into single value
nums.reduce((acc, x) => acc + x, 0);   // 15

// find — first item that passes (returns item or undefined)
nums.find(x => x > 3);                 // 4

// some — true if ANY item passes
nums.some(x => x > 4);                 // true

// every — true if ALL items pass
nums.every(x => x > 0);                // true
```

### Gotcha — sort mutates + sorts as strings by default
```js
[10, 1, 9, 2].sort();              // [1, 10, 2, 9] WRONG (string sort)
[10, 1, 9, 2].sort((a, b) => a - b); // [1, 2, 9, 10] correct ascending
[10, 1, 9, 2].sort((a, b) => b - a); // [10, 9, 2, 1] descending
```

### Gotcha — map/filter/reduce don't mutate original
```js
const original = [1, 2, 3];
const doubled = original.map(x => x * 2); // [2, 4, 6]
original; // still [1, 2, 3]
```

### Useful Others
```js
arr.flat(Infinity);             // flatten nested arrays fully
arr.flatMap(x => [x, x * 2]);  // map + flatten one level
arr.findIndex(x => x > 3);     // index of first match
arr.includes(3);                // boolean check
arr.indexOf(3);                 // first index or -1
[...new Set(arr)];              // deduplicate
arr.slice(1, 3);                // [idx1, idx3) — non-mutating
arr.splice(1, 2);               // removes 2 items from idx 1 — MUTATES
```

---

## 9. Destructuring + Spread + Rest

```js
// Array destructuring
const [a, b, ...rest] = [1, 2, 3, 4]; // a=1, b=2, rest=[3,4]

// Object destructuring with rename + default
const { name: n = 'anon', age = 0 } = user;

// Nested destructuring
const { address: { city } } = user;

// Spread — expand into individual elements
const arr2 = [...arr1, 4, 5];        // shallow copy + append
const obj2 = { ...obj1, key: 'val' }; // shallow merge

// Rest — collect remaining into array/object
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); }
const { a, ...rest } = obj; // rest has everything except a

// Gotcha — spread is SHALLOW
const original = { a: { b: 1 } };
const copy = { ...original };
copy.a.b = 99;
original.a.b; // 99 — same reference!

// Deep clone options
const deep = JSON.parse(JSON.stringify(obj)); // simple, loses functions/undefined
const deep2 = structuredClone(obj);           // modern, handles more types
```

---

## 10. Memoization

### Mental Model
> Cache the result of expensive function calls. If you've seen this input before, return the cached answer instead of computing again.

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage
const expensiveFn = memoize((n) => {
  // imagine this takes 2s
  return n * n;
});
expensiveFn(5); // computes: 25
expensiveFn(5); // cached: 25 instantly
```

### Gotcha — only pure functions (same input = same output)
```js
// Good — deterministic
const memoFib = memoize((n) => n <= 1 ? n : memoFib(n-1) + memoFib(n-2));

// Bad — side effects, depends on external state
const memoFetch = memoize(() => fetch('/api/data')); // stale data problem
```

### Interview Answer
> "Memoization is a performance optimization that caches the return value of a function based on its arguments. If the same arguments are passed again, it returns the cached result instead of recomputing. It's implemented using closures — the cache lives in the closure. Only works correctly for pure functions where the same input always produces the same output."

---

## Quick Recall Cheatsheet

```js
// Unique values
[...new Set(arr)]

// Swap
[a, b] = [b, a]

// Nullish coalescing (only null/undefined, NOT 0 or '')
val ?? 'default'

// Optional chaining
user?.address?.city

// To number
+str  /  Number(str)  /  parseInt(str, 10)

// To boolean
!!value

// Object iteration
Object.keys(obj)     // ['a', 'b']
Object.values(obj)   // [1, 2]
Object.entries(obj)  // [['a',1], ['b',2]]

// Shallow clone
{ ...obj }  /  Object.assign({}, obj)

// Deep clone
structuredClone(obj)

// Check type
typeof x          // 'string', 'number', 'boolean', 'undefined', 'object', 'function'
Array.isArray(x)  // better than typeof for arrays (typeof [] = 'object')
x instanceof Date // for class instances
```

---

_Next topics to add: Generators, WeakMap/WeakRef, Proxy/Reflect, Web Workers, Module system (CJS vs ESM), TypeScript generics_
