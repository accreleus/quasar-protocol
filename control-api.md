# P1-B — Control-plane public/control API (FROZEN)

The HTTP/JSON API the **web client** (and later native clients) talk to: auth, library,
launch, session lifecycle. This is the public face of the control plane; the node agent is
never contacted directly by a client (architecture invariant #1). The launch endpoint is the
hinge between this contract and `signaling.md` (P1-D): it returns the single-use signaling
coordinates the client then connects with.

**Frozen** (see `CLAUDE.md`): P1-2 (auth), P1-3 (CRUD), P1-9/P1-10 (web client) implement
against this freely; changes need Opus + human sign-off. Co-defined with `schema.md` (response
bodies mirror its rows and **session states**) and `signaling.md` (the launch response).

## Conventions
- Base path **`/v1`**. JSON request and response bodies; `Content-Type: application/json`.
- TLS in deployment (architecture: control plane is the public ingress). The web client derives
  the API origin from `location.origin`, mirroring the `signaling.md` rule for the WS URL.
- **Auth:** all endpoints except `/v1/auth/register` and `/v1/auth/login` require
  `Authorization: Bearer <access_token>` (the opaque token from `auth_tokens`, see `schema.md`).
  Missing/invalid/expired/revoked ⇒ `401`.
- **Tokens, never passwords, on the wire** beyond the two auth endpoints (P1-2). The password is
  sent only to register/login over TLS; everything else is bearer-token.
- **IDs** are UUID strings. **Timestamps** are RFC3339/ISO-8601 UTC strings. Enumerations
  (session `state`, `h264_profile`, host `status`, user `role`) use the exact `schema.md` values.
- **Errors** use a uniform body and conventional status codes:
  ```json
  { "error": { "code": "invalid_credentials", "message": "human readable" } }
  ```
  Codes: `validation_failed` (400), `unauthorized` (401), `forbidden` (403), `not_found` (404),
  `conflict` (409, e.g. duplicate email), `capacity_unavailable` (409, no host/GPU can satisfy a
  launch), `rate_limited` (429), `internal` (500).
- **Pagination** (list endpoints) is `?limit=&cursor=`; responses carry `{ "items": [...],
  "next_cursor": "<opaque|null>" }`. Generous defaults at N=1; the shape is real.

---

## Auth

### `POST /v1/auth/register`
```json
// request
{ "email": "a@b.com", "username": "ada", "password": "<plaintext, TLS only>" }
// 201
{ "user": { "id": "<uuid>", "email": "a@b.com", "username": "ada", "role": "user", "created_at": "..." } }
```
Password hashed with **argon2id** (P1-2). `409 conflict` on duplicate email/username. No token is
returned — the client logs in next (keeps register/login flows independent).

### `POST /v1/auth/login`
```json
// request
{ "email": "a@b.com", "password": "<plaintext, TLS only>" }
// 200
{
  "access_token": "<opaque bearer>",
  "token_type": "Bearer",
  "expires_at": "2026-06-02T12:00:00Z",
  "user": { "id": "<uuid>", "email": "a@b.com", "username": "ada", "role": "user" }
}
```
The opaque token is stored **hashed** in `auth_tokens` and returned in plaintext exactly once
here. `401 invalid_credentials` on bad password / unknown email / disabled account (no
distinction, to avoid user enumeration).

### `POST /v1/auth/logout`
Revokes the presented bearer token (`auth_tokens.revoked_at = now()`). `204`. Idempotent.

### `GET /v1/me`
```json
// 200
{ "user": { "id": "<uuid>", "email": "...", "username": "...", "role": "user", "created_at": "..." } }
```

> Token refresh and "list my active tokens/sessions" are deliberate Phase-2 additions; the
> `auth_tokens` table already supports them. Not in the frozen Phase-1 surface.

---

## Library

### `GET /v1/apps`
Lists enabled apps (the library the user can launch).
```json
// 200
{ "items": [
    { "id": "<uuid>", "name": "Foo", "description": "...", "cover_url": "https://...",
      "default_width": 1920, "default_height": 1080, "default_fps": 60, "default_bitrate_kbps": 15000 }
  ], "next_cursor": null }
```
`runtime_spec` and resource defaults are **not** exposed to clients (agent-internal /
scheduler-internal). Disabled apps are omitted.

### `GET /v1/apps/{id}`
Single app, same fields as a list item. `404` if absent or disabled.

> Creating/editing apps and managing hosts (`GET/POST/PATCH /v1/apps`, `GET /v1/hosts`) is the
> **admin** surface (`role=admin`), built in P1-3 against the same `schema.md`. The read shapes
> above are the public subset; admin write shapes are P1-3's to define within this contract's
> conventions (no frozen-interface change — they're additive, admin-gated).

---

## Sessions (launch + lifecycle)

### `POST /v1/sessions` — launch
Creates a session, runs the scheduler (pick host+GPU, reserve VRAM+encode slots), tells the
agent to assign+start, and returns the **signaling coordinates** the client connects with. This
is the single hinge to `signaling.md`.
```json
// request — app_id required; stream params optional (default from the app row)
{
  "app_id": "<uuid>",
  "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000 }
}
// 201
{
  "session": {
    "id": "<uuid>", "app_id": "<uuid>", "state": "assigned",
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "constrained-baseline" },
    "created_at": "..."
  },
  "signaling": {
    "url": "wss://<control-plane-host>/v1/signal",
    "token": "<single-use, plaintext, this response only>",
    "expires_at": "2026-06-01T12:01:00Z"
  }
}
```
- The scheduler reserves transactionally against `gpus` availability (`schema.md`). If nothing
  can satisfy the request: `409 capacity_unavailable` and **no** session row persists. At N=1 the
  single GPU's generous slots always satisfy it — the reservation code path still runs.
- `signaling.token` is single-use, short-TTL (default 60 s), stored **hashed** on the session
  (`signaling_token_hash`). It is returned plaintext only here. The client must connect promptly.
- The response returns as soon as the session is `assigned` (reservation committed, assign sent
  to the agent). The client then connects to `signaling.url?token=…` and watches the session
  advance `assigned → starting → running` via signaling/`GET`.
- `state` may already be `starting`/`running` by the time the client reads it — these are the
  `schema.md` states verbatim.

### `GET /v1/sessions/{id}` — poll lifecycle
```json
// 200
{ "session": {
    "id": "<uuid>", "app_id": "<uuid>", "host_id": "<uuid|null>",
    "state": "running", "state_detail": "pipeline live", "error_message": null,
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "main" },
    "created_at": "...", "started_at": "...", "ended_at": null
  } }
```
Only the owner (or an admin) may read a session (`403` otherwise). The signaling token is
**never** returned here — only in the launch response.

### `GET /v1/sessions` — the user's sessions
List, newest first, owner-scoped. Same item shape as `GET /v1/sessions/{id}`.

### `DELETE /v1/sessions/{id}` — stop
Requests teardown: control plane sends `session_stop` to the agent (`agent-api.md`), session
goes `stopping → stopped`, reservation released. `202` with the current session body. Idempotent
on an already-terminal session (`200`/`202`, no error).

---

## How the client uses this (end-to-end)
```
register ─▶ login ─▶ GET /v1/apps ─▶ POST /v1/sessions {app_id}
                                          │  returns signaling.url + single-use token
                                          ▼
            connect  wss://…/v1/signal?token=…   (signaling.md — control plane validates+consumes,
                                                   bridges to the node agent over agent-api.md)
                                          ▼
            WebRTC offer/answer/ICE, then media + input DataChannel flow (signaling.md / input.md)
                                          ▼
            poll GET /v1/sessions/{id} for state; DELETE to stop
```
The client only ever talks to the control plane; the node agent is reached transparently via the
signaling relay. This is the architecture's split, made literal at the API boundary.
