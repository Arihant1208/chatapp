# L1 — Postgres Scaling: replica, read/write split, PgBouncer

Chat is read-heavy. Before any sharding (that's L5), two cheap wins: serve reads from a **replica**, and
put **PgBouncer** in front so many stateless instances can share Postgres.

---

## Two pools: write and read

```ts
// src/db/pool.ts
export const writePool = new Pool({ connectionString: config.DATABASE_URL,      max: config.PG_POOL_MAX });
export const readPool  = new Pool({ connectionString: config.DATABASE_REPLICA_URL, max: config.PG_POOL_MAX });
```

Routing rule:
- **Writes / mutations** → `writePool` (primary).
- **Reads** → `readPool` (replica) — **unless** read-your-writes applies (below).
- Transactions always use the primary (they contain writes).

```ts
// helper
export const readQuery  = (t:string,p?:unknown[]) => run(readPool,  t, p);
export const writeQuery = (t:string,p?:unknown[]) => run(writePool, t, p);
```

## Replication lag & read-your-writes

Replicas are **asynchronously** behind the primary — usually milliseconds, sometimes seconds under load.
A user who just sent a message could read a replica that hasn't received it yet and see it "missing".

**Fix — read-your-writes window:** after a user performs a write, mark them for a few seconds; while
marked, route *their* reads to the **primary** (or serve from the cache we just invalidated/refreshed).

```ts
// after any write by the user:
await redis.set(`ryw:${userId}`, '1', 'EX', 5);      // 5s window (tune to your lag)
// in read routing:
const pinToPrimary = await redis.get(`ryw:${userId}`);
const rows = await run(pinToPrimary ? writePool : readPool, sql, params);
```

Alternatives (know they exist): synchronous replication (higher write latency), or querying the replica's
LSN to confirm it caught up (more complex). The RYW window is the pragmatic default.

## PgBouncer (connection multiplexing)

Each stateless instance has its own pool; N instances × pool.max can blow past Postgres
`max_connections`. **PgBouncer** sits between the app and Postgres and multiplexes many client
connections onto few server connections.

- Mode: **transaction pooling** (a server connection is assigned only for the duration of a transaction) —
  best concurrency for short web queries.
- **Caveat:** transaction pooling disallows session-level features (session-scoped `SET`, some prepared
  statement caching, `LISTEN/NOTIFY`). With `pg`, avoid relying on session state; if needed, disable
  client-side prepared-statement caching or route those rare ops to a direct connection.
- The app connects to PgBouncer's port instead of Postgres directly; the replica can sit behind its own
  PgBouncer too.

```
app instances ──► PgBouncer (txn pooling) ──► Postgres primary
              └─► PgBouncer ─────────────────► read replica
```

## Config
```bash
DATABASE_URL=postgres://chat:chat@pgbouncer:6432/chat            # primary via bouncer
DATABASE_REPLICA_URL=postgres://chat:chat@pgbouncer-ro:6432/chat # replica via bouncer
PG_POOL_MAX=10
```

## Local dev shortcut
Running a real streaming replica locally is heavy. For dev you can point `DATABASE_REPLICA_URL` at the
**same** primary (so the code paths exist and are exercised) and stand up a true replica only when you
load-test lag behavior. Document that this is a dev-only shortcut.

## `/ready` now checks everything
```ts
await Promise.all([ writeQuery('SELECT 1'), readQuery('SELECT 1'), redis.ping() ]);
```

## Acceptance
- Reads are served from the replica (verify via `application_name` in `pg_stat_activity` or query logs).
- Read throughput rises when the replica is added (measure before/after with k6).
- A user never sees their own just-sent message missing (RYW works, even with induced lag).
- N app instances stay well under Postgres `max_connections` thanks to PgBouncer (check the connection count).
