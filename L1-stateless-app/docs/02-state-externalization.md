# L1 — State Externalization

Two pieces of per-box state remain after L0: **sessions** and **uploaded files**. We move both out so the
app process holds nothing irreplaceable.

---

## Sessions → Redis (with a DB safety net)

### Primary path
- On login, generate a random 256-bit session id, store the session in Redis:
  `SET sess:{id} <json{userId, createdAt}> EX <ttlSeconds>`.
- The `sid` cookie (HttpOnly, Secure, SameSite=Lax) carries only the id.
- `requireUser` does one Redis `GET sess:{id}` — sub-ms, no DB hit on the hot path.
- **Sliding expiry:** on each authenticated request, `EXPIRE sess:{id} <ttl>` to extend active sessions.

```ts
// auth: create
await redis.set(`sess:${id}`, JSON.stringify({ userId, createdAt: Date.now() }), 'EX', TTL);
// auth: verify (middleware)
const raw = await redis.get(`sess:${id}`);
if (!raw) return unauthorized();
await redis.expire(`sess:${id}`, TTL);          // sliding window
req.user = await loadUser(JSON.parse(raw).userId);
```

### Why keep a DB fallback
A Redis outage must not log every user out. Options, pick one and document it:
- **Chosen:** keep the L0 `sessions` table as a **cold fallback** — write the session to both Redis
  (hot) and Postgres (durable) at login; on a Redis miss, check Postgres, and re-warm Redis. Slightly
  more work per login, but a Redis restart is invisible to users.
- Alternative: Redis-only with persistence (AOF) and accept that a full Redis loss requires re-login.

### Revocation
- Logout / "sign out everywhere": `DEL sess:{id}` (and delete the DB fallback row). Because sessions are
  server-side, revocation is immediate — unlike stateless JWTs (which we deliberately don't use yet).

> When we reach APIs/mobile at higher levels we may add short-lived JWT **access** tokens + a revocable
> **refresh** token in Redis. For a cookie-based web app, server-side sessions are simpler and safer.

## Uploaded files → Object storage (S3 / MinIO)

Even though L0 didn't put files on disk, L1 is where uploads become real (avatars, later chat
attachments). Never stream large files through the app.

### Presigned-URL flow
```
1. client: POST /api/uploads  { filename, contentType }
2. app:    authorize, generate an object key  uploads/{userId}/{uuid}
           return a presigned PUT url (expires ~60s)
3. client: PUT the bytes DIRECTLY to object storage using that url
4. client: POST /api/uploads/complete { key }   → app records the key in DB
5. reads:  app returns a presigned GET url (short TTL) for private objects
```
- The app only ever handles small JSON — the bytes go client ↔ storage directly.
- Store just the **object key** in Postgres; never the bytes.
- Local dev uses **MinIO** (S3-compatible) via Docker.

```ts
// presign (aws-sdk v3)
const put = await getSignedUrl(s3, new PutObjectCommand({ Bucket, Key, ContentType }), { expiresIn: 60 });
```

### Env
```bash
S3_ENDPOINT=http://localhost:9000     # MinIO locally; empty/AWS in prod
S3_REGION=us-east-1
S3_BUCKET=chat-uploads
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_FORCE_PATH_STYLE=true              # required for MinIO
```

## Result
After this doc: kill any instance, restart it, start a second instance — sessions survive, files survive,
and either instance serves any user. The app is now stateless. Redis specifics (client, cache-aside,
keys) are in [03-redis.md](03-redis.md).
