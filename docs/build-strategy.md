# Build Strategy — building every level as a real, runnable chat app

We are not just documenting the levels — we are **building each one as its own working chat
application**. This doc explains how the levels relate as code, the shared conventions, and how to run
them.

> High-level "why" for each level lives in [`../implementation-plan/`](../implementation-plan/).
> The concrete "how we build *this* app" lives in each level's own `docs/` folder.

---

## Repo layout

Each level is a **self-contained project folder** at the repo root, so you can build, run, and reason
about it in isolation:

```
/docs                     # cross-level strategy (this file)
/implementation-plan      # high-level plan per level (the why + acceptance gates)
/L0-monolith              # Level 0 app  ← start here
  /docs                   # L0 design docs (architecture, DB, API, build plan)
  /src ...                # code (added when we build)
/L1-stateless-app         # Level 1 app  (evolves L0)
  /docs
  /src ...
/L2-realtime ...          # later levels
```

**Why separate folders, not one evolving app?** So each level stays a clean teaching artifact you can
run and compare. L1 starts as a copy of L0 and then applies its specific changes (externalize state,
add Redis, add a read replica). We keep both so the *diff between levels* is the lesson.

## Shared conventions (all levels)

| Concern | Choice |
|---|---|
| Language / runtime | TypeScript on Node.js 20+ |
| Web framework | Fastify |
| DB driver | `pg` (node-postgres) with a connection pool |
| Migrations | `node-pg-migrate` (plain SQL up/down) |
| Cache / sessions (L1+) | Redis via `ioredis` |
| Validation | `zod` on every request body/params |
| Config | env vars only (12-factor); `.env` for local, never committed |
| Local infra | Docker Compose (Postgres, later Redis/MinIO) |
| Logging | `pino` (structured JSON) |
| Metrics | `prom-client` → `/metrics` (Prometheus format) |
| Tests | `vitest` + a smoke test per endpoint |

## Config & secrets

- Every service reads config from environment variables. A `.env.example` is committed; the real
  `.env` is git-ignored.
- No secrets in code. No credentials in the repo.

## How a level is "done"

A level is finished only when its `docs/05-build-plan.md` acceptance criteria pass **and** its load
test (from the level's plan) has been run to find the breaking point. Then we advance.

## Running any level (target experience)

```bash
cd L0-monolith
cp .env.example .env
docker compose up -d          # starts Postgres (and Redis/MinIO at L1+)
npm install
npm run migrate               # apply DB migrations
npm run dev                   # app on http://localhost:8080
```

## Order of work

1. **L0** — monolith: one process, Postgres, correct data model + API. ← we are here
2. **L1** — make it stateless: externalize sessions/files, add Redis cache, add a read replica.
3. L2+ — real-time, scale, fan-out, data-at-scale, inference, global (later).
