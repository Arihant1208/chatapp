# L0 — Monolith · Implementation Docs

The concrete design for building L0 as a real, runnable chat app: **one Node/TypeScript process
(Fastify) serving the API + a minimal web UI, backed by one PostgreSQL database.**

Read in order:

1. [01-architecture.md](01-architecture.md) — components, request flow, folder structure, tech.
2. [02-data-model.md](02-data-model.md) — schema, migrations, and the race-free `seq` allocation.
3. [03-database-and-connections.md](03-database-and-connections.md) — Postgres pool, config, timeouts,
   transactions, health, graceful shutdown.
4. [04-api-and-app.md](04-api-and-app.md) — endpoints, session auth, reply generation, the web UI.
5. [05-build-plan.md](05-build-plan.md) — ordered build tasks, milestones, acceptance criteria.

## What L0 is (and isn't)

- **Is:** a correct, complete, single-box chat app. Sign up, create conversations, send messages, get a
  reply, reload history in order. Locks in the data model + API contract every later level preserves.
- **Isn't:** scalable, highly available, or real-time. No streaming (L2), no second server (L3), no
  Redis (L1). Replies are a **stub/echo** or a single hosted-model call — real GPU serving is L6.

## Success in one sentence

Two browsers on the same account see each other's messages after refresh, restarting the process loses
nothing, and `GET messages?afterSeq=N` returns exactly the messages after N, in order, at p99 < 200ms.
