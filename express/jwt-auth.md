# JWT Authentication
_Added: 2026-06-05_

---

## Mental Model
> JWT is a signed note the server gives you after login. Every subsequent request, you show that note. The server doesn't need to remember you — it just verifies the signature on the note.

No session stored on server. The token IS the proof.

---

## The Structure of a JWT

```
eyJhbG…header . eyJzdW…payload . signature
```

Three parts, separated by dots, all base64 encoded — **not encrypted**.
Anyone can decode header and payload. The signature is what makes it tamper-proof.

```js
// Decode payload without any library
const payload = JSON.parse(atob(token.split('.')[1]));
// { userId: '123', iat: 1234567890, exp: 1234571490 }
```

> Never put passwords or sensitive data in JWT payload. It's readable by anyone.

---

## The Basic Flow

```
1. User logs in with email + password
2. Server verifies credentials
3. Server signs a JWT with a secret key → sends it back
4. Client stores the token
5. Every request: client sends token in Authorization header
6. Server verifies signature → reads payload → knows who you are
```

---

## The Code

### Sign a token (on login)
```js
const jwt = require('jsonwebtoken');

const token = jwt.sign(
  { userId: user._id, role: user.role }, // payload
  process.env.JWT_SECRET,                // secret key
  { expiresIn: '15m' }                   // short lived
);

res.json({ token });
```

### Verify a token (auth middleware)
```js
function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;

  // Format: "Bearer <token>"
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ message: 'No token provided' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // attach to request for downstream use
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Invalid or expired token' });
  }
}

// Use on protected routes
app.get('/profile', authMiddleware, (req, res) => {
  res.json({ userId: req.user.userId });
});
```

---

## The Real Problem — Expiry vs Security

| Access token expiry | Problem |
|---|---|
| 7 days | Stolen token valid for 7 days |
| 15 minutes | User logged out every 15 minutes |

**Solution: Refresh Token pattern.**

---

## Refresh Token Pattern

Two tokens:

| | Access Token | Refresh Token |
|---|---|---|
| Expiry | Short (15min) | Long (7 days) |
| Stored | Memory / JS variable | httpOnly cookie |
| Used for | Every API request | Only to get new access token |
| If stolen | Expires in 15min | Can't be read by JS (httpOnly) |

### The Flow
```
Login
  → server sends access token (15min) + refresh token (7d, httpOnly cookie)

Access token expires
  → client hits /auth/refresh
  → server reads refresh token from cookie
  → verifies it + checks DB
  → issues new access token

Logout
  → server deletes refresh token from DB
  → client clears access token from memory
```

### Login — issue both tokens
```js
app.post('/auth/login', async (req, res) => {
  const user = await verifyCredentials(req.body);

  const accessToken = jwt.sign(
    { userId: user._id },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { userId: user._id },
    process.env.REFRESH_SECRET,
    { expiresIn: '7d' }
  );

  // Store in DB so we can invalidate on logout
  await user.updateOne({ refreshToken });

  // httpOnly cookie — JS cannot read this
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,     // blocks JS access (document.cookie can't see it)
    secure: true,       // HTTPS only
    sameSite: 'strict', // CSRF protection
    maxAge: 7 * 24 * 60 * 60 * 1000
  });

  res.json({ accessToken });
});
```

### Refresh — issue new access token
```js
app.post('/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.status(401).json({ message: 'No refresh token' });

  try {
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_SECRET);

    // Verify it's still in DB (not logged out)
    const user = await User.findOne({ _id: decoded.userId, refreshToken });
    if (!user) return res.status(401).json({ message: 'Invalid refresh token' });

    const accessToken = jwt.sign(
      { userId: user._id },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );

    res.json({ accessToken });
  } catch (err) {
    return res.status(401).json({ message: 'Expired refresh token, login again' });
  }
});
```

### Logout — invalidate refresh token
```js
app.post('/auth/logout', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  await User.updateOne({ refreshToken }, { refreshToken: null });
  res.clearCookie('refreshToken');
  res.json({ message: 'Logged out' });
});
```

---

## Cookie Flags — What They Actually Do

| Flag | What it does |
|---|---|
| `httpOnly` | Blocks JS from reading cookie (`document.cookie` can't see it). Protects against XSS. |
| `secure` | Cookie only sent over HTTPS, never plain HTTP |
| `sameSite: 'strict'` | Cookie not sent on cross-site requests. Protects against CSRF. |

> Common mistake: thinking httpOnly means HTTP only. It doesn't. It means HTTP API only — no JS access.

---

## Where to Store Access Token on Client

| Storage | XSS safe | Survives refresh |
|---|---|---|
| localStorage | No — JS can read it | Yes |
| sessionStorage | No — JS can read it | No |
| Memory (JS var) | Yes | No — use refresh token to restore |
| httpOnly cookie | Yes | Yes — but needs CSRF protection |

> Best practice: access token in memory, refresh token in httpOnly cookie.

---

## Gotchas

### Gotcha #1 — JWT cannot be invalidated before expiry
```js
// Once issued, a JWT is valid until it expires.
// You can't "revoke" it — the server has no state.
// Solutions:
// 1. Short expiry (15min) — stolen token expires fast
// 2. Token blacklist in Redis — check every request
// 3. Refresh token in DB — deleting it blocks new access tokens
```

### Gotcha #2 — jwt.verify throws, it doesn't return false
```js
// WRONG — bad token throws, doesn't return falsy
const decoded = jwt.verify(token, secret);
if (!decoded) return res.status(401)... // never reached

// CORRECT
try {
  const decoded = jwt.verify(token, secret);
} catch (err) {
  // err.name = 'TokenExpiredError' | 'JsonWebTokenError' | 'NotBeforeError'
  return res.status(401).json({ message: err.message });
}
```

### Gotcha #3 — never put sensitive data in payload
```js
// BAD
jwt.sign({ userId, password, creditCard }, secret);
// payload is base64 encoded — anyone can decode it

// GOOD — only put what you need for authorization
jwt.sign({ userId, role }, secret);
```

---

## Interview Answer
> "JWT is stateless auth. After login, the server signs a token with the user's ID and role and sends it back. The client attaches it to every request in the Authorization header. The server verifies the signature — no DB lookup needed. The problem with long-lived tokens is security, short-lived ones hurt UX. The solution is the refresh token pattern — a short-lived access token stored in memory, and a long-lived refresh token in an httpOnly cookie that JS can't read. When the access token expires, the client silently hits a refresh endpoint to get a new one. On logout, the refresh token is deleted from DB so it can't be reused. The key cookie flags are: httpOnly blocks JS access protecting against XSS, secure ensures HTTPS only, and sameSite strict prevents CSRF."
