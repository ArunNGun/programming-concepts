# Express.js — Deep Dive
_Added: 2026-06-05_

## Topics
- [Middleware](./README.md)
- [JWT Authentication](./jwt-auth.md)

---

## Middleware

### Mental Model
> Middleware is a conveyor belt. Every request passes through stations in order. Each station can inspect it, modify it, stop it, or pass it along. `next()` moves it to the next station.

```
Request → [logger] → [auth] → [body parser] → [route handler] → Response
              ↓          ↓            ↓                ↓
           next()     next()       next()          res.send()
```

### The Signature
```js
function middleware(req, res, next) {
  // do something
  next(); // pass to next middleware
}
```

Three things middleware can do:
1. Call `next()` — pass to next middleware
2. Send a response — `res.send()` / `res.json()` — stops the chain
3. Call `next(err)` — skip to error handling middleware

### Types of Middleware
```js
// 1. Application-level — every request
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// 2. Route-level — specific route only
app.get('/dashboard', authMiddleware, (req, res) => {
  res.json({ data: 'protected' });
});

// 3. Built-in
app.use(express.json());           // parses JSON body → req.body
app.use(express.urlencoded());     // parses form data
app.use(express.static('public')); // serves static files

// 4. Third-party
app.use(cors());
app.use(helmet()); // security headers
```

### Order Matters — Critical
```js
// WRONG — handler registered before json parser
app.get('/users', handler);   // req.body = undefined here
app.use(express.json());      // too late

// CORRECT
app.use(express.json());      // first
app.get('/users', handler);   // now req.body is populated
```

> Middleware runs in the order it's registered. No exceptions.

### next() vs next(err) vs next('route')
| Call | What happens |
|---|---|
| `next()` | Passes to next regular middleware |
| `next(err)` | Skips ALL regular middleware, jumps to error handler |
| `next('route')` | Skips remaining handlers for current route only |

### Gotcha — next() doesn't exit the function
```js
app.use((req, res, next) => {
  console.log('middleware 1');
  next();
  console.log('after next'); // THIS STILL RUNS
});

// Output: middleware 1 → middleware 2 → after next

// BAD — causes "headers already sent" error
app.use((req, res, next) => {
  next();
  res.send('oops'); // runs after next(), response already sent
});

// GOOD — return to stop execution
app.use((req, res, next) => {
  return next();
});
```

### Gotcha — Async Errors Not Caught Automatically
```js
// BREAKS — unhandled async error
app.get('/users', async (req, res) => {
  const users = await db.find(); // throws — Express won't catch it
  res.json(users);
});

// Fix 1: try/catch + next(err)
app.get('/users', async (req, res, next) => {
  try {
    const users = await db.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// Fix 2: asyncHandler wrapper (cleaner)
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users', asyncHandler(async (req, res) => {
  const users = await db.find();
  res.json(users);
}));
// or use: npm install express-async-errors
```

### Error Handling Middleware — Must Have 4 Params
```js
// Must be LAST — after all routes
// Must have ALL 4 params — even if you don't use next
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    message: err.message || 'Internal Server Error'
  });
});
```

> If you write `(err, req, res)` with only 3 params, Express won't recognize it as an error handler.

### Full Production Pattern
```js
// Custom error class
class AppError extends Error {
  constructor(message, status) {
    super(message);
    this.status = status;
  }
}

// Route — throws custom error
app.get('/user/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError('User not found', 404);
  res.json(user);
}));

// Centralized error handler
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ message: err.message });
});
```

### Full Flow Visualization
```
Request
  │
  ├─ app.use(logger)           → next()
  ├─ app.use(express.json())   → next()
  ├─ app.use(authMiddleware)   → next()  OR  next(err) ──→ jumps to error handler
  ├─ app.get('/users', handler) → res.json()   ← chain ends here
  │
  └─ app.use((err,req,res,next) => ...)   ← error handler, always last
```

### Interview Answer
> "Middleware in Express is a function with req, res, and next. Every request passes through middleware in the order they're registered — order matters. next() passes control to the next middleware, next(err) skips all regular middleware and goes straight to the error handler. Error handling middleware is special — it takes 4 arguments starting with err, and must be registered last. The biggest gotcha is async errors — Express doesn't catch them automatically, so you either wrap handlers in try/catch and call next(err), or use an asyncHandler wrapper that catches Promise rejections automatically."
