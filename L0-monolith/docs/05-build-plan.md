# L0 — Build Plan

Ordered tasks to build L0. Each milestone is independently runnable and testable.

## Milestone A — Skeleton boots
- [ ] `npm init`, TypeScript, `tsconfig`, `vitest`, scripts (`dev`, `build`, `migrate`, `test`).
- [ ] `docker-compose.yml` with Postgres 16 + healthcheck.
- [ ] `.env.example` + `src/config.ts` (zod-validated env).
- [ ] `src/index.ts`: Fastify boots, `/health` returns `{ok:true}`, graceful shutdown wired.
- [ ] `pino` logger + `prom-client` `/metrics`.
- **Done when:** `npm run dev` serves `/health` and `/metrics`.

## Milestone B — Database layer
- [ ] `src/db/pool.ts` (pool + `query()` helper + timeouts), `src/db/tx.ts` (`withTx`).
- [ ] `migrations/001_init.sql` (users, conversations, messages, sessions).
- [ ] `npm run migrate` applies cleanly; `/ready` checks `SELECT 1`.
- **Done when:** migrations apply and `/ready` is 200 with DB up, 503 with DB down.

## Milestone C — Auth
- [ ] `argon2` password hashing; `user.repo.ts`.
- [ ] signup / login / logout / me routes; `sessions` rows; `sid` cookie (HttpOnly/Secure/SameSite).
- [ ] `requireUser` middleware.
- **Done when:** signup→login→me round-trips; wrong password 401; logout clears session.

## Milestone D — Conversations & messages
- [ ] repos: `conversation.repo.ts`, `message.repo.ts`.
- [ ] `seq.ts` atomic allocation via `UPDATE ... RETURNING`.
- [ ] conversation CRUD routes (owner-scoped).
- [ ] `message.service.send` in a transaction: user msg → reply → assistant msg.
- [ ] `GET messages?afterSeq=&limit=` ascending by seq.
- [ ] `reply/stub.provider.ts` wired via `REPLY_PROVIDER`.
- **Done when:** the full send flow returns both messages; history reloads in order.

## Milestone E — Web UI
- [ ] `public/index.html` + `app.js`: login/signup, conversation list, thread, composer.
- **Done when:** you can hold a full conversation in the browser.

## Milestone F — Verify & measure
- [ ] `vitest` smoke test per endpoint (happy path + auth failure).
- [ ] `loadtests/l0.js` (k6): CRUD + send mix; ramp to find where p99 crosses 200ms.
- **Done when:** acceptance criteria below all pass and the L0 ceiling RPS is recorded.

## Acceptance criteria (the exit gate)
- [ ] Two browsers, same account: a sent message appears in the other **after refresh**.
- [ ] Restarting the process loses **no** persisted messages.
- [ ] `GET messages?afterSeq=N` returns exactly the messages after N, in order.
- [ ] Concurrent sends to one conversation never produce duplicate `seq` (test with parallel requests).
- [ ] p99 API latency `< 200ms` at ~100 req/s on the target box (from the k6 run).

## ▶ Exit trigger → L1
Advance when **either**: you can't accept downtime for deploys/restarts, **or** the single box is the
bottleneck. Then continue in [`../../L1-stateless-app/docs`](../../L1-stateless-app/docs).

## Explicitly out of scope at L0 (don't build yet)
- Streaming replies (L2) · multiple app servers / load balancer (L3) · Redis (L1) · read replicas (L1)
- Real model inference (L6) · rate limiting beyond a trivial guard · file uploads (add at L1 via object storage)
