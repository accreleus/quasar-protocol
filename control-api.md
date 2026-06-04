# P1-B — Control-plane public/control API (FROZEN)

The HTTP/JSON API the **web client** (and later native clients) talk to: auth, library,
launch, session lifecycle. This is the public face of the control plane; the node agent is
never contacted directly by a client (architecture invariant #1). The launch endpoint is the
hinge between this contract and `signaling.md` (P1-D): it returns the single-use signaling
coordinates the client then connects with.

**Frozen** (see `CLAUDE.md`): P1-2 (auth), P1-3 (CRUD), P1-9/P1-10 (web client) implement
against this freely; changes need Opus + human sign-off. Co-defined with `schema.md` (response
bodies mirror its rows and **session states**) and `signaling.md` (the launch response).

> **Amendment — P2-01 (resource-governance + admission-control), requires sign-off.** Adds the
> §Admission control section below and three error codes — `no_host_available` (503),
> `capacity_exhausted` (503), `session_quota_exceeded` (409) — to the enumerated set. These are
> **additive** (new codes, a new behavioural rule for a launch path that at N=1 never actually
> rejected). **One refinement is not purely additive and is the item this sign-off covers:** the
> launch capacity-rejection moves from `409 capacity_unavailable` to the two new **503** codes,
> because the host being up-but-full (or no host being up) is a *server/transient* condition, not
> a 4xx client error. `capacity_unavailable` is **retained** in the enum (it is frozen — not
> removed) but is **no longer emitted** by the launch path; it is superseded by the two 503 codes.
> No request body, response body, or existing status-code-for-an-existing-code changes. See
> `docs/phase2/P2-01-contract-resource-governance.md`. **Stops at the contract — the governor
> that emits these codes is P2-03.**

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
  `conflict` (409, e.g. duplicate email), `session_quota_exceeded` (409, *P2-01* — caller is at
  their per-user concurrent-session limit), `rate_limited` (429), `no_host_available` (503,
  *P2-01* — no online host/GPU can serve the request), `capacity_exhausted` (503, *P2-01* — a
  matching GPU is online but its free encode slots / VRAM cannot satisfy the request right now),
  `internal` (500). **Retryable:** `no_host_available`, `capacity_exhausted`, `rate_limited`
  (and `session_quota_exceeded` once one of the caller's sessions ends).
  *Superseded:* `capacity_unavailable` (409) — the original Phase-1 launch capacity-rejection;
  retained for compatibility but **no longer emitted** (the launch path now returns the more
  specific 503 `no_host_available` / `capacity_exhausted`, see §Admission control). At N=1 it
  never actually fired, since one GPU's generous slots always satisfied the request.
- **Pagination** (list endpoints) is `?limit=&cursor=`; responses carry `{ "items": [...],
  "next_cursor": "<opaque|null>" }`. Generous defaults at N=1; the shape is real.

---

## Authorization — roles & the admin surface

> *Additive, admin-gated amendment (P1-Auth-Enforce). This does not change any
> request/response shape, status code, or endpoint already frozen above — it makes
> explicit the role gate the admin surface always implied (it is "no frozen-interface
> change — additive, admin-gated", as the Library section already notes). Implemented
> in the control plane as `RequireAuth → RequireAdmin` middleware.*

Two roles exist, from `schema.md` `users.role` (`CHECK (role IN ('user','admin'))`):
`user` (default for every `/v1/auth/register` account) and `admin`.

**Normative rule (server-enforced, never UI-gated).** An admin endpoint **MUST**
reject a valid **non-admin** bearer token with `403 forbidden`, independent of which
client made the call. Hiding the admin UI is **never** the access control — the gate
is this server check. Order of checks: missing/invalid token ⇒ `401`; valid token but
insufficient role ⇒ `403`. The `403` precedes any resource lookup, so an admin
endpoint never leaks existence (e.g. a non-admin `PATCH` of any app id is `403`, not
`404`).

| endpoint | required role | notes |
|---|---|---|
| `POST /v1/apps` | **admin** | create an app |
| `PATCH /v1/apps/{id}` | **admin** | edit an app |
| `GET /v1/hosts`, `GET /v1/hosts/{id}` | **admin** | host/capacity oversight |
| `GET /v1/apps`, `GET /v1/apps/{id}` | user / public | the public library (list is unauthenticated) |
| `GET /v1/sessions/{id}`, `GET /v1/sessions`, `DELETE /v1/sessions/{id}` | **owner or admin** | resource-ownership check (`403` otherwise), not a blanket admin gate |
| everything else (`/v1/me`, `POST /v1/sessions`, …) | user | any authenticated account |

### First-admin bootstrap (decision)
A fresh database has no admin, and `/v1/auth/register` deliberately mints **only**
`role=user` (so the role can never be claimed from the wire). The first admin is
therefore **operator-provisioned**, not "whoever registers first" — on a publicly
reachable fresh instance the latter would let an attacker who races to `register`
seize admin. The control plane, at startup, reads:

- `BOOTSTRAP_ADMIN_EMAIL`, `BOOTSTRAP_ADMIN_USERNAME`, `BOOTSTRAP_ADMIN_PASSWORD`

and, **only when no admin yet exists**, provisions that admin (creating the account,
or promoting an already-registered account with that email). It is idempotent and a
no-op once any admin exists — safe to run on every boot, and serialized by an advisory
lock so concurrently-booting instances cannot create two. Operators rotate/extend admins
thereafter through the admin surface. Leaving the variables unset is valid (no admin is
seeded); a partial set is an operator error.

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
- The scheduler reserves transactionally against `gpus` availability (`schema.md`), and first
  enforces the caller's per-user session quota. Rejections (see §Admission control for the exact
  rule) — in all of which **no session row persists** (quota/availability is checked before the
  row is committed, or the row is rolled back):
  - **`409 session_quota_exceeded`** — the caller already holds `max_concurrent_sessions` active
    sessions (`state ∈ {pending, assigned, starting, running}`).
  - **`503 no_host_available`** — no online host has a GPU that could serve the request.
  - **`503 capacity_exhausted`** — a matching GPU is online but its available encode slots / VRAM
    (totals − active reservations) cannot satisfy the request right now; retryable once a session
    ends.

  At N=1 with generous slots and a quota of 3, the single GPU always satisfies an in-quota
  request — the reservation and quota code paths still run. *(Phase-1 documented a single
  `409 capacity_unavailable` here; P2-01 supersedes it with the codes above — see the amendment
  note at the top of this file.)*
- `signaling.token` is single-use, short-TTL (default 60 s), stored **hashed** on the session
  (`signaling_token_hash`). It is returned plaintext only here. The client must connect promptly.
- The response returns as soon as the session is `assigned` (reservation committed, assign sent
  to the agent). The client then connects to `signaling.url?token=…` and watches the session
  advance `assigned → starting → running` via signaling/`GET`.
- `state` may already be `starting`/`running` by the time the client reads it — these are the
  `schema.md` states verbatim.

### Admission control (P2-01)
> *Additive amendment — the rule the launch path enforces. Defines wire behaviour only; the
> implementation (atomic, no-double-admit-under-concurrency) is P2-03.*

A `POST /v1/sessions` launch is admitted only if **both** gates pass, evaluated in this order:

1. **Per-user session quota.** Let `active` = the caller's sessions in `state ∈ {pending,
   assigned, starting, running}` (the non-terminal, pre-teardown set — see `schema.md`
   `users.max_concurrent_sessions`). If `count(active) ≥ user.max_concurrent_sessions`, reject
   with **`409 session_quota_exceeded`** and persist no row. A `max_concurrent_sessions` of `0`
   blocks every launch for that user. This gate is per-user and independent of host capacity.

2. **Per-GPU capacity (the governor).** The request's resource ask comes from the app row —
   `requested_encode_slots = apps.default_encode_slots`, `requested_vram_mb = apps.default_vram_mb`
   (clients never set these; the `stream` block carries resolution/bitrate, not resource
   reservations). A GPU admits the launch iff, with availability derived per `schema.md` §gpus
   (`total − Σ reservations of sessions in {assigned, starting, running}`):
   ```
   encode_slots_available ≥ requested_encode_slots   AND   vram_available ≥ requested_vram_mb
   ```
   - If **no online host** has any GPU that could serve the request — no host is `online`, or none
     has a GPU whose **totals** could ever satisfy the ask — reject with **`503 no_host_available`**.
     This is the "fleet is empty/down" condition; it is distinct from "full".
   - If a candidate GPU exists (its totals suffice) but **no** candidate currently has enough
     **available** slots/VRAM, reject with **`503 capacity_exhausted`**. This is the "up but full
     right now" condition; it is **retryable** — a later launch may succeed once an active session
     ends and frees its reservation.
   - Otherwise the scheduler picks a satisfying GPU and reserves transactionally
     (`SELECT … FOR UPDATE` on the `gpus` row, P1-8 / P2-03) so concurrent launches cannot
     oversubscribe, commits the `sessions` row as `assigned`, and returns `201`.

Both 503s carry the uniform error body and **no** session row persists. The two conditions are
kept distinct so the client and the admin UI can tell "nothing is serving" (`no_host_available`
— operator should check why the agent is offline) from "everything is busy" (`capacity_exhausted`
— a transient, retry-later condition). `503` (not a 4xx) because in both cases the request is
well-formed and authorized; the server simply has no room to place it now.

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
