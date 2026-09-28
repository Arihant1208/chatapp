# L0 — Architecture

## The whole system

```mermaid
flowchart LR
  subgraph Browser
    UI[Minimal web UI<br/>conversation list · thread · composer]
  end
  UI -->|HTTPS / JSON| APP
  subgraph "One Node process (Fastify)"
    APP[HTTP layer<br/>routes + validation]
    APP --> AUTH[auth / session]
    APP --> SVC[services<br/>conversations · messages]
    SVC --> REPO[repositories<br/>SQL via pg pool]
    SVC --> REPLY[reply provider<br/>stub | hosted model]
  end
  REPO -->|pooled SQL| PG[(PostgreSQL)]
```

Everything runs in **one process**. The database is the only other moving part. That is the entire
point of L0 — minimal surface area, maximal correctness.

## Layers inside the process

We keep clean internal seams so L1+ can pull pieces out without a rewrite.

| Layer | Responsibility | Rule |
|---|---|---|
| **HTTP / routes** | parse, validate (`zod`), map to services, serialize responses | no business logic, no SQL |
| **Auth** | verify session cookie → attach `req.user` | one middleware, used everywhere |
| **Services** | business rules (create conversation, append message + reply) | owns transactions |
| **Repositories** | SQL only, via the shared pool | no HTTP concepts, no business rules |
| **Reply provider** | turn a prompt into an assistant reply | behind an interface, swappable |

> The **reply provider** is an interface (`generateReply(messages) -> string`) with a stub
> implementation (echo / canned) for L0. Swapping in a hosted model API is a config change, and the
> real streaming GPU path arrives at L2/L6 without touching the rest.

## Request flow — sending a message

```
POST /api/conversations/:id/messages   { content }
  → auth middleware: cookie → user
  → validate body (zod)
  → messageService.send(user, convId, content):
        BEGIN
          assert user owns conversation
          seq = nextSeq(convId)                 # atomic, see data-model
          insert user message (seq)
          reply = replyProvider.generate(history)
          seq2 = nextSeq(convId)
          insert assistant message (seq2, author='assistant')
        COMMIT
  → 201 { userMessage, assistantMessage }
```

At L0 the reply is produced **synchronously** inside the request (fine for a stub or a fast hosted
call). L2 makes this a stream; L6 makes it a queued GPU job. The service boundary is where that change
will land.

## Folder structure (target)

```
L0-monolith/
  docker-compose.yml         # Postgres for local dev
  .env.example
  package.json
  migrations/                # node-pg-migrate SQL files
  public/                    # minimal static UI (index.html + app.js)
  src/
    index.ts                 # boot: config, pool, fastify, routes, graceful shutdown
    config.ts                # env parsing + validation (zod)
    db/
      pool.ts                # the pg Pool (see 03-database-and-connections)
      migrate.ts             # migration runner entrypoint
    http/
      routes.auth.ts
      routes.conversations.ts
      routes.messages.ts
      middleware.auth.ts
    services/
      conversation.service.ts
      message.service.ts
      seq.ts                 # nextSeq()
    repos/
      user.repo.ts
      conversation.repo.ts
      message.repo.ts
    reply/
      provider.ts            # interface
      stub.provider.ts       # L0 default
    telemetry/
      logger.ts              # pino
      metrics.ts             # prom-client, /metrics
```

## Non-functional targets at L0

- p99 API latency `< 200ms` at ~100 req/s on a single modest box.
- Zero message loss across a process restart.
- Clean startup/shutdown (drain in-flight, close pool).
- Structured logs + `/metrics` + `/health` from day one (habits that pay off at every later level).
