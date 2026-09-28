# L0 — Database & Connection Handling

How the app talks to PostgreSQL: one shared connection **pool**, sane timeouts, transactions for the
send path, a health check, and graceful shutdown. These choices are small at L0 but set the pattern
every later level reuses.

---

## Why a pool (not a connection per request)

Opening a Postgres connection is expensive (TCP + TLS + auth + backend process fork). A **pool** keeps a
small set of connections warm and hands them out per query. One pool per process. Postgres itself has a
hard ceiling (`max_connections`, ~100 by default), so the app's pool size must stay well under it — this
is exactly why L1 later adds PgBouncer.

## The pool

```ts
// src/db/pool.ts
import { Pool } from 'pg';
import { config } from '../config';

export const pool = new Pool({
  connectionString: config.DATABASE_URL,
  max: config.PG_POOL_MAX,              // e.g. 10 — keep well under Postgres max_connections
  min: 0,
  idleTimeoutMillis: 30_000,            // release idle clients
  connectionTimeoutMillis: 5_000,      // fail fast if the pool is exhausted
  // enforce query time limits at the connection level:
  statement_timeout: 10_000,           // server cancels any statement over 10s
  query_timeout: 10_000,               // client-side guard
  application_name: 'l0-monolith',     // shows up in pg_stat_activity — invaluable for debugging
});

pool.on('error', (err) => {
  // a client sitting idle in the pool errored (e.g. DB restarted). Log; the pool self-heals.
  logger.error({ err }, 'idle pg client error');
});
```

### Pool sizing rule of thumb
`pool.max` per instance × number of instances **must stay below** Postgres `max_connections` (minus a
reserve for admin). At L0 there's one instance, so 10 is plenty. Don't over-size — more connections than
CPU cores rarely helps and can hurt.

## Querying

A thin helper keeps call sites clean and gives one place to add logging/metrics:

```ts
// src/db/pool.ts (cont.)
export async function query<T>(text: string, params?: unknown[]): Promise<T[]> {
  const start = performance.now();
  const res = await pool.query(text, params);
  dbHistogram.observe((performance.now() - start) / 1000);  // Prometheus
  return res.rows as T[];
}
```

- **Always parameterized** (`$1, $2`). Never string-concatenate SQL — that's SQL injection.
- Repositories use `query()`; services use repositories.

## Transactions (the send path needs one)

Allocating `seq` and inserting the message(s) must be atomic. Use a `withTx` helper that checks out one
client, runs your function, and commits/rolls back:

```ts
// src/db/tx.ts
export async function withTx<T>(fn: (c: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const out = await fn(client);
    await client.query('COMMIT');
    return out;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();               // ALWAYS return the client to the pool
  }
}
```

```ts
// message.service.ts (core of send)
return withTx(async (c) => {
  const { rows:[conv] } = await c.query(
    `UPDATE conversations SET next_seq = next_seq + 1, updated_at = now()
      WHERE id = $1 AND owner_id = $2
      RETURNING next_seq - 1 AS seq`, [convId, userId]);
  if (!conv) throw new HttpError(404, 'conversation not found');

  const userMsg = await insertMessage(c, convId, conv.seq, 'user', content);
  const reply   = await replyProvider.generate(await recentHistory(c, convId));
  const { rows:[c2] } = await c.query(
    `UPDATE conversations SET next_seq = next_seq + 1 WHERE id = $1 RETURNING next_seq - 1 AS seq`,
    [convId]);
  const botMsg  = await insertMessage(c, convId, c2.seq, 'assistant', reply);
  return { userMsg, botMsg };
});
```

> **Keep transactions short.** Holding a transaction open across a slow external call locks the row and
> ties up a pooled connection. The L0 stub reply is instant, so this is fine — but note this is exactly
> why L2 moves reply generation **out** of the transaction and streams it separately.

## Config & env

```bash
# .env.example
DATABASE_URL=postgres://chat:chat@localhost:5432/chat
PG_POOL_MAX=10
PORT=8080
SESSION_SECRET=change-me-32-bytes-min
NODE_ENV=development
```

Config is parsed and validated once at boot (fail fast on a missing/invalid var):

```ts
// src/config.ts
export const config = z.object({
  DATABASE_URL: z.string().url(),
  PG_POOL_MAX: z.coerce.number().int().positive().default(10),
  PORT: z.coerce.number().default(8080),
  SESSION_SECRET: z.string().min(32),
  NODE_ENV: z.enum(['development','test','production']).default('development'),
}).parse(process.env);
```

## Health & readiness

```ts
app.get('/health', async () => ({ ok: true }));           // process is up
app.get('/ready',  async (_req, reply) => {                // can it serve?
  try { await query('SELECT 1'); reply.send({ ready: true }); }
  catch { reply.code(503).send({ ready: false }); }
});
```

`/health` (liveness) and `/ready` (can reach the DB) are trivial now but become how the load balancer
and Kubernetes decide routing at L3.

## Graceful shutdown

On `SIGTERM`/`SIGINT`: stop accepting new requests, let in-flight ones finish, then close the pool.

```ts
async function shutdown() {
  await app.close();          // Fastify stops accepting, drains in-flight
  await pool.end();           // close all pooled connections cleanly
  process.exit(0);
}
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

## Local dev database

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: chat
      POSTGRES_PASSWORD: chat
      POSTGRES_DB: chat
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chat"]
      interval: 5s
      timeout: 3s
      retries: 10
volumes: { pgdata: {} }
```

## Failure modes to handle even at L0

| Failure | Behavior |
|---|---|
| Pool exhausted | `connectionTimeoutMillis` rejects fast → 503, not a hang |
| Slow query | `statement_timeout` cancels it server-side → error, not a stuck connection |
| DB restarted | idle clients error and are evicted; next query reconnects |
| Client not released | **bug** — always `release()` in `finally`, or the pool leaks and deadlocks |

## What changes at L1

- Add a **second pool** for a read replica (`readPool`) and route reads to it.
- Put **PgBouncer** in front so many app instances can share Postgres without exhausting connections.
- Move **sessions** out of Postgres into Redis.
See [`../../L1-stateless-app/docs/04-postgres-scaling.md`](../../L1-stateless-app/docs/04-postgres-scaling.md).
