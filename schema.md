# P1-C — Postgres schema (FROZEN)

> **Amendment — P2-01 (resource-governance + admission-control), additive, requires sign-off.**
> Adds one column, `users.max_concurrent_sessions` (`INT NOT NULL DEFAULT 3 CHECK (>= 0)`),
> via migration `0002_user_session_quota`. This is the per-user concurrent-session quota the
> Phase-2 launch path enforces (`control-api.md` `session_quota_exceeded`). It is **purely
> additive** — a new defaulted column, no change to any existing column, type, constraint, or
> to the frozen `0001` migration — so it falls under the documented `control-api.md
> §Authorization` additive-extension exception. No other table changes. The schema's resource
> model (`gpus` derived availability, `sessions` reservation columns) is **unchanged**: P2-01
> defines the *enforcement semantics* over the existing columns, it does not add reservation
> storage. See `docs/phase2/P2-01-contract-resource-governance.md`.

> **Amendment — P2-02 (launcher↔game swap), no DDL change.** The app-swap operation
> (`control-api.md` `POST /v1/sessions/{id}/swap`) introduces **no new column and no new session
> state**. A swap rides **within** the `running` state as `state_detail = 'swapping'` (a
> convention over the existing free-text `sessions.state_detail` column — the `state` CHECK set is
> untouched), because a swap does not change the session's scheduled/reserved status, so no reader
> should have to handle a new top-level state. On a successful swap the control plane updates the
> existing `sessions.app_id` column to the new app (a row write, not a schema change); a failed
> swap that rolled back leaves `app_id` unchanged. The swap must fit within the existing
> `reserved_vram_mb` / `reserved_encode_slots` (no reservation resize in Phase 2). See
> `docs/phase2/P2-02-contract-app-swap.md`.

The persistence model for the control plane. This **replaces Wolf's TOML-as-database**:
all durable control-plane state lives in Postgres (architecture invariant #5 — *State
is external*). The node agent holds no durable state; everything authoritative is here.

This contract is **frozen**: implementation tickets (P1-1, P1-2, P1-3, P1-6, P1-8) build
against it freely, but changing a column, type, or constraint requires Opus + explicit
human sign-off (see `CLAUDE.md`). It co-defines, and must stay consistent with,
`agent-api.md` (P1-A), `control-api.md` (P1-B) and `signaling.md` (P1-D). The session
state machine and the resource model defined here are the single source of truth those
documents refer back to.

## Design stance (end-state at N=1)
Phase 1 runs one host, one GPU, one concurrent session, with generous limits. The schema
is nonetheless shaped for the end state (multi-user, multi-host, multi-GPU, K8s,
resource-governed). Concretely:
- **Hosts and GPUs are first-class rows**, not config. N=1 is just one `hosts` row with one
  `gpus` row. Multi-host (Phase 3) and the K8s DaemonSet model (Phase 4) add rows, not tables.
- **Capacity is per-GPU**, because the real limits are per-GPU: concurrent encode-session
  count (the NVENC/VCN cap) and VRAM. A session reserves against a *specific* GPU. Multi-GPU
  topology (encode on iGPU, render on dGPU — architecture §"Resource governance") is therefore
  expressible from day one.
- **A session is a scheduled, resource-reserved unit.** It carries the host + GPU it was placed
  on and the resources it reserved, even though at N=1 the scheduler's choice is forced.
- **Tokens, never passwords, on the wire** (P1-2). Password hashes (argon2id) and bearer/
  signaling tokens are stored **hashed**; plaintext is never persisted.

## Conventions
- **Postgres ≥ 14.** Primary keys are `UUID` defaulted with `gen_random_uuid()` (core since
  PG13 — no `pgcrypto`/`uuid-ossp` extension needed). **No extensions are required** by this
  schema, to keep the dev/deploy image minimal.
- All timestamps are `TIMESTAMPTZ`, UTC. Audit columns `created_at` / `updated_at` default to
  `now()`; `updated_at` is maintained by the application (or a trigger — see migration 0001).
- **Enums are `TEXT` + `CHECK`**, not Postgres `ENUM` types. Rationale: adding/retiring a
  permitted value is a plain migration (`CHECK` swap) rather than a fragile `ALTER TYPE`; this
  keeps the frozen state machine evolvable through the proper sign-off path without table
  rewrites.
- Money/quantities are integers in explicit units (`*_mb`, `*_kbps`, `*_slots`) — never floats.
- Soft state that the agent reports (capacity, heartbeat) is stored as **last-reported**
  values; it is a cache of node truth, authoritative only for scheduling decisions between
  reports.

## Entity overview
```
users ──< auth_tokens                users 1─< sessions >─1 apps
  │                                    sessions >─1 hosts ─< gpus
  └──< sessions                        sessions >─1 gpus
hosts 1─< gpus
```
Six tables: `users`, `auth_tokens`, `apps`, `hosts`, `gpus`, `sessions`.

---

## `users`
Accounts. Login is by email; `username` is the display handle.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `email` | `TEXT` NOT NULL | unique via `UNIQUE INDEX ON (lower(email))` (case-insensitive, no `citext` extension) |
| `username` | `TEXT` NOT NULL UNIQUE | display handle |
| `password_hash` | `TEXT` NOT NULL | full argon2id PHC string (`$argon2id$v=19$m=...,t=...,p=...$salt$hash`); params live in the string. Never the password. |
| `role` | `TEXT` NOT NULL DEFAULT `'user'` | `CHECK (role IN ('user','admin'))`. Multi-user authz (Phase 2) and admin CRUD (P1-3) need this from the start. |
| `disabled_at` | `TIMESTAMPTZ` NULL | non-null = account deactivated; login + token mint refused. |
| `max_concurrent_sessions` | `INT` NOT NULL DEFAULT `3` | *(P2-01, migration 0002)* per-user cap on simultaneously-active sessions, admin-settable. `CHECK (max_concurrent_sessions >= 0)`; `0` blocks all launches for the user (without disabling the account). Enforced at launch (`control-api.md` `session_quota_exceeded`); "active" = `state ∈ {pending, assigned, starting, running}`. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

> **Per-user quota — design choice (P2-01).** A per-user column (option (a) in the ticket),
> not a single server-wide config default (option (b)), because per-user governance is the
> end-state shape (an admin raises a power user's limit, sets a guest's to a lower value, or
> sets `0` to suspend launching) — it costs one defaulted column now and avoids a migration
> later. The default of `3` is a generous single-host starting point; the real hard limit on a
> busy host is the **per-GPU capacity governor** (encode slots / VRAM), not this per-user
> fairness cap. The quota's "active" set **includes `pending`** (a not-yet-placed launch counts
> against you, so a user cannot evade the cap by spamming launches faster than the scheduler
> places them) and **excludes `stopping`** and the terminal states (a session on its way out
> frees the user's quota immediately). Note this set differs from the **reservation**-holding
> set `{assigned, starting, running}` used for GPU availability — quota counts the in-flight
> `pending` row, GPU reservation does not (nothing is reserved until `assigned`).

## `auth_tokens`
Opaque bearer tokens issued at login (P1-B `/auth/login`). The control plane is
horizontally scalable (architecture §"Control plane"); rather than per-instance session
state, tokens are **opaque random strings stored hashed** and validated against Postgres on
each request — the shared state is the DB, so any instance can authenticate any request, and
tokens are **revocable** (unlike a bare JWT). The plaintext token is returned to the client
exactly once at mint time.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | |
| `user_id` | `UUID` NOT NULL → `users(id)` ON DELETE CASCADE | |
| `token_hash` | `TEXT` NOT NULL UNIQUE | SHA-256 (hex) of the opaque token. Lookup key. |
| `expires_at` | `TIMESTAMPTZ` NOT NULL | |
| `revoked_at` | `TIMESTAMPTZ` NULL | set by `/auth/logout`; non-null = invalid. |
| `last_used_at` | `TIMESTAMPTZ` NULL | touched on use (best-effort; not on the hot path's critical section). |
| `user_agent` | `TEXT` NULL | for the user's "active sessions" view later. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

Index: `(user_id)` for listing/revoking a user's tokens.
A token is valid iff `revoked_at IS NULL AND expires_at > now()`.

## `apps`
The library. **Replaces Wolf's per-app TOML.** An app is a launchable container plus the
defaults the scheduler and pipeline need.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | |
| `name` | `TEXT` NOT NULL | display name |
| `description` | `TEXT` NOT NULL DEFAULT `''` | |
| `cover_url` | `TEXT` NULL | library art |
| `runtime_spec` | `JSONB` NOT NULL DEFAULT `'{}'` | container launch spec the **node agent** consumes: `{ "image", "args":[], "env":{}, "mounts":[], "gpu":true }`. JSONB (not frozen columns) because this is agent-internal launch detail that will grow; the scheduler does **not** read it. |
| `default_vram_mb` | `INT` NOT NULL DEFAULT `1024` | resource hint the **scheduler** reserves (explicit column, not parsed from JSONB). |
| `default_encode_slots` | `INT` NOT NULL DEFAULT `1` | encode sessions this app needs (normally 1). |
| `default_width` | `INT` NOT NULL DEFAULT `1920` | launch defaults for the P1-5 pipeline; per-launch overrides live on `sessions`. |
| `default_height` | `INT` NOT NULL DEFAULT `1080` | |
| `default_fps` | `INT` NOT NULL DEFAULT `60` | |
| `default_bitrate_kbps` | `INT` NOT NULL DEFAULT `15000` | |
| `enabled` | `BOOLEAN` NOT NULL DEFAULT `true` | hidden from the public library when false. |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | |

## `hosts`
One row per node agent (P1-A registration). At N=1 there is exactly one. Host-level
capacity (CPU/mem) lives here; GPU capacity is per-row in `gpus`.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | issued by the control plane on first registration; the agent persists/echoes it on reconnect. |
| `node_name` | `TEXT` NOT NULL UNIQUE | stable agent-provided identity (configured name / machine-id) used to map a reconnect to the same row. |
| `node_secret_hash` | `TEXT` NULL | SHA-256 of the per-node credential issued at enrollment (see `agent-api.md`); authenticates reconnects. |
| `status` | `TEXT` NOT NULL DEFAULT `'offline'` | `CHECK (status IN ('online','offline','draining'))`. `draining` = no new sessions, existing ones finish (Phase 3 rolling ops; modeled now). |
| `agent_version` | `TEXT` NULL | last reported. |
| `cpu_cores` | `INT` NULL | last-reported host capacity. |
| `mem_mb` | `INT` NULL | last-reported host capacity. |
| `last_registered_at` | `TIMESTAMPTZ` NULL | |
| `last_heartbeat_at` | `TIMESTAMPTZ` NULL | liveness; `online` requires recent heartbeat. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

## `gpus`
Per-GPU capacity — the real resource budget. A `sessions` row reserves against exactly one
GPU. Totals are reported by the agent; **availability is derived** (totals minus the sum of
reservations held by active sessions), so reservations cannot drift from session truth.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | |
| `host_id` | `UUID` NOT NULL → `hosts(id)` ON DELETE CASCADE | |
| `index` | `INT` NOT NULL | GPU index on the host. `UNIQUE (host_id, index)`. |
| `vendor` | `TEXT` NULL | e.g. `'amd'`, `'nvidia'`, `'intel'`. |
| `model` | `TEXT` NULL | |
| `vram_mb_total` | `INT` NOT NULL | reported capacity. |
| `encode_slots_total` | `INT` NOT NULL | concurrent encode-session cap (NVENC/VCN limit). |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | |

**Availability (derived, not stored):**
```sql
-- VRAM free on a GPU:
vram_mb_total  - COALESCE(SUM(s.reserved_vram_mb)     FILTER (WHERE s.gpu_id = g.id AND s.state IN ('assigned','starting','running')), 0)
-- encode slots free on a GPU:
encode_slots_total - COALESCE(SUM(s.reserved_encode_slots) FILTER (WHERE s.gpu_id = g.id AND s.state IN ('assigned','starting','running')), 0)
```
The scheduler reserves and checks this transactionally (`SELECT ... FOR UPDATE` on the `gpus`
row, see P1-8) so two launches cannot oversubscribe. At N=1 the check always passes; the code
path is real.

## `sessions`
The central unit: a scheduled, resource-reserved, lifecycle-tracked stream. Created by
P1-B `/sessions` (launch), placed by the scheduler, run by the agent (P1-A), connected via
signaling (P1-D).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | session id, used across all four contracts. |
| `user_id` | `UUID` NOT NULL → `users(id)` | owner. |
| `app_id` | `UUID` NOT NULL → `apps(id)` | what's launched. |
| `host_id` | `UUID` NULL → `hosts(id)` | set on assign; NULL while `pending`. |
| `gpu_id` | `UUID` NULL → `gpus(id)` | set on assign; the reserved GPU. |
| `state` | `TEXT` NOT NULL DEFAULT `'pending'` | `CHECK (state IN ('pending','assigned','starting','running','stopping','stopped','failed'))`. See state machine below. |
| `state_detail` | `TEXT` NULL | human-readable substate / progress. |
| `error_message` | `TEXT` NULL | populated on `failed`. |
| **launch params** | | drive the P1-5 pipeline; default from `apps` then overridable per launch. |
| `width` | `INT` NOT NULL | |
| `height` | `INT` NOT NULL | |
| `fps` | `INT` NOT NULL | |
| `bitrate_kbps` | `INT` NOT NULL | |
| `h264_profile` | `TEXT` NOT NULL DEFAULT `'constrained-baseline'` | `CHECK (h264_profile IN ('constrained-baseline','main','high'))`. P1-11 negotiates this up; the Phase-0 floor is the default. |
| **reservation** | | what was reserved on assign. |
| `reserved_vram_mb` | `INT` NOT NULL DEFAULT `0` | |
| `reserved_encode_slots` | `INT` NOT NULL DEFAULT `0` | |
| **signaling token (single-use)** | | one active token per session; see `signaling.md`. |
| `signaling_token_hash` | `TEXT` NULL UNIQUE | SHA-256 of the single-use token minted at launch. |
| `signaling_token_expires_at` | `TIMESTAMPTZ` NULL | short TTL (default 60 s). |
| `signaling_token_consumed_at` | `TIMESTAMPTZ` NULL | set atomically on first valid signaling connect; non-null ⇒ already used ⇒ reject. **Single-use is enforced here.** |
| **timestamps** | | |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |
| `assigned_at` | `TIMESTAMPTZ` NULL | |
| `started_at` | `TIMESTAMPTZ` NULL | entered `running`. |
| `ended_at` | `TIMESTAMPTZ` NULL | entered `stopped`/`failed`. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | bumped on every state change. |

Indexes: `(user_id, created_at DESC)` for a user's session list; `(host_id)` and `(gpu_id)`
for the availability sums; partial index `(gpu_id) WHERE state IN ('assigned','starting','running')`
to make the reservation sum cheap.

> **Single-token-per-session note (known limitation — see `signaling.md`).** Phase 1 keeps one
> active signaling token *on the session row*, which means a session cannot be re-signalled after
> its token is consumed without re-launching (the mid-session WebRTC reconnection limitation
> documented in `signaling.md`). The migration path is additive and deliberately deferred:
> lift the three `signaling_token_*` columns into a child table
> ```sql
> CREATE TABLE session_tokens (
>     id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
>     session_id   UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
>     token_hash   TEXT NOT NULL UNIQUE,
>     expires_at   TIMESTAMPTZ NOT NULL,
>     consumed_at  TIMESTAMPTZ,
>     created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
> );
> ```
> Then a fresh single-use token can be minted for an already-`running` session (a reconnect),
> and signaling validation joins `session_tokens` instead of reading the session row. Nothing
> else in this schema changes; the message shapes and relay in `signaling.md`/`agent-api.md` are
> untouched.

---

## Session state machine (shared contract)
This is the canonical lifecycle. `agent-api.md` and `control-api.md` use exactly these
names. State is owned by the **control plane** (it writes the row); the agent *reports*
progress via callbacks (P1-A `session_state`) and the control plane maps those onto these
states.

```
              launch (P1-B)    scheduler places     agent acks assign,
              creates row      + reserves GPU       brings pipeline up
 (none) ───────▶ pending ───────▶ assigned ───────▶ starting ───────▶ running ───┐
                    │                 │                 │                 │       │ stop
                    │                 │                 │                 │       ▼  (P1-B / idle / drain)
                    │                 │                 │                 └────▶ stopping ───▶ stopped
                    │                 │                 │                            │
   ── failure from ─┴─────────────────┴─────────────────┴────────────────────────────┴──▶ failed
      ANY non-terminal state (pending, assigned, starting, running, stopping)
```

`stopped` and `failed` are the only **terminal** states; the other five are non-terminal and
**every one of them can transition directly to `failed`**.

| state | meaning | resources reserved? | who drives the transition | → `failed` cause |
|---|---|---|---|---|
| `pending` | row created, not yet placed | no | control plane (launch) | scheduler error; `capacity_unavailable` after the row exists; launch abandoned |
| `assigned` | scheduler chose host+GPU, reserved VRAM+slots, sent `session_assign` | **yes** | control plane (scheduler) | agent `ack{ok:false}`; assign times out; host goes `offline` before start |
| `starting` | agent acked assign, bringing up compositor→encode→webrtcbin | yes | agent callback | pipeline build error (`session_state:failed`); agent WS drops; start times out |
| `running` | pipeline live, signaling offer available; client may connect | yes | agent callback | pipeline crash (`session_state:failed`); **agent WS drops / host heartbeat-miss → host `offline`** |
| `stopping` | teardown requested, pipeline coming down | yes (released on terminal) | control plane or agent | agent dies mid-teardown; teardown times out (control-plane reconciliation still ends it terminal) |
| `stopped` | terminated normally, **reservation released** | no | agent callback (or control plane on confirmed teardown) | — (terminal) |
| `failed` | terminated abnormally; `error_message` set, **reservation released** | no | agent callback / control-plane timeout / host-offline reaper | — (terminal) |

### Failure & reservation-release invariants (load-bearing)
1. **`failed` is reachable from every non-terminal state.** There is no non-terminal state from
   which a session cannot fail; a stuck session is a bug, not a designed state.
2. **Every terminal transition releases the reservation in the same transaction** that writes the
   terminal state. The release is unconditional: reservation is *held* only while
   `state ∈ {assigned, starting, running}` (exactly the GPU-availability-sum filter), so on
   `pending → failed` the release is a no-op (nothing was reserved) and on every other path it
   returns the held VRAM + encode slots. There is **no terminal path that leaves a reservation
   dangling.**
3. **The agent WebSocket dropping while a session is `running` is an explicit failure edge.**
   Per `agent-api.md` §Reconnection: a heartbeat-miss / lost agent connection past the threshold
   marks the host `offline` and the control-plane reaper drives **all** that host's non-terminal
   sessions (including `running` and `stopping`) to `failed`, releasing each reservation. This is
   the multi-host failure model exercised at N=1 — the reaper is the authority of last resort, so
   liveness never depends on a cooperative `session_state` callback that a dead agent can't send.
4. Whether the client's WebRTC transport is connected is a *transport* detail, not a session
   state — the session is `running` once the pipeline is up regardless of browser connection
   (architecture invariant #3: transport is an interface, not part of the session model). A
   browser disconnect therefore does **not** by itself fail the session; see the mid-session
   reconnection limitation in `signaling.md`.

## Migrations
**Tooling: `golang-migrate`** with plain-SQL, paired up/down files. Chosen because:
- It is the de-facto standard for Go services; the control-plane skeleton (P1-1) embeds the
  `migrations/` dir via `embed.FS` and runs it both at service boot and as a CLI
  (`migrate -path migrations -database "$DATABASE_URL" up`).
- **Plain SQL, no ORM** — migrations are reviewable wire-level truth, matching the "real SQL,
  not TOML-as-database" stance; no schema-from-struct magic that hides the contract.
- Linear integer versioning is easy to reason about and to gate in CI.

Layout (`migrations/` inside the control-plane module so `//go:embed` can reach it):
```
control-plane/migrations/
  0001_initial_schema.up.sql     -- all six tables, CHECKs, indexes, updated_at trigger
  0001_initial_schema.down.sql   -- drops them in FK-safe order
  0002_user_session_quota.up.sql -- (P2-01) ADD users.max_concurrent_sessions INT NOT NULL DEFAULT 3 CHECK (>= 0)
  0002_user_session_quota.down.sql -- DROP that column
```
The golang-migrate CLI can target this path directly:
`migrate -path control-plane/migrations -database "$DATABASE_URL" up`
File naming is golang-migrate's `{version}_{description}.{up|down}.sql`. Future changes are new
numbered pairs; **0001 is frozen** like the rest of this contract (so `0002` is a new pair, not
an edit to `0001`). The default applies to existing rows, so no backfill is needed. The
committed SQL in `migrations/` is the authoritative DDL — this document is its prose companion;
if they ever disagree, that is a bug to fix under sign-off, not a silent divergence.
