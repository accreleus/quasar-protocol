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

> **Amendment — P2-02 (launcher↔game swap), additive, requires sign-off.** Adds the
> `POST /v1/sessions/{id}/swap` endpoint (§Sessions), two error codes — `session_not_swappable`
> (409), `swap_exceeds_reservation` (409) — and one Authorization-table row (owner-or-admin, same
> rule as `DELETE`). **Purely additive:** a new endpoint + new codes, no change to any existing
> shape; signaling is explicitly **unchanged** (the swap keeps the same WebRTC transport — see
> `signaling.md`). The swapped app must **fit within the session's held reservation** in Phase 2
> (no reservation resize — deferred). See `docs/phase2/P2-02-contract-app-swap.md`. **Stops at the
> contract — the interpipe implementation is P2-07.**

> **Amendment — P3-01 (host-lifecycle + multi-host scheduling), additive, requires sign-off.**
> Adds the host drain/cordon admin operations `POST /v1/hosts/{id}/drain` and
> `POST /v1/hosts/{id}/uncordon` (§Hosts), two Authorization rows (both **admin**), and the
> `host_lost` session `state_detail` convention (`schema.md`) the offline reaper stamps. **Purely
> additive:** new admin endpoints + a free-text `state_detail` value; **no new error code** (reuses
> `not_found`/`conflict`/`forbidden`), **no DDL** (the `hosts.status` value `draining` already
> exists, `schema.md`), and **no change to signaling or agent-api** (a force-drain reuses the
> existing `session_stop` `reason:"host_draining"`). The host drain/cordon state machine is defined
> in `schema.md` §"Host status state machine". See `docs/phase3/P3-01-contract-host-lifecycle.md`.
> **Stops at the contract — P3-03 implements the lifecycle, P3-04 the failover.**

> **Amendment — P5-01 (storage & state foundation), additive, requires sign-off.** Adds the
> §Storage section (admin home oversight + the caller's own usage read), one error code —
> `home_in_use` (409) — emitted by the launch path, two optional **request** fields on the
> existing admin app create/PATCH (`apps.managed_home` / `apps.home_container_path` —
> additive JSON fields, absent = unchanged), and the matching Authorization rows. **Usage is
> visibility-only in this amendment — no quota column, no quota enforcement** (operator
> decision at sign-off; a quota is a clean later additive amendment if wanted).
> **Purely additive:** new endpoints + new codes + new optional fields; no existing shape,
> status code, or endpoint changes. **`agent-api.md` is explicitly unchanged** — the
> session_assign/swap `app.mounts` array already carries arbitrary mount strings, and the
> per-user home rides in it; this was a considered decision, not an omission. Backing DDL in
> `schema.md` (`user_homes`, two `apps` columns — migration 0008). See
> `docs/phase5/P5-01-contract-storage.md`. **Stops at the contract — P5-02 implements the
> provider, P5-04/P5-05 the guards.**

> **Amendment — #175 (agent backing-store reaping), additive, signed off 2026-06-12 (operator approval, PR #180).** Adds the
> §Agent storage GC section: two **agent-authenticated** HTTP endpoints
> (`GET /v1/agent/storage/gc-pending`, `POST /v1/agent/storage/gc-confirm`) by which a
> node-agent pulls the tombstoned homes on its own host that are past the 24h GC grace period,
> reaps each backing store host-side (docker volume / directory — the control plane has no
> host access, invariant #1), and confirms the reaped ids so the row is hard-deleted. **No new
> user/admin endpoint, no new error code, no change to any existing shape.** Auth is the
> existing per-node `node_secret` (the agent reconnect credential), not a user bearer token —
> see the section for the scheme. **`agent-api.md` is byte-identical** — this is a *new
> additive HTTP surface*, not a WebSocket-message change; the pull deliberately avoids touching
> the frozen agent WS contract. Backing semantics note in `schema.md` (`user_homes` GC
> lifecycle — janitor now row-deletes only host-unpinned tombstones; pinned ones go via the
> agent confirm). See issue #175.

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
  their per-user concurrent-session limit), `session_not_swappable` (409, *P2-02* — session is not
  in a state that can be app-swapped), `swap_exceeds_reservation` (409, *P2-02* — the new app needs
  more than the session's held reservation), `session_not_traceable` (409, *P4-01* — deep trace
  cannot be toggled: the session is not `running`, its host is offline, or the agent rejected a live
  toggle), `home_in_use` (409, *P5-01* —
  the caller already has a live session of this managed-home app; the per-(user, app) home is
  single-writer), `rate_limited` (429), `no_host_available` (503,
  *P2-01* — no online host/GPU can serve the request), `capacity_exhausted` (503, *P2-01* — a
  matching GPU is online but its free encode slots / VRAM cannot satisfy the request right now),
  `internal` (500). **Retryable:** `no_host_available`, `capacity_exhausted`, `rate_limited`
  (and `session_quota_exceeded` / `home_in_use` once one of the caller's sessions ends).
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
| `POST /v1/hosts/{id}/drain`, `POST /v1/hosts/{id}/uncordon` | **admin** | *(P3-01)* host lifecycle — cordon a host out of service / return it |
| `GET /v1/apps`, `GET /v1/apps/{id}` | user / public | the public library (list is unauthenticated) |
| `GET /v1/sessions/{id}`, `GET /v1/sessions`, `DELETE /v1/sessions/{id}` | **owner or admin** | resource-ownership check (`403` otherwise), not a blanket admin gate |
| `POST /v1/sessions/{id}/swap` | **owner or admin** | *(P2-02)* same ownership check as `DELETE` |
| `POST /v1/sessions/{id}/stats` | **owner or admin** | *(P4-01)* the client posts its own session's browser telemetry — same ownership check as `DELETE` |
| `GET /v1/admin/sessions/{id}/metrics` | **admin** | *(P4-01)* per-session telemetry read (oversight) |
| `POST /v1/admin/sessions/{id}/trace` | **admin** | *(P4-01)* toggle the session's deep trace |
| `POST /v1/me/devices` | user (self) | *(P4-01)* upsert the caller's own device capability; owner is the bearer identity, never a body field |
| `GET /v1/admin/storage/homes` | **admin** | *(P5-01)* list managed homes (storage oversight) |
| `DELETE /v1/admin/storage/homes/{id}` | **admin** | *(P5-01)* tombstone a home for GC |
| `GET /v1/me/storage` | user (self) | *(P5-01)* the caller's own per-app storage usage |
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

### `POST /v1/sessions/{id}/swap` — swap the running app (P2-02)
Swaps the **source app** of a live session (launcher → game, or game → game) **without tearing
down the WebRTC transport**: the node-agent swaps the source container behind a GStreamer
interpipe boundary while encode + `webrtcbin` stay up, so the browser stream never re-negotiates
(`signaling.md` is unchanged — same offer, same DataChannel). Owner-or-admin (same rule as
`DELETE`).
```json
// request — the app to swap to
{ "app_id": "<uuid>" }
// 202 — swap accepted; agent is performing it. Body is the current session.
{ "session": {
    "id": "<uuid>", "app_id": "<uuid>",   // app_id is still the OLD app until the swap completes
    "state": "running", "state_detail": "swapping", "error_message": null,
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "main" },
    "created_at": "...", "started_at": "...", "ended_at": null
  } }
```
- **Async, like launch/stop.** The control plane validates, sends `session_swap_app`
  (`agent-api.md`) to the assigned host, sets `state_detail = "swapping"`, and returns `202`. The
  client polls `GET /v1/sessions/{id}` to watch `state_detail` go `swapping → ` (a running detail)
  on success. **The top-level `state` stays `running` throughout** — swap does not change the
  session's scheduled/reserved status (see `schema.md`).
- **On success** the control plane sets `sessions.app_id` to the new app and clears the `swapping`
  detail. **On a failed swap that the agent rolled back**, `app_id` is unchanged and the session
  stays `running` on the previous app (the detail reports the failure). **On a failed swap with no
  recoverable previous app**, the session goes `failed` and its reservation is released
  (`agent-api.md` defines which). A browser already attached keeps its media throughout.
- **Reservation rule (Phase 2): the swap must fit within the held reservation.** The new app's
  `default_vram_mb` / `default_encode_slots` must each be ≤ the session's `reserved_vram_mb` /
  `reserved_encode_slots`. If not, `409 swap_exceeds_reservation` and **no** swap is attempted
  (the session is untouched). Reservation *resize* on swap is deferred past Phase 2.
- **Errors** (the session is left untouched in every rejection):
  - `409 session_not_swappable` — the session is not in a swappable state: top-level `state` is not
    `running`, or a swap is already in progress (`state_detail = "swapping"`).
  - `404 not_found` — no such app, or the app is disabled (same visibility rule as `GET /v1/apps/{id}`).
  - `409 swap_exceeds_reservation` — as above.
  - `403 forbidden` — caller is neither owner nor admin; `404 not_found` — no such session.

---

## Hosts (admin lifecycle, P3-01)
> *Additive, admin-gated amendment. New endpoints; no change to any existing shape. The host
> status values and transitions are defined in `schema.md` §"Host status state machine".*

Host **read** oversight (`GET /v1/hosts`, `GET /v1/hosts/{id}`, and the P2-09 capacity reads) is
the admin oversight surface. P3-01 adds the two **lifecycle** operations an operator uses to take a
host out of service for maintenance and bring it back. Both are **admin-only** (a valid non-admin
bearer is `403`, before any host lookup, per §Authorization) and both return the updated host body.

### `POST /v1/hosts/{id}/drain` — cordon a host
Marks an `online` host `draining`: the scheduler places **no new sessions** on it, while its
existing sessions are allowed to finish (graceful) or are stopped now (force). The host stays
`draining` (reachable, still heartbeating) until an admin uncordons it — it does **not**
auto-transition to `offline`.
```json
// request — force is optional (default false = graceful)
{ "force": false }
// 200 — host is now draining
{ "host": { "id": "<uuid>", "node_name": "gpu-host-01", "status": "draining", "...": "..." } }
```
- **Graceful (`force:false`, default):** the host goes `draining`; running sessions are left to end
  on their own. The drain stops nothing.
- **Force (`force:true`):** the host goes `draining` and the control plane sends `session_stop` with
  `reason:"host_draining"` (`agent-api.md`) to every non-terminal session on the host; each tears
  down `stopping → stopped` and releases its reservation.
- **Idempotent.** Draining an already-`draining` host is a `200` no-op (a `force:true` re-drain
  stops any sessions that have since appeared).
- **Errors:** `404 not_found` (no such host); `409 conflict` (host is `offline` — nothing to drain).

### `POST /v1/hosts/{id}/uncordon` — return a host to service
Returns a `draining` host to `online` so the scheduler may place on it again.
```json
// 200 — host is back online
{ "host": { "id": "<uuid>", "node_name": "gpu-host-01", "status": "online", "...": "..." } }
```
- Requires the host's **agent to be connected** (only a live, heartbeating host can accept work).
  Idempotent on an already-`online` host (`200` no-op).
- **Errors:** `404 not_found`; `409 conflict` (host is `offline` — its agent is not connected, so
  there is nothing to return to service; the host comes back `online` on its own when its agent
  reconnects).

---

## Telemetry & devices (P4-01)
> *Additive, observability amendment. New endpoints; no change to any existing shape, status
> code, or endpoint. Reads are admin-gated, writes are owner-gated (the bearer identity, never
> a body field) — telemetry and device data are **observability, never the access control**,
> enforced server-side like every endpoint here. Stored per `schema.md` `session_metrics` /
> `user_devices`. No consumer/optimizer reads `user_devices` in this phase.*

### `POST /v1/sessions/{id}/stats` — client posts its session telemetry
The browser reports its own `getStats()`-derived telemetry for **its** live session. Owner-or-
admin (same ownership rule as `DELETE /v1/sessions/{id}`; a non-owner is `403`, an unknown id
`404`). Returns **`202`** (accepted, no body). Untrusted client input: the body is bounded to
**≤ 64 samples per request and ≤ 32 KB**, and **rate-limited** (`429 rate_limited`); only the
keys in the `schema.md` field dictionary are persisted (unknown keys are ignored, not stored).
```json
// request — a small batch of recent samples (browser clock, ms)
{ "samples": [
    { "ts_unix_ms": 1735689600000,
      "metrics": { "fps": 59.6, "bitrate_kbps": 14710, "rtt_ms": 12, "jitter_buffer_ms": 28,
                   "decode_ms": 1.4, "packets_lost": 0, "frames_dropped": 0,
                   // presentation pacing (#108, always-on; schema.md dictionary):
                   "present_fps": 59.8, "present_interval_sd_ms": 2.9,
                   "present_interval_p95_ms": 18.0, "playout_target_ms": 100,
                   // deep trace only — the staged glass-to-glass budget (schema.md dictionary):
                   "glass_to_glass_ms": 71, "encode_ms": 4.6, "network_pacing_ms": 7.5,
                   "decode_display_ms": 30.9, "interactive_ms": 92 } }
  ] }
// 202 — accepted (no body); samples are written with source='browser'
```
- Each sample becomes a `session_metrics` row with `source='browser'`. The staged-budget keys
  (`glass_to_glass_ms`, `network_pacing_ms`, `decode_display_ms`, `interactive_ms`, …) appear
  only when the session is in deep trace; the lightweight keys always.
- Best-effort: a malformed sample is dropped, not fataled. Accepting telemetry never affects
  session state.

### `GET /v1/admin/sessions/{id}/metrics` — read per-session telemetry (admin)
Returns the **bounded recent window** of telemetry for one session, both sources, newest first.
Admin-only (`403` before any lookup per §Authorization; `404` for an unknown session to an admin).
```json
// 200 — paginated, newest first; ?limit=&cursor=&source=agent|browser optional
{ "items": [
    { "source": "agent",   "ts_unix_ms": 1735689600000,
      "metrics": { "fps": 59.8, "bitrate_kbps": 14820, "encode_ms": 4.6, "frames_dropped": 1 } },
    { "source": "browser", "ts_unix_ms": 1735689600100,
      "metrics": { "fps": 59.6, "rtt_ms": 12, "jitter_buffer_ms": 28, "decode_ms": 1.4,
                   "glass_to_glass_ms": 71 } }
  ],
  "next_cursor": null }
```
The window is bounded by `schema.md`'s retention policy — this is "recent history", not the full
series. The admin session **list** (`GET /v1/admin/sessions`) may additionally carry an optional
additive `latest_metrics` object per item (the most-recent merged sample) so the list view shows
current values without an N+1 fan-out; absent when a session has no telemetry yet.

### `POST /v1/admin/sessions/{id}/trace` — toggle deep trace (admin)
Turns the session's deep glass-to-glass trace on/off. The control plane relays `session_trace`
(`agent-api.md`) to the owning agent and returns the current session body.
```json
// request
{ "deep_trace": true }
// 200 — relayed; body is the current session (same shape as GET /v1/sessions/{id})
{ "session": { "id": "<uuid>", "state": "running", "state_detail": "pipeline live", "...": "..." } }
```
- **Errors:** `404 not_found` (no such session); `409 session_not_traceable` (the session is not
  `running`, its host is `offline`, or the agent rejected a live toggle — e.g. deep trace is
  start-time-only on that agent, `agent-api.md`). A rejected toggle **never** changes session
  state — the session is left exactly as it was.

### `POST /v1/me/devices` — upsert the caller's device capability
The client posts its login-time connection + decode-capability probe. Owner is the **bearer
identity** (never a body field); a user can only write their own devices. Upsert on
`(user_id, device_key)` (`schema.md` `user_devices`): insert on first sight, update
`capabilities`/`last_seen_at`/`user_agent` thereafter. Body is **size-validated** (`400
validation_failed` on garbage) and **rate-limited** (`429`, low ceiling — the upsert is
idempotent on `(user_id, device_key)`, so a login-frequency call is ample). `measured_at` in
`capabilities` is **server-stamped**, ignoring any client-supplied value.
```json
// request
{ "device_key": "<client-generated opaque id, persisted in localStorage>",
  "capabilities": { "codecs": {"h264":true,"hevc":false,"av1":false,"vp9":true},
                    "max_decode_height": 2160, "bandwidth_kbps": 48000, "rtt_ms": 12 } }
// 200 — upserted (no sensitive body)
{ "device": { "id": "<uuid>", "first_seen_at": "...", "last_seen_at": "..." } }
```
- `device_key` is an app-scoped opaque id the client generates and persists — **not** a hardware
  fingerprint (see `schema.md` `user_devices` privacy note). The probe is **best-effort and
  non-blocking** at login (`P4-08`); failure to post is silent and never blocks sign-in.
- **No consumer.** Nothing in this phase reads `user_devices` to alter a session/codec/fps
  decision — the optimizer is a later phase; Phase 7 surfaces/manages the device list.

---

## Storage (P5-01)

> *Additive amendment — per-user managed homes (Phase 5). DDL companion: `schema.md`
> `user_homes` + `apps.managed_home`/`home_container_path`.*

When an app has `managed_home = true`, the control plane injects one extra mount into the
app's `mounts` array at dispatch (assign **and** swap) so the user's per-(user, app) home is
bound at `home_container_path` (default `/home/quasar`). The wire shape of
`session_assign`/`session_swap_app` is untouched — the home rides in the existing opaque
`mounts` array (`agent-api.md` unchanged, by design).

Launch-path behaviour (POST /v1/sessions, additive rule only):
- managed-home app + the caller already has a live session of the **same app** ⇒
  `409 home_in_use` (single-writer per (user, app); a different app concurrently is fine).

Storage usage is **visibility-only** in this amendment: `bytes_used` is reported and
surfaced (admin + self reads below) but nothing enforces a cap. A per-user quota, if ever
wanted, is a later additive amendment (one column + one error code).

### `GET /v1/admin/storage/homes` — list managed homes (admin)
Query: `user_id`, `app_id`, `pending_gc` (bool) — all optional filters. Paginated like
`GET /v1/users`.
```json
{ "items": [ { "id": "…", "user_id": "…", "app_id": "…", "host_id": "…",
  "provider": "volume", "ref": "quasar-home-…", "bytes_used": 123456789,
  "created_at": "…", "last_used_at": "…", "gc_after": null } ], "next_cursor": null }
```

### `DELETE /v1/admin/storage/homes/{id}` — tombstone a home (admin)
Marks the home for GC (`gc_after = now()`); the janitor reaps the backing store
asynchronously. `202` accepted; `404` unknown; `409 home_in_use` if a live session of that
(user, app) currently mounts it.

### `GET /v1/me/storage` — the caller's own usage
```json
{ "items": [ { "app_id": "…", "app_name": "…",
  "bytes_used": 123456789, "last_used_at": "…" } ] }
```
Owner is the bearer identity.

### Agent storage GC (#175 amendment — signed off 2026-06-12)

> *Backing-store reaping for tombstoned homes. The control plane tombstones a `user_homes`
> row (`gc_after = now()`) but cannot remove the home's backing store — it has no host/docker
> access (invariant #1). So the **node-agent pulls** the reapable homes on its own host,
> removes the docker named volume / local directory host-side, and confirms the ids, which is
> what hard-deletes the row. This is a **new additive HTTP surface; the agent WebSocket
> contract (`agent-api.md`) is byte-identical** — no new WS message.*

**Authentication (not a user bearer token).** These two endpoints authenticate the calling
*node-agent*, not a user. The agent presents:
- `Authorization: Bearer {node_secret}` — the per-node secret issued at enrollment (the same
  credential the agent uses to reconnect over `/agent/ws`).
- `X-Quasar-Node: {node_name}` — the agent's stable node name.

The control plane looks up the `hosts` row by `node_name` and verifies `node_secret` against
`hosts.node_secret_hash` using the **same `hex(sha256(secret))` scheme** the agent WS reconnect
uses (constant-time). The resolved `host_id` scopes every query — an agent can only ever see
and reap homes pinned to its own host. Any failure (missing/empty headers, unknown node, bad
secret) → `401 unauthorized`; the response never distinguishes which.

#### `GET /v1/agent/storage/gc-pending`
Returns up to 100 homes pinned to the calling agent's host that are past the 24h GC grace
period (`gc_after IS NOT NULL AND gc_after + interval '24 hours' < now()`). Host-unpinned
(`host_id IS NULL`) tombstones are never returned here — no agent owns them; the control-plane
janitor row-deletes those directly.
```json
{ "homes": [ { "id": "…", "provider": "volume", "ref": "quasar-home-…" } ] }
```
`provider` is `"volume"` or `"local"`; `ref` is the docker volume name or the absolute host
directory path. The list is a hint, not a lock — an empty list is the steady state.

#### `POST /v1/agent/storage/gc-confirm`
Body: `{ "home_ids": ["…", …] }`. For each id the control plane hard-deletes the row **only**
if it is still a past-grace tombstone on the calling host:
```sql
DELETE FROM user_homes
WHERE id = $1 AND host_id = $callingHost
  AND gc_after IS NOT NULL AND gc_after + interval '24 hours' < now()
```
This guard makes a confirm a **no-op** for a home that was *revived* (a launch cleared
`gc_after` between the agent's pull and its confirm) or *relocated* to another host — the
live row survives, and the agent's reap of a now-stale backing store is harmless (its reap is
idempotent). Response:
```json
{ "deleted": 2 }
```
`deleted` is the count of rows actually removed (≤ `home_ids` length).

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
