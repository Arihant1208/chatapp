# L0 — Data Model

Get this right and it barely changes through L7. The one non-obvious decision — the per-conversation
**sequence number** — is explained at the end; it is the backbone of ordering, sync, and pagination
for every later level.

## Schema

```sql
-- migrations/001_init.sql

CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()

CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,             -- argon2id hash; never the raw password
  display_name  TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversations (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title      TEXT,
  next_seq   BIGINT NOT NULL DEFAULT 1,    -- atomic per-conversation counter (see below)
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX conversations_owner_idx ON conversations (owner_id, updated_at DESC);

CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  seq             BIGINT NOT NULL,          -- monotonic within the conversation
  author          TEXT   NOT NULL CHECK (author IN ('user','assistant')),
  content         TEXT   NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (conversation_id, seq)             -- ordering + "everything after seq N" + dedupe
);
CREATE INDEX messages_conv_seq_idx ON messages (conversation_id, seq);

-- sessions live in the DB at L0 (moved to Redis at L1)
CREATE TABLE sessions (
  id         TEXT PRIMARY KEY,             -- random 256-bit token id
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX sessions_user_idx ON sessions (user_id);
```

## Entities

- **users** — identity + argon2id password hash. (OAuth can be added later; the shape doesn't change.)
- **conversations** — owned by a user; carries `next_seq`, the counter that hands out sequence numbers.
- **messages** — belong to a conversation, ordered by `seq`, authored by `user` or `assistant`.
- **sessions** — server-side session rows at L0 (a signed cookie holds only the session id).

## Migrations

- Tool: `node-pg-migrate`. Files in `migrations/`, applied by `npm run migrate`.
- Every migration is **backward-compatible** (expand-then-contract) from the start — a habit that becomes
  mandatory at L3+ where you can't stop the app to change schema.
- `npm run migrate` runs on deploy/boot in dev; in prod it's a separate, gated step.

## The `seq` decision (read this)

**Problem:** wall-clock `created_at` is not a safe order key — clocks skew, and two messages can share a
timestamp. We need a stable, gap-tolerant, per-conversation order.

**Solution:** a per-conversation monotonic counter, allocated **atomically** so concurrent sends can't
collide:

```sql
-- inside the same transaction as the INSERT:
UPDATE conversations
   SET next_seq = next_seq + 1, updated_at = now()
 WHERE id = $1
RETURNING next_seq - 1 AS seq;
```

The `UPDATE ... RETURNING` takes a row lock on that conversation, so two simultaneous sends serialize and
each gets a distinct `seq` — no `MAX(seq)+1` race, no gaps that matter.

**Why it pays off later:**
- **Ordering** independent of clocks.
- **Sync / pagination:** `GET messages?afterSeq=N` is the multi-device catch-up primitive (L1+), the
  offline-delivery primitive (L4), and the cursor for infinite scroll.
- **Dedupe / idempotency anchor** for retries once streaming and at-least-once delivery arrive (L2/L4).

> Add it now even though L0 doesn't strictly need concurrency safety — retrofitting ordering after you
> have data is painful.

## Client-generated message ids (idempotency, forward-looking)

Accept an optional `clientMsgId` (UUID) on send and store it (add a nullable unique column later). It
lets a retried request be recognized instead of creating a duplicate — cheap to honor at L0, essential
by L2/L4.
