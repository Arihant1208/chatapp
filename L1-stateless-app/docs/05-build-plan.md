# L1 — Build Plan

L1 starts from a **copy of the working L0 app** and makes it stateless + read-scalable. Build in this order.

## Milestone A — Fork L0
- [ ] Copy `L0-monolith` → `L1-stateless-app` (code + docker-compose + migrations).
- [ ] Confirm it still boots and passes the L0 smoke tests unchanged.

## Milestone B — Redis in
- [ ] Add Redis to `docker-compose.yml`; add `REDIS_URL`, `CACHE_TTL_SECONDS`, `SESSION_TTL_SECONDS` to config.
- [ ] `src/cache/redis.ts` (ioredis client + reconnect + graceful quit).
- [ ] `cached()` helper with **fail-open** try/catch and hit/miss metrics.
- **Done when:** `/ready` pings Redis; killing Redis doesn't 5xx the read paths.

## Milestone C — Sessions → Redis (+ DB fallback)
- [ ] Move session create/verify/revoke to Redis (`sess:{id}`, sliding `EXPIRE`).
- [ ] Keep the Postgres `sessions` row as a durable fallback; re-warm Redis on a miss.
- **Done when:** restart the app → still logged in; flush Redis → still logged in (via DB fallback); logout revokes immediately.

## Milestone D — Cache-aside on hot reads
- [ ] Cache `user:{uid}:convs` and `conv:{cid}:tail`; invalidate (`DEL`) on the relevant writes.
- [ ] Metrics for hit ratio.
- **Done when:** cache hit ratio > 80% under a browsing load; a write is never followed by a stale read.

## Milestone E — Read replica + read/write split
- [ ] Add `DATABASE_REPLICA_URL`, `readPool`/`writePool`, `readQuery`/`writeQuery`.
- [ ] Route reads → replica, writes → primary; transactions → primary.
- [ ] Read-your-writes window via `ryw:{userId}` in Redis.
- **Done when:** reads hit the replica; induced lag never shows a writer stale data.

## Milestone F — PgBouncer
- [ ] Add PgBouncer (transaction pooling) to compose; point `DATABASE_URL` at it.
- [ ] Verify session-pooling caveats don't bite (prepared-statement handling).
- **Done when:** two app instances stay well under Postgres `max_connections`.

## Milestone G — Object storage
- [ ] Add MinIO to compose; S3 config; presigned PUT/GET; `/api/uploads` + `/complete`.
- **Done when:** a file uploads directly to storage and its key persists; the app never streams bytes.

## Milestone H — Prove statelessness & measure
- [ ] Run **two** instances on different ports; hit each — either serves any user.
- [ ] `kill -9` one mid-traffic → zero data loss, user just retries.
- [ ] Do a zero-downtime swap: start new instance, stop old → no failed requests.
- [ ] `loadtests/l1.js` (heavier read mix): assert p99 < 200ms, watch cache hit ratio, replica lag, primary write CPU; find the **write ceiling**.

## Acceptance criteria (exit gate)
- [ ] Kill any instance mid-traffic → **zero** lost messages.
- [ ] Zero-downtime deploy (start new, drain old) with **no** error spike.
- [ ] Cache hit ratio on conversation reads **> 80%**.
- [ ] Replication lag never shows a user their own just-sent message as missing.
- [ ] Read throughput scales when the replica is added (measured).

## ▶ Exit trigger → L2
Advance when you need the server to **push** — stream an LLM answer token-by-token, or deliver a message
the instant it exists rather than on refresh.

## Out of scope at L1
- Load balancer + many servers as a managed fleet (L3) · WebSockets/SSE (L2) · pub/sub fan-out (L4)
- Sharding the message store (L5) · real model inference (L6)
