# Rate Limiting
_Added: 2026-06-05_

---

## Mental Model
> Rate limiting is a bouncer at a club. You get X entries per hour. Try to come in more than that — you wait outside. Protects the server from spam, abuse, brute force, and DDoS.

---

## Why It Matters
- Brute force login attacks (try 1M passwords)
- One user hammering 10,000 req/sec
- Scraping your entire database
- One bad actor taking down the server for everyone

---

## The Three Algorithms

### 1. Fixed Window
```
Divide time into fixed buckets (e.g. every 60s).
Count requests in current bucket.
If count > limit → reject. Reset when bucket resets.
```
**Problem — boundary burst:**
```
Allow 100 req/min.
User sends 100 at 11:59 → allowed.
User sends 100 at 12:00 → allowed (new window resets).
200 requests in 2 seconds. Limit bypassed.
```

### 2. Sliding Window
```
Look at the last 60s from NOW (rolling).
Count requests in that window.
No boundary burst problem. More accurate.
```

### 3. Token Bucket (most common in production)
```
Bucket holds N tokens.
Each request consumes 1 token.
Tokens refill at fixed rate (e.g. 10/sec).
Bucket empty → reject.

Allows bursts, but sustained rate controlled by refill speed.
Used by AWS, Stripe, most real APIs.
```

---

## Basic Implementation
```js
const rateLimit = require('express-rate-limit');

// Global — all routes
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: { error: 'Too many requests, slow down.' },
  standardHeaders: true,
  legacyHeaders: false,
});
app.use(globalLimiter);

// Stricter — auth routes only
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 login attempts per 15 min
  message: { error: 'Too many login attempts.' },
});
app.post('/auth/login', authLimiter, loginHandler);
```

---

## The Multi-Server Problem
In-memory rate limiting breaks when you scale:
```
Server 1 (memory: user A = 99 requests)
Server 2 (memory: user A = 0 requests)  ← load balancer sends here

User A hits Server 2 → counter says 0 → allowed
Limit completely bypassed.
```

**Fix: Redis — all servers share one counter.**

---

## Redis-Backed Rate Limiting
```js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args),
  }),
});

app.use(limiter);
```

---

## Sliding Window in Redis (how it works under the hood)
```js
async function isRateLimited(userId, limit, windowMs) {
  const now = Date.now();
  const windowStart = now - windowMs;
  const key = `rate:${userId}`;

  await redisClient.zRemRangeByScore(key, 0, windowStart); // remove old requests
  const count = await redisClient.zCard(key);              // count in window

  if (count >= limit) return true;

  await redisClient.zAdd(key, { score: now, value: `${now}` }); // log request
  await redisClient.expire(key, Math.ceil(windowMs / 1000));
  return false;
}
```

---

## Response Headers to Send
```
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1623456789   ← unix timestamp when limit resets
Retry-After: 60               ← seconds until they can retry
```

---

## Gotchas

### Rate limit by user ID, not just IP
```js
// Bad — shared IPs (offices, NAT) penalize everyone
keyGenerator: (req) => req.ip

// Better — use user ID when authenticated
keyGenerator: (req) => req.user?.id || req.ip
```

### Different limits per tier
```js
const limiter = rateLimit({
  max: (req) => req.user?.isPro ? 1000 : 100,
});
```

### 429 not 403
```
403 = you don't have permission, ever
429 = you're fine, just slow down
```

### Don't rate limit health checks
```js
// Health check BEFORE limiters — load balancers ping this constantly
app.get('/health', (req, res) => res.send('ok'));
app.use(globalLimiter); // after
```

---

## Interview Answer
> "Rate limiting controls how many requests a client can make in a given time window. The three main algorithms are fixed window (simple but has boundary burst problem), sliding window (accurate, rolling window), and token bucket (allows bursts, refills at fixed rate — most production systems use this). In Express, express-rate-limit handles the basics, but in-memory storage breaks with multiple servers. The fix is a Redis store — all servers share the same counter. Rate limit by user ID when authenticated, not just IP. Different tiers get different limits. Always return 429 with Retry-After headers so clients know when to back off."

---

## TODO — System Design Deep Dive (Saturday 2026-06-06)
Design a rate limiter for a public API:
- 10,000 req/sec across 20 servers
- Free tier: 100 req/min
- Pro tier: 1000 req/min
