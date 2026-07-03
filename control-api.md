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
> in `schema.md` §"Host status state machine". See `docs/completed/phase3/P3-01-contract-host-lifecycle.md`.

> **Amendment — admin delete (app + host), additive, requires sign-off.** Adds two
> admin-gated terminal operations: `DELETE /v1/apps/{id}` (§Library) and `DELETE /v1/hosts/{id}`
> (§Hosts), plus their two Authorization rows (both **admin**). **Purely additive:** new admin
> endpoints; **no change to any existing shape**; **no new error code** (reuses `not_found` /
> `conflict`). Both are *refuse-if-in-use*: a `409 conflict` while the target is live (an app
> with a non-terminal session; a host that is online or still hosting a non-terminal session),
> else a `204` hard delete that **cascades the target's terminal session history** and tombstones
> its managed homes for GC. The cascade + the FK policy changes are defined in `schema.md`
> (`sessions.app_id`/`sessions.host_id` → `ON DELETE CASCADE`, `user_homes.host_id` → `ON DELETE
> SET NULL`, migration `0014`). No change to signaling or agent-api. See
> `docs/ui-polish/2026-06-24-ui-polish.md` (Thread 1).
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

> **Amendment — P9-01 (native-client prelude), additive, requires sign-off.** Supports the
> first-party native client (Phase 9, issue #234). Two additions: **(1)** an optional
> **client version handshake** on `POST /v1/auth/login` — request `client_version` /
> `contract_version` (optional; web/legacy clients omit them), response `min_client_version`
> / `latest_client_version` (advisory; absent = no floor advertised). This amendment defines
> the **fields and their semantics only — the enforcement rule (hard-gate below the floor,
> soft-warn below latest) is P9-08 (#236)**, not here. **(2)** a third telemetry **source**,
> `native`: an optional `client` discriminator on `POST /v1/sessions/{id}/stats`
> (`"browser"` default | `"native"`) that sets the persisted `session_metrics.source`, and
> `native` becomes a valid value of the existing `?source=` filter on the admin metrics
> read. The discriminator is a **label, not access control** — ownership is still the bearer
> identity, never a body field (the P4-01 rule is unchanged). **Purely additive:** new
> optional request fields, new optional response fields, one new enum value; no existing
> shape, status code, or endpoint changes. The capability-report ingest (AS10-08/AS10-12) is
> **unchanged** — native rides the existing `user_devices.capabilities` JSONB.
> **`agent-api.md`, `signaling.md`, `input.md`, `native-client.md` are unchanged** (gamepad
> input + HEVC/AV1 are deferred, not in this prelude). Backing change in `schema.md`: the
> `session_metrics.source` `CHECK` widens to include `'native'` (**migration 0014**). See
> `docs/phase9/P9-01-contract-prelude.md`. **Stops at the contract — P9-07 implements the
> native producer + migration 0014, P9-08 the handshake enforcement.**

> **Amendment — ST-01 (Observability v2 — session trace), additive, requires sign-off.** Adds the
> §Session trace (Observability v2) section: seven endpoints — five **admin-only** reads
> (`GET .../trace`, `.../trace/window`, `.../trace/metrics`, `.../trace/events`,
> `.../diagnostic-bundle`), one **admin-only** write (`POST .../trace/annotations`), and two
> **owner-or-admin** client ingest endpoints (`POST /v1/sessions/{id}/trace/events`,
> `POST /v1/sessions/{id}/trace/clock`) — plus their Authorization-table rows. **No existing
> request/response shape, status code, or endpoint changes.** Reads return a **bounded window**
> (default 5 min, clamp `[2,10]` min — "recent history", never the full series). The diagnostic
> bundle assembles the existing `session_metrics` JSONB (normalized to a read-time taxonomy) joined
> with the new `session_trace_events`/`session_trace_clock` tables (`schema.md`, migration `0016`)
> plus a v0 **observational** classifier verdict (no automatic action). Backing DDL in `schema.md`.
> **`agent-api.md`** gains one additive upstream message, `session_trace_event` (fire-and-forget,
> no session-state authority). `signaling.md`, `input.md`, `native-client.md` are unchanged. See
> `docs/session-trace/contract-amendment.md` and `docs/session-trace/trace-format.md`.

> **Amendment — SPT-05 (Stream Perf Tuning Phase C), additive, requires sign-off.** Three pieces.
> **(1)** Adds the §Host encoder certification section: six **admin-only** endpoints (a
> script-orchestrated run lifecycle — open a run, launch/finalize one bench cell, poll/complete the
> run — plus a read of the latest verdicts) and their Authorization-table rows, backed by the new
> `host_encoder_certification` table (`schema.md`, migration `0018`). **(2)** A **behavioral**
> scheduler decision rule (no wire-shape change): at launch, resolving the **default** profile on a
> target host consults the latest certification row and will not default-start a profile whose
> `encode_ms_p95 > 0.70 × budget_ms` (`verdict='unsafe'`) — capping to the next sustainable rung of
> the same resolution family; an **explicit** profile/override request always bypasses the cap.
> `POST /v1/sessions` and `GET /v1/me/profiles` are unchanged on the wire. **(3)** Adds one
> **optional** request field to `POST /v1/sessions`: `client_type ∈ {"native","browser"}` (absent ⇒
> `"browser"`, lenient parse — an unexpected value coerces to `"browser"`, never a `400`). Its only
> effect is gating the Path-B H.264 profile lift (main/high) to a launch that **itself** declares
> `client_type:"native"` **and** whose account's latest stored device probe is native-capable — this
> fixes a bug where a stored native probe could otherwise poison a subsequent **browser** launch on
> the same account with a profile Chrome cannot decode. The web SPA does not send the field (floor
> by default); the native client (quasar-client, PB-3) must send it to receive the lift.
> **Purely additive** — new endpoints + one new optional request field; no existing shape, status
> code, or endpoint changes. SPT-07's probe-envelope consumption reads the **existing**
> `user_devices.capabilities` JSONB — no schema/API change. `abr_mode` (`QUASAR_ABR_MODE`) is
> node-agent config, not a contract change. **`agent-api.md` is unchanged** — certification composes
> entirely from the existing session lifecycle + `session_metrics` upstream message. See
> `docs/stream-perf/contract-amendment.md`.

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
  more than the session's held reservation), `home_in_use` (409, *P5-01* —
  the caller already has a live session of this managed-home app; the per-(user, app) home is
  single-writer), `profile_ineligible` (409, *AS10-03* — a user-facing launch selected a stream
  profile that is `ineligible` for the caller's device, or a non-user-facing profile without an
  admin/explicit-override bypass), `restart_required` (409, *host-runtime-settings* — a `PATCH` to
  `/v1/admin/hosts/{id}/settings` changed a restart-class knob while the host has live sessions
  but `restart_confirm` was not `true`; includes `{ "live_sessions": N }` in the error body),
  `rate_limited` (429), `no_host_available` (503,
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
| `DELETE /v1/apps/{id}` | **admin** | *(admin-delete)* remove an app from the catalog — refuse-if-in-use |
| `GET /v1/hosts`, `GET /v1/hosts/{id}` | **admin** | host/capacity oversight |
| `POST /v1/hosts/{id}/drain`, `POST /v1/hosts/{id}/uncordon` | **admin** | *(P3-01)* host lifecycle — cordon a host out of service / return it |
| `DELETE /v1/hosts/{id}` | **admin** | *(admin-delete)* forget an offline host — refuse-if-online-or-in-use |
| `GET /v1/apps`, `GET /v1/apps/{id}` | user / public | the public library (list is unauthenticated) |
| `GET /v1/sessions/{id}`, `GET /v1/sessions`, `DELETE /v1/sessions/{id}` | **owner or admin** | resource-ownership check (`403` otherwise), not a blanket admin gate |
| `POST /v1/sessions/{id}/swap` | **owner or admin** | *(P2-02)* same ownership check as `DELETE` |
| `POST /v1/sessions/{id}/stats` | **owner or admin** | *(P4-01)* the client posts its own session's browser telemetry — same ownership check as `DELETE` |
| `GET /v1/admin/sessions/{id}/metrics` | **admin** | *(P4-01)* per-session telemetry read (oversight) |
| `POST /v1/me/devices` | user (self) | *(P4-01)* upsert the caller's own device capability; owner is the bearer identity, never a body field |
| `GET /v1/me/devices` | user (self) | *(AS10-08)* read the caller's own latest device capability record; owner is the bearer identity |
| `GET /v1/me/profiles` | user (self) | *(AS10-02)* stream profile eligibility + recommendation for the caller's device; advisory, owner is the bearer identity |
| `GET /v1/admin/storage/homes` | **admin** | *(P5-01)* list managed homes (storage oversight) |
| `DELETE /v1/admin/storage/homes/{id}` | **admin** | *(P5-01)* tombstone a home for GC |
| `GET /v1/me/storage` | user (self) | *(P5-01)* the caller's own per-app storage usage |
| `POST /v1/me/password` | user (self) | *(CP-01)* change the caller's own password; subject is the bearer identity, never a body field. Revokes all active tokens on success — client must re-authenticate |
| `GET /v1/admin/config/catalog` | **admin** | *(host-runtime-settings)* read the knob catalog |
| `GET /v1/admin/hosts/{id}/settings` | **admin** | *(host-runtime-settings)* read a host's resolved settings + overrides |
| `PATCH /v1/admin/hosts/{id}/settings` | **admin** | *(host-runtime-settings)* update per-host overrides |
| `GET /v1/admin/sessions/{id}/trace`, `.../trace/window`, `.../trace/metrics`, `.../trace/events` | **admin** | *(ST-01)* the bounded recent session trace (samples + events + clock) |
| `GET /v1/admin/sessions/{id}/diagnostic-bundle` | **admin** | *(ST-01/ST-06)* the assembled bundle — metadata + clock + aligned series + events + derived windows + classifier verdict |
| `POST /v1/admin/sessions/{id}/trace/annotations` | **admin** | *(ST-01)* an operator annotation marker on the trace timeline |
| `POST /v1/sessions/{id}/trace/events` | **owner or admin** | *(ST-01)* the client posts its own session's browser trace events — same ownership check as `POST .../stats`; `202` on accept |
| `POST /v1/sessions/{id}/trace/clock` | **owner or admin** | *(ST-01/ST-05)* the client posts its own session's client↔host clock-offset estimate — same ownership check as `POST .../stats`; `202` on accept |
| `GET /v1/admin/hosts/{id}/encoder-certification` | **admin** | *(SPT-05)* read a host's encoder-certification verdicts (latest per configuration) |
| `POST /v1/admin/hosts/{id}/encoder-certification/runs` | **admin** | *(SPT-05)* open a certification run (reserve the per-host lock, return the cell plan) |
| `POST /v1/admin/hosts/{id}/encoder-certification/cells` | **admin** | *(SPT-06)* launch one pinned bench cell |
| `POST /v1/admin/hosts/{id}/encoder-certification/cells/{sid}/finalize` | **admin** | *(SPT-06)* derive the verdict from real agent metrics, upsert + teardown |
| `POST /v1/admin/hosts/{id}/encoder-certification/runs/{run_id}/complete` | **admin** | *(SPT-06)* close the run (release the per-host lock) |
| `GET /v1/admin/hosts/{id}/encoder-certification/runs/{run_id}` | **admin** | *(SPT-05)* poll a run's status/progress |
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
{ "email": "a@b.com", "password": "<plaintext, TLS only>",
  // optional, native client only (P9-01); web/legacy clients omit them:
  "client_version": "1.2.0", "contract_version": "p9-01" }
// 200
{
  "access_token": "<opaque bearer>",
  "token_type": "Bearer",
  "expires_at": "2026-06-02T12:00:00Z",
  "user": { "id": "<uuid>", "email": "a@b.com", "username": "ada", "role": "user" },
  // optional version advisory (P9-01); both absent ⇒ no floor advertised:
  "min_client_version": "1.0.0", "latest_client_version": "1.3.0"
}
```
The opaque token is stored **hashed** in `auth_tokens` and returned in plaintext exactly once
here. `401 invalid_credentials` on bad password / unknown email / disabled account (no
distinction, to avoid user enumeration).

> *(P9-01, additive) `client_version` / `contract_version` are **optional** request fields a
> native client sends to identify itself; a request without them (every web / Phase-1 client)
> behaves exactly as before. `min_client_version` / `latest_client_version` are **optional**
> response advisories. A client below `min_client_version` SHOULD be hard-gated and one below
> `latest_client_version` soft-warned — but the **enforcement rule is defined in P9-08
> (#236)**; this amendment only specifies the fields. `client_version` is a semver string the
> client owns; `contract_version` is the `protocol/` version tag the client built against.*

### `POST /v1/auth/logout`
Revokes the presented bearer token (`auth_tokens.revoked_at = now()`). `204`. Idempotent.

### `GET /v1/me`
```json
// 200
{ "user": { "id": "<uuid>", "email": "...", "username": "...", "role": "user", "created_at": "..." } }
```

> Token refresh and "list my active tokens/sessions" are deliberate Phase-2 additions; the
> `auth_tokens` table already supports them. Not in the frozen Phase-1 surface.

### `POST /v1/me/password`
```json
// request — RequireAuth; the subject is the bearer identity, never a body field
{ "current_password": "<plaintext, TLS only>", "new_password": "<plaintext, TLS only>" }
// 204 — no body
```
*(CP-01, additive)* Self-service change-password. The caller proves possession of the account by
supplying `current_password`, verified against the stored argon2id hash exactly as login does. On
success the password hash is rotated, **all** of the user's active bearer tokens are revoked (log
out everywhere), and **`204`** is returned with no body — the client **must** re-authenticate with
the new password (the bearer used for this call is invalid immediately after the `204`). Errors:
`401 invalid_credentials` if `current_password` is wrong; `400 validation_failed` if `new_password`
fails the password-strength rule (the same length bounds as `/v1/auth/register`). No new field on
any existing shape; reuses the login password-verify path, the registration strength rule, and the
existing token-revocation path. Gated by `RequireAuth` like the other `/v1/me` routes.

---

## Library

### `GET /v1/apps`
Lists enabled apps (the library the user can launch).
```json
// 200
{ "items": [
    { "id": "<uuid>", "name": "Foo", "description": "...", "cover_url": "https://...",
      "default_width": 1920, "default_height": 1080, "default_fps": 60, "default_bitrate_kbps": 15000,
      "default_profile_id": "1440p60", "profile_policy": "prefer",
      "display_stream": { "width": 2560, "height": 1440, "fps": 60, "bitrate_kbps": 20000 } }
  ], "next_cursor": null }
```
`display_stream` is the user-facing stream advertised in the library: it resolves the
app/global stream-profile policy when available, and otherwise falls back to the legacy
`default_*` stream fields. `runtime_spec` and resource defaults are **not** exposed to
clients (agent-internal / scheduler-internal). Disabled apps are omitted.

### `GET /v1/apps/{id}`
Single app, same fields as a list item. `404` if absent or disabled.

> Creating/editing apps and managing hosts (`GET/POST/PATCH /v1/apps`, `GET /v1/hosts`) is the
> **admin** surface (`role=admin`), built in P1-3 against the same `schema.md`. The read shapes
> above are the public subset; admin write shapes are P1-3's to define within this contract's
> conventions (no frozen-interface change — they're additive, admin-gated).

### `DELETE /v1/apps/{id}` — remove an app *(admin-delete)*
Admin-only. Removes an app from the catalog entirely.
```json
// 204 No Content — app removed; its terminal session history is purged
```
- **Refuse-if-in-use.** `409 conflict` if any **non-terminal** session
  (`pending`/`assigned`/`starting`/`running`/`stopping`) references the app — disable it
  (`PATCH … {"enabled": false}`) and let live sessions drain first.
- **Cascade.** On success the app row is deleted; its **terminal session history** (and those
  sessions' metrics/tokens) cascade away, and its managed homes are tombstoned for GC. FK policy in
  `schema.md` (`sessions.app_id` → `ON DELETE CASCADE`, `user_homes.app_id` → `ON DELETE SET NULL`,
  migration `0014`).
- **Errors:** `404 not_found` (no such app); `409 conflict` (in use by a non-terminal session).

---

## Sessions (launch + lifecycle)

### `POST /v1/sessions` — launch
Creates a session, runs the scheduler (pick host+GPU, reserve VRAM+encode slots), tells the
agent to assign+start, and returns the **signaling coordinates** the client connects with. This
is the single hinge to `signaling.md`.
```json
// request — app_id required; profile_id is the normal user-facing selector
// (AS10-03); the explicit stream block remains available for admin/debug/back-compat.
{
  "app_id": "<uuid>",
  "profile_id": "1080p60",
  "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000 }
}
// 201
{
  "session": {
    "id": "<uuid>", "app_id": "<uuid>", "state": "assigned",
    "profile_id": "1080p60",
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

#### Launch by profile (AS10-03)
> *Additive amendment — adds an optional `profile_id` request field and an optional
> `profile_id` response field (persisted on the session, `schema.md`); changes no existing
> shape. Wants Opus + human sign-off per the frozen-contract rule.*

`profile_id` is the normal user-facing way to launch: it names a profile from the AS10-01
catalog (the user-facing ids returned by `GET /v1/me/profiles`) and the control plane resolves
it to concrete `width`/`height`/`fps`/`bitrate_kbps`/`h264_profile`/`playout0_ms`. Rules:

- **Resolution & FPS are fixed for the session.** In-session adaptation (ABR, AS10-04+) changes
  bitrate/quality only — never the resolution or frame rate.
- **H.264 profile is negotiated for the transport.** A profile's nominal `h264_profile` (e.g.
  `high`) is its preference for a capable client; the **browser (WebRTC) receiver rejects High**
  (verified on both VA and NVENC), so for today's browser-only transport the launch resolves
  `h264_profile` down to the **constrained-baseline** floor. The selected `profile_id` still
  records the rung. A future native client (AS10-12) — or an explicit `stream.h264_profile`
  override — lifts it. (This is why the resolved `stream.h264_profile` above reads
  `constrained-baseline` even for a `high`-nominal profile.)
- **Eligibility gate.** A user-facing launch naming a profile that is **`ineligible`** for the
  caller's device (the same verdict `GET /v1/me/profiles` returns) is rejected with
  **`409 profile_ineligible`** and no session row persists. A `risky` profile is allowed. A
  caller with no/stale probe is never rejected for eligibility (unknown → allow).
- **Unknown `profile_id`** → `400 validation_failed`.
- **Overrides & bypass (back-compat / admin / debug).** An explicit `stream` block beats the
  profile field-by-field, and its presence — or an **admin** caller — bypasses the eligibility
  gate (and allows selecting a non-user-facing debug/internal profile such as `720p30`). With no
  `profile_id`, behaviour is exactly as before (AS-02 tier selection capped by the app defaults).
- The selected `profile_id` is **persisted on the session** and echoed in every session body
  (launch + `GET`); it is `null` for a legacy/tier/override launch. The resolved concrete values
  live in `stream`; full profile metadata is at `GET /v1/me/profiles` keyed by this id.

#### `client_type` — Path-B identity binding (SPT-05)
> *Additive amendment — one optional top-level string request field; changes no existing shape
> (absent ⇒ today's behavior exactly). No schema column, no frozen `protocol/` contract touched
> beyond this file.*

`POST /v1/sessions` accepts an optional `"client_type": "native" | "browser"` field (absent ⇒
`"browser"`; parsed **leniently** — any value other than the exact string `"native"` is treated as
`"browser"`, never rejected). Its **only** effect is the SPT Path-B H.264 profile lift: the lift to
a rung's preferred `main`/`high` profile fires only when the launch request itself declares
`client_type:"native"` **and** the account's latest stored device probe is native and can decode
that profile. A `"browser"`/absent declaration keeps the constrained-baseline floor regardless of
what probe is stored — this binds the lift to the **launching client's own declaration** rather
than the account's last-seen probe, so a native session can never poison a subsequent browser
launch on the same account with a profile Chrome cannot decode. The web SPA does not send this
field; a native client (Phase 9) must send `client_type:"native"` to receive the lift.

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
    "profile_id": "1080p60",
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

### `DELETE /v1/hosts/{id}` — forget a host *(admin-delete)*
Removes a host record entirely — the operator's way to clear a **dead/decommissioned** host from
the fleet. Intended for an `offline` host (its agent is gone for good); a host that is merely
draining or temporarily offline does not need forgetting, since it returns to service on its own
when its agent reconnects. A forgotten host that later re-enrolls comes back as a **fresh** record.
```json
// 204 No Content — host record, its GPUs, and its terminal session history are removed
```
- **Refuse-if-in-use.** The host must be **offline** (its agent is **not** connected) **and** hold
  **no non-terminal session**. The online check is against the live agent connection, not only the
  stored `status`, so a host that reconnected since the page loaded is not silently deleted.
- **Cascade.** On success the host row is deleted; its `gpus` rows and its **terminal session
  history** (with the sessions' metrics/tokens) cascade away, and its managed homes are tombstoned
  for GC (the backing store died with the host). FK policy in `schema.md` (migration `0014`).
- **Errors:** `404 not_found` (no such host); `409 conflict` (host is online, or still holds a
  non-terminal session). The body distinguishes the two.

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
// request — a small batch of recent samples (client clock, ms)
{ "client": "native",   // optional (P9-01): "browser" (default) | "native"; sets session_metrics.source
  "samples": [
    { "ts_unix_ms": 1735689600000,
      "metrics": { "fps": 59.6, "bitrate_kbps": 14710, "rtt_ms": 12, "jitter_buffer_ms": 28,
                   "decode_ms": 1.4, "packets_lost": 0, "frames_dropped": 0,
                   // presentation pacing (#108, always-on; schema.md dictionary):
                   "present_fps": 59.8, "present_interval_sd_ms": 2.9,
                   "present_interval_p95_ms": 18.0, "playout_target_ms": 100,
                   // glass-to-glass budget (always-on via the abs-capture-time RTP extension):
                   "glass_to_glass_ms": 71, "network_pacing_ms": 7.5,
                   "decode_display_ms": 30.9 } }
  ] }
// 202 — accepted (no body); samples are written with source = the `client` value (default 'browser')
```
- Each sample becomes a `session_metrics` row with `source =` the request's `client`
  (default `'browser'`; `'native'` for the native client — P9-01; an unknown `client` value
  is rejected, not silently coerced). The glass-to-glass keys
  (`glass_to_glass_ms`, `network_pacing_ms`, `decode_display_ms`) are **always-on** — glass-to-
  glass is measured from the `abs-capture-time` RTP header extension (no per-frame overlay,
  hardware-independent). *(Supersedes the removed deep-trace toggle / pixel-overlay instrument.)*
- Best-effort: a malformed sample is dropped, not fataled. Accepting telemetry never affects
  session state.

### `GET /v1/admin/sessions/{id}/metrics` — read per-session telemetry (admin)
Returns the **bounded recent window** of telemetry for one session, both sources, newest first.
Admin-only (`403` before any lookup per §Authorization; `404` for an unknown session to an admin).
```json
// 200 — paginated, newest first; ?limit=&cursor=&source=agent|browser|native optional
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

> **Amendment — AS10-12 (native-client capability/metrics/cert contract), additive, requires sign-off.**
> A **future native client** reports its capabilities, playback metrics, and per-profile
> certification through the **same `POST /v1/me/devices`** endpoint, into the **same
> `user_devices.capabilities` JSONB column**, as an **additive superset of the web probe**
> documented below. The native report's full wire schema is the new **frozen** contract
> `protocol/native-client.md`. **No endpoint, request/response shape, status code, or DB
> migration changes** — the server additively validates a `report_version` integer (dropped
> if non-integer) and passes the native sub-objects (`os`, `decode`, `audio`, `input`,
> `metrics`, `health`) through within the existing sanitizer bounds (depth 8 / width 64 /
> string 512); the flat `codecs` map stays the eligibility surface (the richer per-codec
> `decode{}` is forward-data, not a new gate). An existing web blob is, unchanged, a valid
> (minimal) native-family report and stores byte-identically. The 8 KB body cap
> (`maxDeviceBodyBytes`) is unchanged — a worst-case native payload fits with headroom (see
> `protocol/native-client.md` §Server handling). No live consumer in AS10-12 — stored
> forward-data, like the web probe.

#### Extended capability / certification record (AS10-08)

> *Additive amendment (AS10-08). The `capabilities` JSON column is deliberately
> **schema-free** (`schema.md` `user_devices`) — this section documents the
> richer shape the web client now posts, but **no existing field, request, or
> response shape changes**, and **no DB migration** is involved. The server
> stores the blob opaquely after **sanitizing** it (bounded string lengths,
> validated `client_type`, clamped numeric ranges, structural junk dropped) and
> server-stamping `measured_at`. Fields the server does not model round-trip
> verbatim within those bounds.*

The probe evolved from a connection/decode probe into a **client performance
certification record**. The extended (and entirely optional/best-effort) fields:

```json
{ "device_key": "<opaque id>",
  "capabilities": {
    "client_type": "web",
    "codecs": {"h264":true,"hevc":false,"av1":false,"vp9":true},
    "max_decode_height": 2160,
    "bandwidth_kbps": 48000,
    "rtt_ms": 12,
    "browser":  { "name": "Chrome", "version": "126.0.0.0" },
    "platform": "macOS",
    "display":  { "width": 2560, "height": 1440, "refresh_hz": 120 },
    "features": { "jitter_buffer_target": true, "playout_delay_hint": true,
                  "pointer_lock": true, "coalesced_pointer_events": true, "gamepad": true },
    "profiles": {
      "<profile-id>": {
        "h264_profile_decoded": "constrained-baseline",
        "decode_pass": true, "present_pass": true,
        "decode_ms": 1.4, "present_fps": 59.8, "dropped_ratio": 0.0,
        "measured_at": "<RFC3339>"
      }
    }
  } }
```
- `client_type` is validated against a known set (`web`/`native`); anything else
  is normalised to `web` server-side.
- `browser`/`platform`/`display`/`features` are **best-effort feature detection**;
  any may be absent. `display.refresh_hz` is omitted when not measurable.
- `profiles` is a **per-profile certification map**, keyed by stream-profile id
  (`GET /v1/me/profiles`). Its purpose is the AS10-03 finding that **browser codec
  acceptance is not proof of hardware decode** — `h264_profile_decoded` records the
  H.264 profile that *actually decoded* (e.g. `constrained-baseline`), with
  decode/presentation pass-fail and basic playback metrics. In AS10-08 it is the
  **shape only** (populated empty / where trivially available); the harness that
  fills it from historical playback is AS10-10.

### `GET /v1/me/devices` — read the caller's latest device capability record (AS10-08)

> *Additive, user-self amendment (AS10-08). New endpoint; changes no existing
> request/response shape, status code, or frozen endpoint. Follows the same
> user-self pattern as `POST /v1/me/devices` / `GET /v1/me/profiles`.*

Returns the caller's **most-recently-seen** device row (by `last_seen_at`),
including the stored capability record verbatim. Owner is the **bearer identity**;
a user can only read their own devices. `404 not_found` when the caller has no
device row yet.
```json
// 200
{ "device": {
    "id": "<uuid>", "device_key": "<opaque id>",
    "first_seen_at": "...", "last_seen_at": "...",
    "capabilities": { "...": "the sanitized, measured_at-stamped blob, verbatim" } } }
// 404 — caller has no device record
{ "error": { "code": "not_found", "message": "no device record for caller" } }
```

---

### `GET /v1/me/profiles` — stream profile eligibility + recommendation (AS10-02)

> *Additive, user-self amendment (AS10-02). New endpoint; changes no existing
> request/response shape, status code, or frozen endpoint. Follows the same
> user-self pattern as `POST /v1/me/devices` / `GET /v1/me/storage`. Wants Opus +
> human sign-off per the frozen-contract rule.*

Returns the stream profile catalog (AS10-01, user-facing profiles only) annotated
with an **eligibility verdict** and **reason codes** for the caller's device, plus a
**recommended** profile. Owner is the **bearer identity**. **Advisory only** — this does
**not** change what a launch does (the launch path still flows through the legacy tier
ladder until AS10-03 migrates it). The verdict is computed from the caller's latest
**fresh** `user_devices` probe (≤ 30 days; a stale/absent probe yields a conservative,
low-confidence recommendation).

Each profile is classified `eligible` (passes every hard check and has bandwidth
headroom), `risky` (launchable but with a soft concern — thin headroom, browser-client
risk, or an unconfirmable high-refresh display), or `ineligible` (fails a hard check).
`recommended_id` is the highest fully-eligible profile (catalog order); with no usable
probe it is the conservative default (`1080p60`) and `confidence` is `low`.

```json
// 200
{
  "recommended_id": "1080p60",
  "confidence": "high" | "low",
  "notes": [ { "code": "probe_missing", "message": "..." } ],
  "profiles": [
    {
      "id": "1080p60", "display_name": "1080p · 60 FPS",
      "width": 1920, "height": 1080, "fps": 60,
      "codecs": [ { "codec": "h264", "status": "launchable" },
                  { "codec": "hevc", "status": "future" },
                  { "codec": "av1",  "status": "future" } ],
      "h264_profile": "high",
      "nominal_bitrate_kbps": 12000, "min_offer_bandwidth_kbps": 14400,
      "recommended_offer_bandwidth_kbps": 18000, "headroom_factor": 1.5,
      "abr_floor_kbps": 4000, "max_startup_rtt_ms": 0, "min_decode_height": 1080,
      "high_refresh_display": "none", "hardware_encoder_required": false,
      "browser_client": "recommended", "playout0_ms": 50, "visibility": "user",
      "eligibility": "eligible",
      "reasons": [ { "code": "bandwidth_too_low", "message": "..." } ]
    }
  ]
}
```
- **Reason codes** (stable, append-only): `bandwidth_too_low`, `rtt_too_high`,
  `decode_height_too_low`, `codec_not_supported`, `host_encoder_not_supported`,
  `display_refresh_unknown`, `browser_playout_unsupported`,
  `historical_client_performance_failed`, `probe_missing`, `probe_stale`.
  Network/decode/codec checks are skipped for an unmeasured probe field (unknown → allow);
  host-encoder and historical-failure inputs are not yet populated in AS10-02 (reserved for
  later issues) and are part of the contract from the start.
- Debug/internal profiles (`720p30`) are **never** returned.

---

## Session trace (Observability v2, ST-01)
> *Additive, admin-gated amendment. New endpoints; no change to any existing shape, status code,
> or endpoint. Reads are admin-gated (`RequireAuth → RequireAdmin`, `403` before any resource
> lookup); the two client ingest endpoints are owner-or-admin, identical to `POST .../stats`.
> Telemetry is observability, never access control and never a session-state authority. Backing
> DDL: `schema.md` §`session_trace_events` / §`session_trace_clock`, migration `0016`. Format
> reference: `docs/session-trace/trace-format.md`.*

A **trace** is the read-time union of everything already keyed by `session_id`: the existing
`session_metrics` samples, the new discrete `session_trace_events`, and the optional
`session_trace_clock` offset. There is no separate `trace_id` and no new retention mechanism —
traces reuse the `session_metrics` rolling-window + terminal-prune + `ON DELETE CASCADE` model.
Every read below is a **bounded window**: default the recent 5 min, clamped to `[2, 10]` min.

### `POST /v1/sessions/{id}/trace/events` — client posts trace events
The browser reports its own discrete events for **its** live session. Owner-or-admin (same
ownership rule as `POST /v1/sessions/{id}/stats`; non-owner `403`, unknown id `404`). Returns
**`202`** (accepted, no body). Untrusted input: body bounded to **≤ 64 events per request and
≤ 32 KB**, and **rate-limited** (`429 rate_limited`); an event whose `type` is not in the
`trace-format.md` §3.3 v1 allow-list is **dropped, not stored**.
```json
// request — a small batch of recent client events (client clock, ms)
{ "client": "browser",   // optional: "browser" (default) | "native"
  "events": [
    { "ts_unix_ms": 1735689600100, "type": "playout.changed",
      "payload": { "from_ms": 100, "to_ms": 67, "reason": "degrade" } },
    { "ts_unix_ms": 1735689600250, "type": "client.visibility_changed",
      "payload": { "hidden": true } }
  ] }
// 202 — accepted (no body); known-type events stored source='browser', unknown types dropped
```
Best-effort: a malformed event is dropped, not fataled. Accepting trace events never affects
session state.

### `POST /v1/sessions/{id}/trace/clock` — client posts a clock-offset estimate
The browser reports its client↔host clock-offset estimate (from the deep-trace ping/pong sync)
for **its** live session. Owner-or-admin (same ownership rule as the trace-events POST). Returns
**`202`** (accepted, no body). A client with no stable estimate simply never posts here — absence
of a `session_trace_clock` row means **unmeasured**, never a synthesized `client_offset_ms: 0`
(`trace-format.md` §4, no false precision).
```json
// request
{ "client_offset_ms": -3.2, "uncertainty_ms": 1.8 }
// 202 — accepted (no body); upserts the session's session_trace_clock row
```
A malformed/non-finite (`NaN`/`Infinity`) value is dropped, not stored.

### Trace reads (admin)
All admin reads accept `?limit=&cursor=` pagination (existing convention) and the window above.

- **`GET /v1/admin/sessions/{id}/trace`** — the bounded recent trace: clock + the taxonomy series
  + events. `404` for an unknown session (to an admin).
  ```json
  // 200
  { "session_id": "<uuid>", "window": { "from_ms": 1735689300000, "to_ms": 1735689600000 },
    "clock": { "client_offset_ms": -3.2, "uncertainty_ms": 1.8, "measured_at": "..." },
    "series": { "encoder.encode_ms": [ { "ts_unix_ms": 1735689600000, "v": 4.6 } ],
                "client.present_interval_sd_ms": [ { "ts_unix_ms": 1735689600100, "v": 2.9 } ] },
    "events": [ { "source": "agent", "ts_unix_ms": 1735689600000, "type": "abr.retarget",
                  "payload": { "from_kbps": 14000, "to_kbps": 11000, "reason": "gcc_downshift" } } ] }
  ```
- **`GET /v1/admin/sessions/{id}/trace/window?from=&to=`** — same shape, explicit window
  (`from`/`to` are `ts_unix_ms`); clamped to the 10-min max.
- **`GET /v1/admin/sessions/{id}/trace/metrics?names=encoder.encode_ms,abr.setpoint_kbps`** — only
  the requested taxonomy series (`{ "series": { … } }`); unknown names omitted.
- **`GET /v1/admin/sessions/{id}/trace/events?types=abr.retarget,playout.changed`** — only the
  requested event types (`{ "events": [ … ] }`); all types if `?types=` omitted.
- `clock` is **either** `{client_offset_ms, uncertainty_ms, measured_at}` **or**
  `{"unmeasured": true}` — never an offset-0 default.
- `series` values are normalized-at-read from existing `session_metrics` JSONB (taxonomy v1,
  `trace-format.md` §2); no new sample write path.

### `POST /v1/admin/sessions/{id}/trace/annotations` — operator annotation (admin)
Admin-only. Records an operator marker on the trace timeline (e.g. "flipped
`QUASAR_ABR_FLOOR_KBPS` here" during an A/B). Returns `201`. Stored as a `session_trace_events`
row with a reserved type (`operator.annotation`), so it appears inline on the timeline.
```json
// request
{ "ts_unix_ms": 1735689600500, "label": "flipped abr floor to 4000", "tags": ["ab-test"] }
// 201 — { "id": "<uuid>" }
```

### `GET /v1/admin/sessions/{id}/diagnostic-bundle` — the money endpoint (admin)
Admin-only (`403` before lookup; `404` unknown session). Assembles, for a bounded window (default
5 min, clamp `[2,10]` min), everything needed to answer "network, encoder, or client?" in one
call. Built by joining the existing `session_metrics` JSONB (normalized to taxonomy v1) +
`session_trace_events` + `session_trace_clock`. Includes a v0 **observational** classifier verdict
(no automatic action).
```json
// 200
{
  "trace": { "session_id": "<uuid>", "host_id": "<uuid>", "profile_id": "1080p60",
             "started_at": "2026-06-26T12:00:00Z", "ended_at": null },
  "window": { "from_ms": 1735689300000, "to_ms": 1735689600000 },
  "clock": { "client_offset_ms": -3.2, "uncertainty_ms": 1.8, "measured_at": "..." },
  "series": {
    "encoder.encode_ms":   [ { "ts_unix_ms": 1735689600000, "v": 4.6 } ],
    "encoder.fps":         [ { "ts_unix_ms": 1735689600000, "v": 59.8 } ],
    "abr.setpoint_kbps":   [ { "ts_unix_ms": 1735689600000, "v": 11000 } ],
    "transport.rtt_ms":    [ { "ts_unix_ms": 1735689600100, "v": 28 } ],
    "transport.packets_lost": [ { "ts_unix_ms": 1735689600100, "v": 14 } ],
    "client.present_interval_sd_ms": [ { "ts_unix_ms": 1735689600100, "v": 12.4 } ],
    "client.glass_to_glass_ms":      [ { "ts_unix_ms": 1735689600100, "v": 71 } ]
  },
  "events": [
    { "source": "agent",   "ts_unix_ms": 1735689600000, "type": "abr.retarget",
      "payload": { "from_kbps": 14000, "to_kbps": 11000, "reason": "gcc_downshift" } },
    { "source": "browser", "ts_unix_ms": 1735689600100, "type": "playout.changed",
      "payload": { "from_ms": 50, "to_ms": 75, "reason": "degrade" } }
  ],
  "derived_windows": {
    "hitches":                   [ { "from_ms": 1735689600050, "to_ms": 1735689600120, "present_interval_sd_ms": 18.6 } ],
    "abr_downshifts":            [ { "ts_unix_ms": 1735689600000, "from_kbps": 14000, "to_kbps": 11000 } ],
    "encoder_saturation":        [ { "from_ms": 1735689600000, "to_ms": 1735689600200, "encode_ms_p95": 20.1 } ],
    "likely_network_congestion": [ { "from_ms": 1735689600050, "to_ms": 1735689600200, "packets_lost_delta": 14, "rtt_ms_p95": 60 } ]
  },
  "classifier": {
    "verdict": "likely_network_congestion",
    "evidence": [ "gcc downshift coincident with packets_lost rise and rtt p95 60ms",
                  "encode_ms p95 below encoder ceiling; host fps steady",
                  "client tab not hidden during the window" ]
  }
}
```
- `classifier.verdict` is **observational only** — one of `likely_encoder_saturation` /
  `likely_network_congestion` / `likely_client_presentation_limit` / `nominal` /
  `indeterminate_client_hidden` / `unknown`, with its `evidence` logged. **No automatic action.**
  `nominal` = healthy session (no negative signal AND the tab was not hidden);
  `indeterminate_client_hidden` fires when the *only* reason no verdict was reached is the
  `client.is_hidden` guard; `unknown` is reserved for genuine insufficient-data windows.

---

## Host encoder certification (Stream Perf Tuning Phase C, SPT-05)
> *Additive, admin-gated amendment. New endpoints; no change to any existing shape, status code,
> or endpoint. `403` before any resource lookup; `404` for an unknown host/run to an admin;
> `409 conflict` if a run is already in flight on that host, or the host is not `online`. Backing
> DDL: `schema.md` §`host_encoder_certification`, migration `0018`. `agent-api.md` is unchanged —
> certification composes entirely from the existing session lifecycle + `session_metrics` upstream
> message. See `docs/stream-perf/contract-amendment.md` §B.*

Records the measured, sustainable encode envelope of a concrete (host, GPU, encoder, profile,
bench-bitrate) configuration, so the scheduler can avoid default-starting a profile a host cannot
hold in real time. **As-built (SPT-06): script-orchestrated, not control-plane-autonomous** — a
bench session needs a real WebRTC peer to drive frame flow, so the harness
(`deploy/run-spt06-certify.sh`) supplies a Chrome-for-Testing peer and drives the loop: open a run
→ per-cell (launch → CFT peer drives → finalize) → complete. The control plane still owns the
measurement-to-verdict logic (read `session_metrics` → derive verdict → upsert); it does not spawn
a browser itself.

### `GET /v1/admin/hosts/{id}/encoder-certification` — read verdicts (admin)
Returns the **latest** `host_encoder_certification` row per configuration for the host (the
upsert-latest table means "latest" is just "the row"). Optional filters
`?gpu_index=&encoder=&profile_id=` narrow the set; `?max_age_s=` treats anything older as omitted
(the staleness horizon below).
```json
// 200
{ "host_id": "<uuid>",
  "certifications": [
    { "gpu_index": 0, "encoder": "va", "profile_id": "1080p60",
      "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 8000,
      "verdict": "unsafe",
      "encode_ms_p50": 19.2, "encode_ms_p95": 20.1, "encode_ms_max": 24.8,
      "output_fps": 45.3, "drop_rate": 0.0, "live_write_stable": true,
      "sample_window_ms": 20000, "sample_count": 900, "agent_version": "0.6.0",
      "measured_at": "2026-06-27T03:00:00Z", "updated_at": "2026-06-27T03:00:00Z" }
  ] }
```
An uncertified host (no rows) returns `{ "host_id": "<uuid>", "certifications": [] }` — **not** a
`404` (the host exists; it simply has no verdicts yet). The scheduler treats "no row" exactly as
"uncertified" (no cap).

### Run lifecycle (admin, script-orchestrated)
**1. `POST .../runs` — open a run.** Body optionally scopes rungs/encoders/bitrates; absent ⇒ the
host's default matrix. Reserves the per-host lock, returns the cell plan:
```json
// request (all fields optional)
{ "gpu_index": 0, "encoder": "va", "profiles": ["1080p60","1080p45","720p60"],
  "bitrates_kbps": [4000,6000,8000,12000] }
// 202 — run opened
{ "run_id": "<uuid>", "host_id": "<uuid>", "status": "running", "started_at": "...",
  "cells": [ { "profile_id": "1080p60", "bitrate_kbps": 8000 } ] }
```
**2. `POST .../cells` — launch one cell.** Body `{run_id, profile_id, bitrate_kbps}`. Launches a
Diagnostics session pinned to the target host/GPU and returns its `session_id` + signaling token
so the harness can attach a CFT peer. Subject to normal admission control (a fully reserved GPU →
`409`, not a stolen slot).

**3. (harness) connects a CFT peer** to that session so frames flow + decode.

**4. `POST .../cells/{sid}/finalize`** — reads the session's **real** agent metrics
(warmup-skipped window), computes the verdict, **upserts** `host_encoder_certification`, and tears
the session down. **If zero agent samples survived, returns `422` and writes NO row** — it never
fabricates a false `unsafe`.

**5. `POST .../runs/{run_id}/complete`** — closes the run, releases the per-host lock.

### `GET /v1/admin/hosts/{id}/encoder-certification/runs/{run_id}` — poll a run (admin)
```json
// 200 — in progress
{ "run_id": "<uuid>", "host_id": "<uuid>", "status": "running",
  "started_at": "2026-06-27T03:00:00Z", "progress": { "completed": 6, "total": 12 } }
// 200 — done
{ "run_id": "<uuid>", "host_id": "<uuid>", "status": "completed",
  "started_at": "2026-06-27T03:00:00Z", "ended_at": "2026-06-27T03:04:30Z",
  "certified_count": 12, "summary": { "ok": 7, "capped": 2, "unsafe": 3 } }
// 200 — failed (e.g. host went offline mid-run)
{ "run_id": "<uuid>", "host_id": "<uuid>", "status": "failed",
  "error_message": "host_lost during bench" }
```
`status ∈ {running, completed, failed}`.

### Scheduler decision rule — certification-aware session-start cap
> *Behavioral rule in the scheduler / profile-resolution path. Changes **no** request/response
> shape — `POST /v1/sessions` and `GET /v1/me/profiles` are unchanged on the wire.*

At session start, resolving the **default** profile for a launch on a target host (host + GPU +
active encoder already chosen by placement):
1. Look up the latest `host_encoder_certification` row for `(host_id, gpu_index, encoder,
   profile_id)`. A row older than a staleness horizon (default 7 days, tunable) counts as
   **uncertified**.
2. **The cap:** do not default-start a profile whose certified `encode_ms_p95 > 0.70 ×
   budget_ms` (`budget_ms = 1000.0 / fps`) on the target host — equivalently, `verdict='unsafe'`
   is not a default start.
3. **Cap to the next sustainable rung** of the same resolution family, preferring an fps downshift
   over a resolution drop. `sessions.profile_id` still records the **served** rung.
4. **`capped` verdict** ⇒ start the rung but not at the top of its bitrate band, only if
   `live_write_stable = true`; if `false`, treat `capped` as `unsafe` for default-start.
5. **Uncertified / stale / unknown encoder** ⇒ no cap — today's tier-default behavior. The cap only
   ever *lowers* a default, never raises one.
6. **Explicit override wins.** An explicit `profile_id`/width/height/fps on `POST /v1/sessions`, or
   an admin force flag, bypasses the cap entirely.

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

## Host Runtime Settings (host-runtime-settings-admin)

> *Additive, admin-gated amendment. New endpoints; no change to any existing shape, status
> code, or endpoint. All three endpoints are gated by `RequireAuth → RequireAdmin` — a valid
> non-admin bearer is `403` before any resource lookup, consistent with §Authorization.*

The control plane maintains a **server-side knob catalog** and a **per-host sparse override
table** (`schema.md` `host_settings`). A missing `host_settings` row means the host has **no
overrides**, so every knob is left at the **agent's env value** (`QUASAR_*`).

> **#194 amendment — resolution precedence is `stored-override → agent env → catalog default`.**
> The `config_update` pushed to the agent carries only the **sparse overrides** (see
> `agent-api.md`); the agent overlays them on its env baseline. So clearing an override reverts
> the knob to the **agent's env**, not the catalog default, and an un-overridden knob on a GPU
> host keeps its `QUASAR_ENCODER=nvenc` (it is no longer silently forced to the `openh264`
> catalog default). The `resolved` field below is a **display** view (`catalog-default ←
> overrides`); it does **not** see the agent's env, so on a host whose env differs from the
> catalog default it can differ from what the agent actually runs. Surfacing the agent's true
> effective config in `/admin` is a tracked follow-up.

### `GET /v1/admin/config/catalog` — read the knob catalog
Returns the full typed catalog of all tunable knobs. Used by the admin UI to render controls
generically (bool → toggle, int → number input, enum → select, etc.).
```json
// 200
{
  "knobs": [
    {
      "key": "abr_enabled",
      "type": "bool",
      "default": false,
      "nullable": false,
      "class": "live",
      "env_var": "QUASAR_ABR"
    },
    {
      "key": "abr_floor_kbps",
      "type": "int",
      "default": null,
      "min": 1,
      "nullable": true,
      "class": "live",
      "env_var": "QUASAR_ABR_FLOOR_KBPS"
    },
    {
      "key": "encoder",
      "type": "enum",
      "default": "openh264",
      "enum": ["openh264", "va", "nvenc"],
      "nullable": false,
      "class": "restart",
      "env_var": "QUASAR_ENCODER"
    }
  ]
}
```
Each catalog entry carries:
- **`key`** — the knob identifier used in `overrides` JSONB and `config_update` messages.
- **`type`** — one of `bool`, `int`, `float`, `enum`, `string`.
- **`default`** — the catalog default (equals the env-var default; never `null` for non-nullable knobs).
- **`min` / `max`** — optional numeric bounds (present for `int`/`float` knobs that have them).
- **`enum`** — optional array of valid string values (present when `type = "enum"`).
- **`nullable`** — whether `null` is a valid value (a `null` override **clears** the knob — on the agent it reverts to the env baseline; in the `resolved` display view it falls back to the catalog default; a knob whose default is `null` explicitly accepts it).
- **`class`** — `live` (takes effect on the next session launch, no restart) or `restart` (requires an agent restart; see `agent-api.md`).
- **`env_var`** — the corresponding environment variable on the agent container (informational; displayed in the UI).

### `GET /v1/admin/hosts/{id}/settings` — read a host's settings
Returns the resolved (effective) knob values and the sparse override map for one host.
```json
// 200
{
  "resolved": {
    "abr_enabled": true,
    "abr_floor_kbps": null,
    "abr_floor_ratio": 0.3,
    "gop": 60,
    "encoder": "va"
  },
  "overrides": {
    "abr_enabled": true,
    "encoder": "va"
  },
  "pending_restart": false
}
```
- **`resolved`** — the full set of effective knob values for this host: for each catalog knob,
  the override value if one exists, otherwise the catalog default.
- **`overrides`** — the sparse map of only the knobs that differ from their catalog defaults.
  Empty object when no overrides are set.
- **`pending_restart`** — `true` when a restart-class knob was changed (or the `restart`
  command was sent) and the agent has not yet reconnected reporting the new encoder/device.
  Cleared when the agent's next `register` + `config_update` cycle completes.
- **Errors:** `404 not_found` — no host with that id.

### `PATCH /v1/admin/hosts/{id}/settings` — update per-host overrides
Sets, updates, or clears individual knob overrides for a host. Validates the full override map
against the catalog before persisting. Pushes the new resolved config to the agent immediately
as a `config_update` message (`agent-api.md`).
```json
// request — sparse map; a null value clears that override (restores catalog default)
{
  "overrides": {
    "abr_enabled": true,
    "encoder": "va",
    "abr_floor_kbps": null
  },
  "restart_confirm": false
}
// 200 — new effective state
{
  "resolved": { "abr_enabled": true, "encoder": "va", "abr_floor_kbps": null, "...": "..." },
  "overrides": { "abr_enabled": true, "encoder": "va" },
  "restart_triggered": false
}
```
- **Validation.** Unknown keys → `400 validation_failed`. Wrong type → `400 validation_failed`.
  Out-of-range value (beyond `min`/`max`) → `400 validation_failed`. Only the keys in the
  request `overrides` map are touched; unmentioned keys are unchanged.
- **`null` value.** Setting a key to `null` deletes that override, returning the knob to its
  catalog default on the next resolve. A knob whose catalog default is `null` (nullable) may
  be set to `null` explicitly — this is a valid value, not a deletion marker for those knobs;
  the control plane distinguishes "absent from overrides" from "present as null".
- **Restart-class knobs with live sessions.** When any restart-class knob is changed and the
  host has one or more sessions in `state ∈ {assigned, starting, running}`:
  - If `restart_confirm` is absent or `false` → **`409 restart_required`** with body
    `{ "error": { "code": "restart_required", "message": "...", "live_sessions": N } }`.
    No override is persisted and the agent is not contacted.
  - If `restart_confirm: true` → the overrides are persisted, `restart_triggered: true` is
    returned, `pending_restart` is set on the host row, the `restart` command is sent to the
    agent (`agent-api.md`), and the agent exits; its container restart policy brings it back.
    Running sessions are **not** forcibly stopped — the restart command races gracefully with
    the container lifecycle; operators should drain first (`POST /v1/hosts/{id}/drain`) if they
    want a clean cut.
- **Live-class knobs.** Overrides are persisted and a `config_update` message is sent to the
  agent immediately (`agent-api.md`). The agent applies new values to the next session build —
  live sessions are unaffected. `restart_triggered` is `false`.
- **`restart_triggered: true`** in the response means the `restart` downstream message was sent
  to the agent this request; `pending_restart` on the host is now `true`. The caller should
  poll `GET /v1/admin/hosts/{id}/settings` to detect when `pending_restart` clears (agent
  reconnected with updated config).
- **Errors:** `404 not_found` — no host with that id; `400 validation_failed` — bad knob key,
  wrong type, or out-of-range value; `409 restart_required` — as above.

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
