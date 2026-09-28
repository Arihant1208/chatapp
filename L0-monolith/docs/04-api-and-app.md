# L0 — API, Auth & the Web UI

## API contract

All endpoints are JSON under `/api`. Auth is a session cookie. Every body/param is validated with `zod`.

### Auth

| Method | Path | Body | Returns |
|---|---|---|---|
| `POST` | `/api/auth/signup` | `{ email, password, displayName? }` | `201` + sets `sid` cookie |
| `POST` | `/api/auth/login` | `{ email, password }` | `200` + sets `sid` cookie |
| `POST` | `/api/auth/logout` | — | `204`, clears cookie |
| `GET`  | `/api/auth/me` | — | `{ id, email, displayName }` |

### Conversations

| Method | Path | Body | Returns |
|---|---|---|---|
| `POST` | `/api/conversations` | `{ title? }` | `201 { id, title, createdAt }` |
| `GET`  | `/api/conversations` | — | `[{ id, title, updatedAt }]` (owner's, newest first) |
| `GET`  | `/api/conversations/:id` | — | `{ id, title, createdAt }` |
| `DELETE` | `/api/conversations/:id` | — | `204` |

### Messages

| Method | Path | Query / Body | Returns |
|---|---|---|---|
| `GET`  | `/api/conversations/:id/messages` | `?afterSeq=N&limit=50` | `[{ id, seq, author, content, createdAt }]` ascending |
| `POST` | `/api/conversations/:id/messages` | `{ content, clientMsgId? }` | `201 { userMessage, assistantMessage }` |

- **Ordering & paging** are always by `seq`. `afterSeq` is the cursor (default 0). `limit` capped at 100.
- **Authorization:** every conversation/message route checks `conversation.owner_id === req.user.id`
  (returns 404, not 403, to avoid leaking existence).

### Error shape

```json
{ "error": { "code": "not_found", "message": "conversation not found" } }
```
HTTP status carries the class (400 validation, 401 unauth, 404 not found, 409 conflict, 500 server).

## Auth design (L0)

- **Password hashing:** `argon2id` (via `argon2`). Never store or log raw passwords.
- **Session:** on login, create a `sessions` row with a random 256-bit id and `expires_at` (e.g. 30 days).
  Set it as an **`HttpOnly`, `Secure`, `SameSite=Lax`** cookie named `sid`. The cookie holds only the id;
  the server looks up the row.
- **Middleware:** `requireUser` reads `sid`, loads a non-expired session → `req.user`, else `401`.
- **Why DB sessions here:** simple and revocable (delete the row). L1 moves them to Redis for speed and
  to keep the app stateless across many instances.

```ts
// middleware.auth.ts (essence)
const sid = req.cookies.sid;
const [s] = await query('SELECT user_id FROM sessions WHERE id=$1 AND expires_at > now()', [sid]);
if (!s) return reply.code(401).send({ error:{ code:'unauthorized', message:'login required' }});
req.user = await userRepo.byId(s.user_id);
```

## Reply provider

```ts
// reply/provider.ts
export interface ReplyProvider {
  generate(history: {author:string; content:string}[]): Promise<string>;
}
// reply/stub.provider.ts  — L0 default
export const stubProvider: ReplyProvider = {
  async generate(history) {
    const last = history.at(-1)?.content ?? '';
    return `You said: "${last}". (stub reply — real model comes at L6)`;
  },
};
```

Config `REPLY_PROVIDER=stub|openai|anthropic` selects the implementation. A hosted-model provider is a
few lines (call the API, return text). Streaming that response is the L2 upgrade; the interface stays.

## The web UI (minimal, on purpose)

A single static page in `public/` — no framework needed at L0:

- **Left:** conversation list (`GET /api/conversations`), "New chat" button.
- **Right:** message thread (`GET .../messages`) + a composer that `POST`s and appends both returned
  messages.
- Auth: a simple login/signup form; on success the cookie is set and the app loads.

> We keep the UI deliberately thin so the lesson stays on the backend. When L2 introduces streaming,
> the UI gains an `EventSource`/`fetch`-stream reader — the first place the frontend gets interesting.

## Observability from day one

- `pino` structured logs with a per-request id.
- `prom-client` default metrics + a request-duration histogram + the DB-query histogram → `GET /metrics`.
- These make the L0 acceptance test ("p99 < 200ms at 100 rps") measurable rather than a guess.
