# L1 — Redis: client, caching, keys, failure policy

Redis does three jobs at L1: **session store** (see 02), **cache-aside** for hot reads, and **rate-limit
counters**. It is an accelerator layered over Postgres — never the source of truth.

---

## The client

Use `ioredis`. One shared client per process (it pipelines/multiplexes over a single connection; you do
**not** need a pool for normal commands). Add a **second** client only when you introduce blocking ops or
pub/sub (that's L4, not now).

```ts
// src/cache/redis.ts
import Redis from 'ioredis';
import { config } from '../config';

export const redis = new Redis(config.REDIS_URL, {
  maxRetriesPerRequest: 2,          // fail a command fast rather than hang the request
  enableReadyCheck: true,
  connectTimeout: 3_000,
  retryStrategy: (times) => Math.min(times * 200, 2_000),  // reconnect backoff
  lazyConnect: false,
});

redis.on('error', (err) => logger.warn({ err }, 'redis error'));  // warn, not crash — we fail open
```

Graceful shutdown: `await redis.quit();` alongside closing the pg pools.

### Env
```bash
REDIS_URL=redis://localhost:6379
CACHE_TTL_SECONDS=60
SESSION_TTL_SECONDS=2592000        # 30 days
```

## Cache-aside (the pattern we use everywhere)

**Read:** try cache → on miss read DB → populate cache with a TTL → return.
**Write:** write DB (source of truth) → **invalidate** (or refresh) the affected keys.

```ts
// generic helper
export async function cached<T>(key: string, ttl: number, load: () => Promise<T>): Promise<T> {
  try {
    const hit = await redis.get(key);
    if (hit) { cacheHits.inc(); return JSON.parse(hit) as T; }
  } catch { /* fail open: fall through to DB */ }
  cacheMisses.inc();
  const val = await load();
  try { await redis.set(key, JSON.stringify(val), 'EX', ttl); } catch { /* ignore cache write errors */ }
  return val;
}
```

> **Every Redis access is wrapped in try/catch and falls back to Postgres.** A cache outage must degrade
> to "slower but correct", never to an error. This is the single most important rule in this doc.

## What we cache (and how we invalidate)

| Data | Key | TTL | Invalidate on |
|---|---|---|---|
| Conversation list for a user | `user:{uid}:convs` | 60s | create/delete/rename a conversation, or new message (updates `updated_at`) |
| Recent messages (tail) of a conversation | `conv:{cid}:tail` | 60s | new message in that conversation |
| User profile | `user:{uid}:profile` | 300s | profile update |

- **Tail cache detail:** cache the last N (e.g. 50) messages as a JSON array. On a new message, the
  simplest correct move is `DEL conv:{cid}:tail` (invalidate); a later optimization is to push onto a
  cached list. Start with invalidate — it's obviously correct.
- **Pagination beyond the tail** (older history via `afterSeq`) is **not** cached — it's rare and cheap
  to hit the replica.

## Avoiding the dual-write trap

Writing to Postgres **and** Redis in the same request is a mini distributed transaction: if the second
write fails, they disagree. Our policy:
- **DB is written first and is authoritative.** The cache is only ever **invalidated** (a `DEL`), not
  relied upon to hold the new value. A failed `DEL` just means a stale entry that expires within the TTL —
  self-healing, not corrupting.
- This keeps correctness even when Redis misbehaves, and sets up the L4 pattern (write DB → emit event →
  consumers update caches/indexes) cleanly.

## Rate-limit counters (basic, per-user)

A simple fixed-window counter to catch abuse early (full algorithms come at L3+):

```ts
// allow N sends per 60s window
const k = `rl:send:${userId}:${Math.floor(Date.now()/60000)}`;
const n = await redis.incr(k);
if (n === 1) await redis.expire(k, 60);
if (n > LIMIT) throw new HttpError(429, 'rate limited');
```
On a Redis error here we **fail open** (allow) — a cache outage shouldn't block all sends.

## Local dev
```yaml
# add to docker-compose.yml
  redis:
    image: redis:7
    command: ["redis-server","--appendonly","yes"]   # AOF persistence
    ports: ["6379:6379"]
```

## Acceptance for the Redis layer
- Cache hit ratio on conversation reads **> 80%** under normal browsing (watch `cacheHits/cacheMisses`).
- Killing Redis mid-traffic: requests keep succeeding (slower); no 5xx from the cache path; sessions
  survive via the DB fallback.
- After a write, a subsequent read never returns stale data for the writer (invalidate + read-your-writes).
