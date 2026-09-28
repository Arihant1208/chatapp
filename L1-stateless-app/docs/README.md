# L1 — Stateless App + Real Data Tier · Implementation Docs

L1 evolves L0. The app itself gains almost no features — instead we make it **disposable**: any instance
can die or deploy with zero loss, reads scale via a cache + replica, and no state lives in the process.

Start from a copy of the working L0 app, then apply the changes in these docs.

Read in order:

1. [01-architecture.md](01-architecture.md) — what moves where; the stateless rule.
2. [02-state-externalization.md](02-state-externalization.md) — sessions → Redis, files → object storage.
3. [03-redis.md](03-redis.md) — the Redis client, cache-aside strategy, key schema, TTLs, failure policy.
4. [04-postgres-scaling.md](04-postgres-scaling.md) — read replica, read/write split, read-your-writes,
   PgBouncer.
5. [05-build-plan.md](05-build-plan.md) — ordered tasks, acceptance criteria, exit gate.

## The one rule of L1

**App servers hold no state.** No sessions in memory, no files on local disk, no sticky anything.
Everything that must survive a restart lives in Postgres, Redis, or object storage. If you can `kill -9`
any instance mid-traffic and lose nothing, L1 is working.

## What we add over L0

| Piece | Purpose |
|---|---|
| **Redis** (`ioredis`) | session store, cache-aside for hot reads, rate-limit counters |
| **Read replica** | serve reads off a replica; primary handles writes |
| **PgBouncer** | let many app instances share Postgres without exhausting connections |
| **Object storage** (S3/MinIO) | uploaded files via presigned URLs, not local disk |
| **Async propagation seam** | write DB → emit event → update cache (avoid the dual-write trap) |
