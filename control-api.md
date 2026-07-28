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

> **Amendment — multi-codec (HEVC/AV1), additive, requires sign-off (signed off 2026-07-25).** Adds a
> codec dimension to the session API, all ship-dark (h264 behaviour is byte-identical until an admin
> flips a profile's codec status to launchable). Four pieces, no existing shape or status code
> changes. **(1)** Every session body (`POST /v1/sessions`, `GET /v1/sessions/{id}`,
> `GET /v1/sessions`) gains **`stream.codec`** (`"h264" | "h265" | "av1"`), the resolved session
> codec, alongside `stream.h264_profile`. Additive; `"h264"` for every pre-multi-codec session.
> **(2)** `POST /v1/sessions` accepts an optional **`stream.codec`** override (admin/diagnostic). It is
> validated against the wire codec set (`h264|h265|av1`; a bad value ⇒ `400 validation_failed`) and,
> unlike the other `stream.*` overrides, is orthogonal to the eligibility envelope: a codec-only
> override does not bypass the profile eligibility gate. It bypasses the device-decode and
> decode-failure-history clamps (this is the forced re-test path for a fixed encoder) but **not** the
> host-encoder clamp: forcing a codec the placed host cannot encode returns **`409 conflict`** (no
> session persists), because host encoder capability is physics, not overridable. **(3)** The admin
> `/v1/admin/stream-profiles` write path may set a profile's **`codecs[].status`** (the rollout
> switch). **(4)** Codec resolution is **server-side** (see §Authorization note). Reason codes:
> `validation_failed` (400) and `conflict` (409) are reused; no new code. `agent-api.md`
> (`session_assign.stream.codec`, `capacity.codecs`) and `schema.md` (`sessions.codec`,
> `stream_profiles.codecs`, `hosts.codecs`, `user_device_profile_history.codec`) carry the wire and
> storage halves. See `docs/design/plans/2026-07-22-multi-codec-hevc-av1-spec.md` §3.

> **Amendment — UI-P1 (app classification + per-user favourites), signed off 2026-07-27.** Five
> pieces for the library redesign. Four are additive; **piece (5) is breaking and is a security
> fix, not a feature** — it gets its own block below. **(1)** The app read shape gains **`kind`**
> (`"game" | "desktop"`, backed by `apps.kind`, `schema.md`), a **presentation-only** library
> classification: nothing in scheduling, admission, profile/codec resolution, or the agent wire
> reads it. It is added to **`AppListItem`**, so **`App` and `AdminApp` inherit it** — both are
> `allOf` compositions over `AppListItem` (`openapi.yaml`), so there is no separate admin variant
> to add it to and no read shape that omits it. **(2)** The app **write** shape (`AppWrite` —
> `POST /v1/apps`, `PATCH /v1/apps/{id}`) gains an **optional** `kind`. **Absent means "server
> default on create, unchanged on patch" — absence is NEVER a zero value.** An explicit
> `kind: ""` is **not** "use the default"; it is `400 validation_failed`, as is any value outside
> the enum. The DB `CHECK` is the backstop, not the primary gate. **(3)** `AppListItem` gains
> **`favourite`** (boolean, **always serialized**): whether **the calling user** has favourited
> this app. Resolved per request from the bearer identity — never a stored property of the app,
> never settable via `AppWrite`, never assertable by a client. **(4)** Two new **`RequireAuth`**
> routes, `PUT /v1/me/favourites/{app_id}` and `DELETE /v1/me/favourites/{app_id}` (§Library —
> favourites), plus their two Authorization rows: **any authenticated account, explicitly not
> admin**. Backed by the new `user_app_favourites` join table (`schema.md`, **migration 0034**,
> which also adds `apps.kind`). **(5)** **`GET /v1/apps` changes from public to `RequireAuth`
> (breaking — next block).** No new error code: `validation_failed` (400), `unauthorized` (401)
> and `not_found` (404) are reused throughout. **`agent-api.md`, `signaling.md`, `input.md`,
> `native-client.md` are unchanged** — a library classification and a per-user favourite never
> reach a node agent. See `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 1".

> **Amendment — UI-P3 (runtime presets), additive, signed off 2026-07-27.** A **runtime preset**
> is a reusable container configuration — image, launch arguments, environment, mounts, and the
> managed-home storage defaults — that many apps inherit instead of repeating.
> **(0) The name is load-bearing.** The operator asked for an "App Launch Profile"; that name
> **collides with UI-P4's launch profiles**, which are the quality/encode chain, an entirely
> unrelated object one admin page away. **Runtime preset**, **stream profile** and **launch
> profile** are three distinct nouns and stay distinct in the table, the API, the docs and the UI
> copy. "Preset" also says *inherit-and-override* without implying an ordered chain, which is
> exactly what this is and exactly what a launch profile is not.
> **(1)** A new **admin-only resource**, `runtime_presets`, with five routes under
> `/v1/admin/runtime-presets` (§Runtime presets) and their Authorization rows. Every one is
> `RequireAuth → RequireAdmin`, server-enforced. **(2)** `AdminApp` gains **`runtime_preset_id`**
> (nullable uuid) and `AppWrite` gains the same field, **optional and tri-state on patch**
> (absent = unchanged, explicit `null` = clear, uuid = set). It sits on **`AdminApp`, not
> `AppListItem`** — a preset is container configuration like `runtime_spec`, not library
> presentation, so it is not part of the public read shape. **(3) `NULL` means the app carries
> everything itself, which is exactly today's behaviour**, so every existing app is unchanged and
> this is purely additive (`schema.md`, **migration 0035**).
> **(4) The merge is server-side and happens AT LAUNCH, never in the admin UI on save.** This is
> the load-bearing decision of the feature: if the UI flattened a preset into the app on save,
> editing the preset later would not reach apps already using it and the object would be a
> template, not a shared configuration. The control plane merges on the existing per-launch
> `runtime_spec` assembly path, so **editing a preset changes the next launch of every app using
> it, with no app edit**. **(5) Merge rules** (`schema.md` carries the same list):
> **env** — preset first, app second, **a key set on both takes the app's value**;
> **mounts** — appended, preset first, **no dedupe** (two mounts on one container path is a real
> misconfiguration and must surface, not be silently resolved);
> **args** — appended, preset first;
> **image** — the app overrides when set, blank inherits;
> **managed_home / home_container_path** — the preset provides the default, the app may override.
> **(6) Delete-in-use is `409 conflict`, enforced server-side.** The admin UI disables its Delete
> button when "Used by" is non-empty; that is a UX affordance and **never** the enforcement (the
> same posture as `DELETE /v1/apps/{id}`'s refuse-if-in-use). No new error code:
> `validation_failed` (400), `unauthorized` (401), `forbidden` (403), `not_found` (404) and
> `conflict` (409) are all reused.
> **`agent-api.md` is UNCHANGED, and deliberately so.** The agent still receives exactly one
> opaque, already-flattened `app` object in `session_assign` / `session_swap_app` — a preset is
> invisible from the agent side by construction, and there is no new wire field on that contract.
> `signaling.md`, `input.md` and `native-client.md` are likewise unchanged. Rollout order, as for
> UI-P1: **control plane before client** — `crud.decodeJSON` sets `DisallowUnknownFields()`, so a
> client sending `runtime_preset_id` to a control plane without this amendment is a hard `400`,
> not a silent ignore. See
> `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 3" and
> `docs/design/plans/2026-07-27-admin-mockup-implementation-notes.md` §12.

> **Amendment — UI-P1 (5): `GET /v1/apps` now requires authentication. BREAKING, non-additive,
> signed off 2026-07-27.** Recorded honestly rather than dressed up as additive.
> **What was wrong.** `GET /v1/apps` was the one `/v1` route registered without the auth
> middleware (`control-plane/internal/crud/handler.go:73` — a bare `mux.HandleFunc`, no
> `requireAuth` wrapper) and it declared `security: []` (`openapi.yaml:242`). Any unauthenticated
> caller who could reach the control plane received the **full app catalogue** — every app's
> name, description, cover-art URL and stream defaults. On a self-hosted deployment that is an
> information-disclosure defect, not a feature: this product has no anonymous browsing surface,
> and the sibling read `GET /v1/apps/{id}` has always required a bearer.
> **What changes.** The route is now `RequireAuth` (`security: [bearerAuth]`, `401` added to its
> responses). An unauthenticated caller goes from `200` + the catalogue to `401`. Nothing else
> about the endpoint changes — same path, same pagination parameters, same `200` body (plus the
> additive `kind` / `favourite` keys from (1) and (3)).
> **Blast radius — verified, not assumed.** Both known consumers already send the bearer: the web
> SPA (`web/src/api/library.ts:28` — `listApps(token)` through the shared `apiFetch({ token })`)
> and `quasar-client` (`core/src/control/mod.rs:259` — `self.send(HttpMethod::Get, "/v1/apps",
> true, None)`, whose third argument is `auth`). **No known consumer breaks.** What does break is
> an unauthenticated third-party scrape of the catalogue — which is the point.
> **It also unblocks (3):** `favourite` becomes resolvable without inventing an optional-auth
> middleware, because a caller now always has an identity and the per-user join is always
> well-defined.
> **The rest of the unauthenticated surface was audited in the same pass and is correct as-is:**
> `POST /v1/auth/register`, `POST /v1/auth/login`, `POST /v1/auth/logout` (it revokes the
> presented token, so it must accept an already-invalid one), `GET /agent/ws` and `GET /v1/signal`
> (both authenticate **in-handler** — the per-node `node_secret` and the single-use signaling
> token respectively, neither of which is a bearer header), and `GET /health`. **`GET /v1/apps`
> was the only offender.** A future reader should know the sweep happened and need not repeat it.

> **Amendment — UI-P4 (stream profiles + launch profiles). NOT PURELY ADDITIVE. Requires Opus +
> explicit human sign-off.** This changes what a "profile" *means* across the whole API, so the
> non-additive parts are stated first and are not buried.
>
> **What is breaking.**
> **(B1)** `GET /v1/me/profiles` **changes what it returns**. It returned the stream-profile
> catalogue, where every entry carried a single `width`/`height`/`fps`/`nominal_bitrate_kbps`.
> It now returns **launch profiles**, which have no single resolution: each carries an ordered
> `rungs[]` of stream profiles, any one of which may be the one a session actually resolves to.
> A convenience `nominal` block echoes the **top rung's** numbers so a picker has something to
> render, but it is **advertised, not resolved** — the truth for a running session is that
> session's own `stream` block. A client that reads `profiles[].width` breaks.
> **(B2)** `profile_policy` **loses the value `custom`**. The enum becomes
> `inherit | prefer | force` on **both** the read shape (`AppListItem`, therefore `App` and
> `AdminApp`) and the write shape (`AppWrite`), and `profile_policy: "custom"` is
> `400 validation_failed`. Under the two-object model every app points at a launch profile;
> `custom` existed only so an app could opt out of profiles entirely, and it was also the one
> mode that could not express a codec.
> **(B3)** The stream-profile object **loses `codecs[]` and gains `codec`** (a single catalog
> codec, `h264 | hevc | av1`). The `launchable | future | unsupported` **status enum disappears**:
> a rung is one codec, so there is nothing left for a status to describe. A codec is offered
> because a rung using it exists in a launch profile, and withdrawn by removing that rung. The
> multi-codec amendment's "flip `codecs[].status` to `launchable`" rollout switch is therefore
> retired and replaced by "add or remove a rung".
>
> **What is additive.**
> **(A1)** A new admin resource, **launch profiles**, with five routes under
> `/v1/admin/launch-profiles`, all `RequireAuth → RequireAdmin`.
> **(A2)** The existing admin stream-profile surface gains `POST` and `DELETE`
> (`/v1/admin/stream-profiles`, `/v1/admin/stream-profiles/{id}`); rungs are now created and
> deleted, not only edited. Both were already implemented for `GET`/`PATCH` but had never been
> written into this prose — they are documented here for the first time, along with
> `GET`/`PATCH /v1/admin/profile-policy`.
> **(A3)** Every session body gains **`stream_profile_id`** (nullable): the **rung** the launch
> resolved to. `profile_id` is unchanged and still carries the **user's pick**, which is now a
> launch profile. Two different questions, two different fields.
> **(A4)** Write responses on both admin objects may carry a **`warnings[]`** array. A warning is
> never a failure; see the h264-floor rule below.
> **(A5)** `openapi.yaml` gains `profile_policy` on **`AppWrite`**. This is a **drift fix, not a
> new field**: the Go handler has always validated `req.ProfilePolicy` on create and patch
> (`crud/handler.go:347,418`) while the schema omitted the property entirely.
>
> **Object ids.** Existing ids are preserved as **launch profile** ids (`1080p60` stays
> `1080p60` and is now a launch profile); rungs get new ids of the form
> `<launch-profile-id>-<codec>` (`1080p60-h264`). This keeps `sessions.profile_id`,
> `user_device_profile_history.profile_id` and `host_encoder_certification.profile_id` (all
> un-FK'd `TEXT`) pointing at something that still exists and still means what it meant.
>
> **The H.264 floor.** A launch profile **must** contain at least one rung whose codec is `h264`
> (`400 validation_failed` otherwise, on create and on patch). "And it must be **last**" is a
> **warning**, not a rejection: rejecting would make a migrated launch profile whose stored codec
> order puts h264 first permanently uneditable, and it would add no safety, because the actual
> guarantee is the resolve-time backstop — if no rung survives the clamp chain, the **last h264
> rung dispatches unconditionally**, bypassing every clamp including its own
> `hardware_encoder_required`. What "h264 not last" actually costs is that rungs after it are
> unreachable, which is exactly what a warning is for.
>
> **Eligibility semantics invert, deliberately.** A launch profile is eligible if **any** rung is.
> A 4K launch profile therefore no longer disappears for a client that cannot decode 4K: it is
> offered and resolves to its H.264 1080p floor rung. This **is** a user-visible behaviour change
> and is recorded as a decision, not discovered later. Mitigation: a launch profile whose **top**
> rung is ineligible while a lower one is not is classified **`risky`**, not `eligible`, so it can
> never become `recommended_id`.
>
> Backed by `schema.md` (`launch_profiles`, `launch_profile_rungs`, `stream_profiles.codec`,
> `sessions.stream_profile_id`, three repointed foreign keys, **migration 0036**). **`agent-api.md`
> is unchanged**: the agent still receives one resolved `stream` block and one `codec`; it has
> never known what a profile is. `signaling.md`, `input.md`, `native-client.md` are unchanged.
> **`quasar-client` is affected by (B1)** and its protocol pin is already five tags stale; the
> re-pin is deliberately deferred to its own piece of work rather than done blind. No new error
> code: `validation_failed` (400), `unauthorized` (401), `forbidden` (403), `not_found` (404) and
> `conflict` (409) are reused throughout. See
> `docs/design/plans/2026-07-28-phase4-profile-restructure-respec.md`.

> **Amendment — UI-P5 (per-app launchable launch profiles), additive, requires sign-off.** An app
> may constrain **which launch profiles a user can pick** from the menu beside Play.
> **(1)** `AdminApp` gains **`launchable_profile_ids`** (array of launch-profile ids, **always
> serialized**, `[]` for every pre-UI-P5 app) and `AppWrite` gains the same field, **optional**.
> **`[]` = unrestricted = today's behaviour**, so the feature ships inert: nothing changes for any
> existing app until an operator configures one. **Non-empty = INTERSECT with eligibility** — the
> allow-list can only ever *narrow* what eligibility already permits, never widen it.
> It sits on **`AdminApp`, not `AppListItem`**, for the same reason as `runtime_preset_id`: it is
> operator configuration, and a client is served the already-filtered menu by (2) rather than being
> asked to intersect anything itself.
> **(2)** `GET /v1/me/profiles` gains an **optional `app_id` query parameter**. With it the
> response is narrowed to what that app offers; without it the response is unchanged. An `app_id`
> that does not resolve is **`404 not_found`**, under the same visibility rule as
> `GET /v1/apps/{id}` — never a silent fall-back to the full catalogue, which would widen the menu
> on a typo.
> **(3) THE APP'S OWN DEFAULT IS IMPLICITLY ALWAYS INCLUDED** (`default_profile_id` under
> `profile_policy: prefer`) and cannot be removed. It is deliberately **not** stored in the
> allow-list — it is one field away, and a second copy would need keeping in sync with the column
> on every default change.
> **(4) Only meaningful for `inherit` and `prefer`.** `force` pins the app's launch profile
> outright, so no allow-list can ever apply. That is **mirrored server-side**: setting one on a
> `force` app is `400 validation_failed`, and switching an app *to* `force` **clears** any stored
> list even when the patch says nothing about it. Storing a list that can never apply would leave a
> rule that does nothing today and silently takes effect the moment the policy changes back.
>
> **(5) THE NEW FAILURE MODE, stated precisely.** `POST /v1/sessions` **rejects a `profile_id`
> outside the app's allow-list with `409 profile_not_launchable_for_app`, and no session row
> persists.** This is a new *behaviour* on an existing endpoint, so it is worth being exact about
> what is and is not additive: **no request shape, response shape or status code changes** — `409`
> was already emitted by this endpoint (profile overrides disabled, codec unsupported by the host)
> and is already in its declared set. What is new is a condition under which it fires — a condition
> unreachable for an app nobody has restricted — plus **one new error code**, which is additive
> because `Error.code` is an open string with no enum: a client that does not know the code falls
> through to its generic-409 branch exactly as it does today. **We consider it additive**, on the
> same reading that made P2-01's new admission rejections additive: a new rule on a launch path
> that, at the shipped configuration, never triggers.
> **Why `409` and not `400` or `403`.** The id is valid, exists, and is user-visible — `400
> validation_failed` is already this endpoint's answer for an **unknown** `profile_id`, and reusing
> it would make "no such profile" and "this app does not offer it" indistinguishable to a client.
> `403 forbidden` in this contract is the **role** gate ("valid token, insufficient role", raised
> before any resource lookup); this refusal says nothing about the caller's identity and every
> caller gets the same answer. `409` puts it with its two neighbours: `profile_ineligible` (409 —
> the *device* refuses a valid profile) and `conflict` (409 — the app's `force` policy refuses an
> override).
> **Why its OWN code and not the generic `conflict`.** On this endpoint `conflict` already carries
> two unrelated conditions — profile overrides are disabled, and the placed host cannot encode the
> requested codec — and a client cannot tell them apart except by message string, which is not part
> of this contract. The three want different responses, and this one is the only **recoverable**
> member: it means the caller's menu is *stale* (an operator narrowed the allow-list after it was
> rendered), so the remedy is to re-read `GET /v1/me/profiles?app_id=…` and re-pick. That is the
> same reasoning that gave `profile_ineligible` its own code, and it matches the house style of
> every other distinct 409 on this API (`session_quota_exceeded`, `home_in_use`,
> `session_not_swappable`, `swap_exceeds_reservation`).
>
> **The rule that must not be got wrong.** **Filtering the list in the UI is NOT enforcement.**
> (2) exists so a client renders the right menu; (5) is the gate, and it is checked server-side
> regardless of which client called — exactly as admin endpoints are gated regardless of client
> (CLAUDE.md invariant #6). A client-side-only allow-list is the same class of defect as a
> client-side admin flag. The implicit path is closed too: a launch with **no** `profile_id`
> resolves through the user preference / global default / recommendation chain, none of which knows
> about the app, so a source outside the allow-list is **skipped** rather than used — otherwise the
> implicit path would grant what the explicit path is rejected for. **And the `stream` override
> path is closed:** `stream` is available to every authenticated caller (no role gate), so it
> bypasses the *eligibility* gate but never this one — otherwise one extra field would defeat the
> feature outright. Only `role=admin` bypasses it.
>
> **What this rule does NOT cover: `POST /v1/sessions/{id}/swap`.** It is a launch-time rule. A swap
> re-checks only reservation fit, so a session launched on an unrestricted app keeps its wider
> stream when swapped into a restricted one. That is forced by P2-07's no-resize / no-renegotiate
> contract (issue #68) and is recorded in §Sessions rather than quietly left to be found.
>
> Backed by `schema.md` (`app_launch_profiles`, **migration 0037**). **Deleting a launch profile
> CASCADES** its allow-list rows rather than being refused: an allow-list entry is a *restriction*
> naming a catalogue object, not a reference that would be left pointing at nothing, so it is not
> added to `DELETE /v1/admin/launch-profiles/{id}`'s three refuse-if-referenced dimensions. The
> cost — a cascade that empties a list turns it back into "unrestricted" — is recorded in the
> migration header and the affected apps are written into the admin activity log, so the widening is
> never silent. **`agent-api.md` is unchanged**: the agent receives one resolved `stream` block and
> has never known what a profile is. `signaling.md`, `input.md`, `native-client.md` are unchanged.
> No new error code: `validation_failed` (400), `unauthorized` (401), `forbidden` (403),
> `not_found` (404) and `conflict` (409) are reused. Rollout order, as for UI-P1/UI-P3: **control
> plane before client** — `crud.decodeJSON` sets `DisallowUnknownFields()`, so a client sending
> `launchable_profile_ids` to a control plane without this amendment is a hard `400`. See
> `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 5" and
> `docs/design/plans/2026-07-27-admin-mockup-implementation-notes.md` §3.

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
  admin/explicit-override bypass), `profile_not_launchable_for_app` (409, *UI-P5* — the selected
  launch profile is valid and eligible but is not in the app's `launchable_profile_ids` allow-list;
  the caller's menu is stale, so re-read `GET /v1/me/profiles?app_id=…` and re-pick),
  `restart_required` (409, *host-runtime-settings* — a `PATCH` to
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
| `POST /v1/apps` | **admin** | create an app — *(UI-P5)* including its `launchable_profile_ids` allow-list |
| `PATCH /v1/apps/{id}` | **admin** | edit an app — *(UI-P5)* including its `launchable_profile_ids` allow-list. **UI-P5 adds no new route**: the allow-list rides these two, which are already `RequireAuth → RequireAdmin`, so a non-admin bearer is `403` before the field is ever looked at |
| `DELETE /v1/apps/{id}` | **admin** | *(admin-delete)* remove an app from the catalog — refuse-if-in-use |
| `GET /v1/admin/runtime-presets`, `GET /v1/admin/runtime-presets/{id}` | **admin** | *(UI-P3)* read the shared runtime presets, each with its `used_by` app list |
| `POST /v1/admin/runtime-presets` | **admin** | *(UI-P3)* create a runtime preset |
| `PATCH /v1/admin/runtime-presets/{id}` | **admin** | *(UI-P3)* edit a runtime preset — takes effect on the **next launch** of every app using it, with no app edit |
| `DELETE /v1/admin/runtime-presets/{id}` | **admin** | *(UI-P3)* delete a runtime preset — **refuse-if-in-use (`409`)**. The admin UI's disabled Delete button is a UX affordance, never the enforcement |
| `GET /v1/admin/stream-profiles` | **admin** | *(AS10-01; prose added UI-P4)* list the encode rungs |
| `POST /v1/admin/stream-profiles` | **admin** | *(UI-P4)* create an encode rung |
| `PATCH /v1/admin/stream-profiles/{id}` | **admin** | *(AS10-01; prose added UI-P4)* edit an encode rung |
| `DELETE /v1/admin/stream-profiles/{id}` | **admin** | *(UI-P4)* delete an encode rung — **refuse-if-listed-by-any-launch-profile (`409`)** |
| `GET /v1/admin/launch-profiles`, `GET /v1/admin/launch-profiles/{id}` | **admin** | *(UI-P4)* read the launch profiles, each with its ordered rungs and its `used_by` list |
| `POST /v1/admin/launch-profiles` | **admin** | *(UI-P4)* create a launch profile — must contain an h264 rung |
| `PATCH /v1/admin/launch-profiles/{id}` | **admin** | *(UI-P4)* edit a launch profile, including reordering its rungs (order **is** preference) |
| `DELETE /v1/admin/launch-profiles/{id}` | **admin** | *(UI-P4)* delete a launch profile — **refuse-if-referenced (`409`)** by any app, the global policy, or any user preference |
| `GET /v1/admin/profile-policy`, `PATCH /v1/admin/profile-policy` | **admin** | *(AS10-03; prose added UI-P4)* read/update the global default launch profile and whether users may override |
| `GET /v1/hosts`, `GET /v1/hosts/{id}` | **admin** | host/capacity oversight |
| `POST /v1/hosts/{id}/drain`, `POST /v1/hosts/{id}/uncordon` | **admin** | *(P3-01)* host lifecycle — cordon a host out of service / return it |
| `DELETE /v1/hosts/{id}` | **admin** | *(admin-delete)* forget an offline host — refuse-if-online-or-in-use |
| `GET /v1/apps`, `GET /v1/apps/{id}` | user | the library — **both reads require auth** *(UI-P1: the list was public until 2026-07-27; see the breaking-change amendment. `favourite` is resolved from the bearer identity, so an anonymous read could not answer it anyway)* |
| `GET /v1/sessions/{id}`, `GET /v1/sessions`, `DELETE /v1/sessions/{id}` | **owner or admin** | resource-ownership check (`403` otherwise), not a blanket admin gate |
| `POST /v1/sessions/{id}/swap` | **owner or admin** | *(P2-02)* same ownership check as `DELETE` |
| `POST /v1/sessions/{id}/stats` | **owner or admin** | *(P4-01)* the client posts its own session's browser telemetry — same ownership check as `DELETE` |
| `GET /v1/admin/sessions/{id}/metrics` | **admin** | *(P4-01)* per-session telemetry read (oversight) |
| `POST /v1/me/devices` | user (self) | *(P4-01)* upsert the caller's own device capability; owner is the bearer identity, never a body field |
| `GET /v1/me/devices` | user (self) | *(AS10-08; **LP-SEC-01**)* read the caller's own devices — **now the full list** (was AS10-08 latest-only); owner is the bearer identity |
| `GET /v1/me/profiles` | user (self) | *(AS10-02; **UI-P4**: now evaluates **launch profiles**, each with its per-rung verdicts; **UI-P5**: optional `?app_id=` narrows the result to that app's allow-list — a convenience, **never the gate**, which is `POST /v1/sessions`)* eligibility + recommendation for the caller's device; owner is the bearer identity |
| `PATCH /v1/me/profile-preferences` | user (self) | *(AS10-03; prose added UI-P4)* the caller's preferred **launch profile**; honoured only while the global policy allows user overrides |
| `GET /v1/admin/storage/homes` | **admin** | *(P5-01)* list managed homes (storage oversight) |
| `DELETE /v1/admin/storage/homes/{id}` | **admin** | *(P5-01)* tombstone a home for GC |
| `GET /v1/me/storage` | user (self) | *(P5-01)* the caller's own per-app storage usage |
| `PUT /v1/me/favourites/{app_id}` | user (self) | *(UI-P1)* favourite an app; owner is the bearer identity — no endpoint takes a `user_id`. Idempotent `204`; `404` under the same visibility rule as `GET /v1/apps/{id}` |
| `DELETE /v1/me/favourites/{app_id}` | user (self) | *(UI-P1)* unfavourite; idempotent **and unconditional** `204` for a well-formed UUID — deliberately never `404` |
| `POST /v1/me/password` | user (self) | *(CP-01)* change the caller's own password; subject is the bearer identity, never a body field. Revokes all active tokens on success — client must re-authenticate |
| `GET /v1/admin/settings` | **admin** | *(LP-SEC-01)* read instance settings (`registration_mode`, `storage_provider`, …) |
| `PATCH /v1/admin/settings` | **admin** | *(LP-SEC-01)* update instance settings — how invites are enabled/disabled from the UI |
| `GET /v1/admin/secrets` | **admin** | *(secrets facility)* list the declared secrets + configured/readable/origin + a **masked** hint — never a value |
| `PUT /v1/admin/secrets/{name}` | **admin** | *(secrets facility)* set/replace a secret's value; write-only on the wire, response is the status shape |
| `DELETE /v1/admin/secrets/{name}` | **admin** | *(secrets facility)* clear a stored secret; any declared env-var fallback takes effect again |
| `POST /v1/admin/invites` | **admin** | *(LP-SEC-01)* mint an invite; plaintext code + magic link returned once |
| `GET /v1/admin/invites` | **admin** | *(LP-SEC-01)* list minted invites (never plaintext) |
| `DELETE /v1/admin/invites/{id}` | **admin** | *(LP-SEC-01)* revoke an invite |
| `PATCH /v1/me/devices/{id}` | user (self) | *(LP-SEC-01)* rename / set trust; owner-scoped, `403` on others' |
| `DELETE /v1/me/devices/{id}` | user (self) | *(LP-SEC-01)* revoke — expires that device's tokens; owner-scoped |
| `GET /v1/admin/config/catalog` | **admin** | *(host-runtime-settings)* read the knob catalog |
| `GET /v1/admin/hosts/{id}/settings` | **admin** | *(host-runtime-settings)* read a host's resolved settings + overrides |
| `PATCH /v1/admin/hosts/{id}/settings` | **admin** | *(host-runtime-settings)* update per-host overrides |
| `POST /v1/admin/hosts/{id}/restart` | **admin** | *(host-observability-2)* restart the host's agent (apply pending restart-class config) |
| `GET /v1/admin/hosts/{id}/console-config` | **admin** | *(CM-01)* read a host's console-mode config + reported capabilities |
| `PATCH /v1/admin/hosts/{id}/console-config` | **admin** | *(CM-01)* update a host's console-mode config |
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
| everything else (`/v1/me`, `POST /v1/sessions`, …) | user | any authenticated account. *(UI-P5: `POST /v1/sessions` additionally refuses a `profile_id` outside the app's allow-list with `409 profile_not_launchable_for_app` — a per-app configuration rule, not a role check, which is why it is `409` and not the `403` this table's admin rows produce. It is refused for **every** non-admin caller, including one supplying an explicit `stream` override, which carries no role gate here)* |

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
// request — invite_code REQUIRED when registration_mode = invite_only (LP-SEC-01)
{ "email": "a@b.com", "username": "ada", "password": "<plaintext, TLS only>",
  "invite_code": "<opaque, from the magic link>" }
// 201 — role always comes from the invite (or 'user'), never a request field
{ "user": { "id": "<uuid>", "email": "a@b.com", "username": "ada", "role": "user", "created_at": "..." } }
```
Password hashed with **argon2id** (P1-2). `409 conflict` on duplicate email/username. No token is
returned — the client logs in next (keeps register/login flows independent).

> *(LP-SEC-01, additive) The endpoint is gated by the persisted `instance_settings.registration_mode`
> (`schema.md`; set via `PATCH /v1/admin/settings`):*
> - *`closed` **(default on a fresh install)** — register is refused (`403 registration_closed`); the
>   invitation system is off until an admin turns it on.*
> - *`invite_only` — `invite_code` is **required**; it is validated + atomically consumed against
>   `invites` (`schema.md`, single-use `UPDATE … RETURNING`). The created account's `role` is the
>   invite's `role` — **never claimable from the register wire**. The `used_count` bump is rolled back
>   on a `409` duplicate so a `409` never burns a single-use invite.*
> - *`open` — today's behaviour; `invite_code` ignored if present.*
>
> *Errors: generic **`400 invalid_invite`** for all of missing/unknown/expired/exhausted/revoked (no
> oracle); `403 registration_closed` when `closed`. Redemption is **rate-limited** (reuse the login
> limiter). The web form reads the code from the magic link's `?invite=<code>` query param — a user
> never types it. The bootstrap-admin path is unaffected by `registration_mode`.*

### `POST /v1/auth/login`
```json
// request
{ "email": "a@b.com", "password": "<plaintext, TLS only>",
  // optional, native client only (P9-01); web/legacy clients omit them:
  "client_version": "1.2.0", "contract_version": "p9-01",
  // optional (LP-SEC-01); when present, binds the minted token to this device:
  "device_key": "<client-generated localStorage UUID>" }
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

> *(LP-SEC-01, additive) `device_key` is an **optional** request field. When present, the server
> upserts the `(user_id, device_key)` `user_devices` row and **stamps the minted
> `auth_tokens.device_id`** so the token is revocable per-device (`schema.md`). When absent
> (legacy/native clients), behaviour is exactly as today: the token is minted with `device_id =
> NULL` and is not device-revocable until a device-declaring re-login. The response shape is
> unchanged.*

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

## Account security — invites + device management (LP-SEC-01)

> **Amendment — LP-SEC-01 (W1 security wave), additive, requires sign-off.** Adds the admin
> **instance-settings** surface (`GET/PATCH /v1/admin/settings`) exposing `registration_mode` and *(storage-config amendment, additive)* `storage_provider` (`auto`|`local`|`volume`, managed-home backing store — see `schema.md §instance_settings`)
> (`closed` **default** | `invite_only` | `open`) so the invitation system is **off by default
> and turned on by an admin in the UI** (persisted in `schema.md` `instance_settings`; the
> `REGISTRATION_MODE` env only seeds first boot); the admin **invite** surface
> (`POST/GET /v1/admin/invites`, `DELETE /v1/admin/invites/{id}`) whose mint returns a **magic
> one-time link**; the register redemption gate + login `device_key` binding (documented inline
> at `POST /v1/auth/register` / `POST /v1/auth/login`); and the owner-self **device
> management** surface (`GET /v1/me/devices` now a **list**, `PATCH`/`DELETE
> /v1/me/devices/{id}`). Enforcement is server-side (role gate + token binding + persisted
> mode), **never UI-gated**. The bootstrap-admin path is unaffected — an admin always exists to
> turn the system on. See `docs/w1-security/LP-SEC-01-contract.md`.

### `GET` / `PATCH /v1/admin/settings` — the enable/disable control (admin)
`RequireAuth → RequireAdmin`. The UI toggle that turns the invitation system on/off.
```json
// GET /v1/admin/settings — 200
{ "settings": { "registration_mode": "closed", "updated_by": "<uuid|null>", "updated_at": "..." } }
// PATCH /v1/admin/settings — request (partial)
{ "registration_mode": "invite_only" }
// 200 — same shape as GET, with updated_by stamped to the acting admin
```
`registration_mode` validated against `{closed, invite_only, open}` (`400 validation_failed`
otherwise); persisted to the `instance_settings` singleton (`schema.md`). Takes effect
immediately — no redeploy. `403` for non-admin (precedes any lookup).

## Encrypted secrets — the reusable operator-credential facility (2026-07-28)

> **Amendment — additive, admin-gated, requires sign-off.** Adds `GET /v1/admin/secrets`,
> `PUT /v1/admin/secrets/{name}` and `DELETE /v1/admin/secrets/{name}`. No existing shape
> changes. Cover artwork's SteamGridDB key is consumer #1; the facility is general, because
> more operator credentials are coming and each one must not reinvent the crypto.

Operator credentials (API keys, passwords, tokens) live **encrypted** in the
`instance_secrets` table (`schema.md`) and are decrypted **only at the point of use**. The
master key is `QUASAR_SECRET_KEY` in the environment and is never in the database, so a
database dump is not enough to read a stored credential.

### The wire is write-only

**No response on any of these routes ever contains a secret value.** There is deliberately no
"reveal" endpoint: a stored credential is for the server to use, and an admin who needs the
value again re-issues it at the provider. A `GET` returns, per declared secret:

| field | meaning |
|---|---|
| `configured` | a value is stored — **whether or not** this control plane can decrypt it |
| `readable` | `false` when a value is stored but the master key is missing or wrong |
| `hint` | the **last 4 characters** of the stored value, and **empty** when the value is short enough that 4 characters would be a meaningful fraction of it |
| `env_set` | whether the declared fallback env var is **present** — presence only, never its value |
| `origin` | which source is actually in effect: `database`, `environment` or `none` |
| `key_version` | which master-key version wrote the row |
| `problem` | why `readable` is `false`, in operator-facing words; never any part of a value |

### Precedence: the database wins, the environment is the fallback

`origin: "database"` outranks `origin: "environment"`. An admin who types a key into the UI must
not be silently overridden by a stale env var — a control that appears to save and then does
nothing is the worst outcome available. Because the env var remains the fallback, an existing
deployment upgrades with **no change** (nothing is stored yet, so the env var is used), and
clearing the stored secret falls **back** to the env var rather than off a cliff. Both facts are
visible: `origin` says which one is live, and `env_set` says the other exists.

A **stored-but-unreadable** secret reports `origin: "none"` and does *not* fall through to the
environment. Silently using a different credential than the one an admin configured is precisely
the surprise this facility exists to prevent.

### Key management, and the two failures that must not look alike

- **Unset master key.** `master_key_configured: false`. This is a **supported state and the
  default**: the control plane boots normally, everything unrelated works, and secret-backed
  features report themselves unavailable. A key is *never* generated and persisted on first
  boot — a generated key would diverge across a multi-node deployment and make a database
  backup unrestorable without the node that invented it.
- **Wrong master key.** `configured: true, readable: false`, with a `problem` naming the master
  key specifically. `PUT` answers `409 conflict` with a *recovery* message ("restore the
  original `QUASAR_SECRET_KEY`, or set this secret again"), which is a different message from
  the *setup* one returned when no key is configured at all. Collapsing these two into "not
  configured" is the exact confusion the facility is written against.
- **Rotation** is not implemented, but is not designed out: every row records the
  `key_version` that wrote it, and `QUASAR_SECRET_KEY_PREVIOUS` supplies decrypt-only
  predecessors, so a rotated deployment keeps reading old rows and a re-encrypt sweep can be
  added with no schema or wire change.
- **Losing the master key means the stored values are unrecoverable** and must be re-entered.
  That is the design, not a defect: see `docs/configuration.md`.

### The routes

```json
// GET /v1/admin/secrets — 200
{ "secrets": [ { "name": "artwork.steamgriddb.api_key", "label": "SteamGridDB API key",
                 "env_var": "QUASAR_STEAMGRIDDB_API_KEY", "configured": true, "readable": true,
                 "hint": "9f2c", "env_set": false, "origin": "database", "key_version": 1,
                 "updated_by": "<uuid|null>", "updated_at": "..." } ],
  "master_key_configured": true, "key_versions": [1] }

// PUT /v1/admin/secrets/{name} — request
{ "value": "<the credential>" }
// 200 — the SAME status shape as GET. The value is not echoed.
{ "secret": { "...": "..." }, "master_key_configured": true }

// DELETE /v1/admin/secrets/{name} — 204
```

An **undeclared** name is `404`, not a new row: this is a registry of secrets the build knows
how to use, not an arbitrary admin-writable key/value store. An empty `value` is `400` — `""`
is indistinguishable from unset at every later read, so storing it would be a silent way to
break a feature; `DELETE` is how you clear one.

`RequireAuth → RequireAdmin` on all three, enforced by the middleware at route registration
(invariant #6). A valid non-admin token is `403` on every one.

### `POST /v1/admin/invites` — mint a magic one-time link (admin)
`RequireAuth → RequireAdmin`.
```json
// request — all fields optional; a bare {} mints a single-use user invite
{ "role": "user", "max_uses": 1, "expires_at": "2026-08-01T00:00:00Z", "note": "for Bob" }
// 201 — plaintext code + ready-to-send link, returned EXACTLY ONCE (never retrievable again)
{ "invite": { "id": "<uuid>", "code": "<opaque ≥128-bit>",
              "invite_url": "https://<instance>/register?invite=<code>",
              "role": "user", "max_uses": 1, "used_count": 0,
              "expires_at": "...", "created_at": "..." } }
```
- `code` generated server-side (≥128-bit), returned plaintext **once**, stored **hashed**
  (`invites.code_hash`). `role` defaults `'user'`; `role:"admin"` mints an admin-provisioning
  invite (admin-only — the only path a wire-redeemed account becomes admin).
- **Magic link:** mint → copy link → send to recipient → they open it → the register form is
  pre-filled from `?invite=<code>` → they set email/username/password. Default `max_uses:1` ⇒
  one-time. `invite_url` is composed server-side if a public base URL is configured, else
  omitted and the admin UI composes it from `code` + `window.location.origin`. The link carries
  only the opaque code — no PII, no role in the URL.
- `403` for non-admin.

### `GET /v1/admin/invites` · `DELETE /v1/admin/invites/{id}` (admin)
- `GET` — list minted invites; **never** the plaintext code (show `id`, `role`, `max_uses`,
  `used_count`, `expires_at`, `revoked_at`, `note`, `created_at`).
- `DELETE {id}` — revoke (`revoked_at = now()`), `204`, idempotent; stops redeeming immediately.

### Device management — owner-self surface
`RequireAuth`; owner is the **bearer identity**, never a body field. `403` (not `404`) if the
device id belongs to another user (no existence leak).

**`GET /v1/me/devices` — the full list** *(LP-SEC-01 supersedes the AS10-08 latest-only shape)*:
```json
// 200 — newest-first by last_seen_at
{ "devices": [
    { "id": "<uuid>", "device_key": "<opaque id>", "name": "Living-room PC", "trusted": false,
      "first_seen_at": "...", "last_seen_at": "...",
      "current": true,                    // the device the bearer token is bound to
      "active_session_id": "<uuid|null>", // from sessions.device_id, if any live session
      "capabilities": { "...": "sanitized blob, verbatim" } }
  ] }
```
> *The AS10-08 single-latest consumer moves to `devices[0]` / the `current` device. This is the
> one non-additive shape change in LP-SEC-01 (signed off). `POST /v1/me/devices` (P4-01 upsert)
> is unchanged.*

**`PATCH /v1/me/devices/{id}`** — rename / set trust:
```json
// request (either/both)
{ "name": "Living-room PC", "trusted": true }
// 200 — updated device (same item shape as the list)
```

**`DELETE /v1/me/devices/{id}` — revoke** (the load-bearing endpoint):
```json
// 204 — device revoked
```
- **Real revocation, not a row delete.** The server **expires every `auth_tokens` row where
  `device_id = {id}`** (`revoked_at = now()`) → that device's next request `401`s. Optionally
  (policy) it also ends that device's live session via the existing `DELETE /v1/sessions/{id}`
  teardown if `sessions.device_id = {id}` — **no agent-wire change**.
- Because `auth_tokens.device_id`/`sessions.device_id` are `ON DELETE SET NULL`, deleting the
  device row never cascades a token/session away — revocation is the explicit token-expire
  above, done **before** any row delete.
- **Re-register defense:** a revoked `device_key` that logs in again gets a *fresh* device row +
  fresh token; it does not silently reclaim the revoked token. Access is regained only by a full
  authenticated login (owner-scope + credentials).

---

## Library

### `GET /v1/apps`
Lists enabled apps (the library the user can launch). **`RequireAuth`** — `401` without a valid
bearer. *(UI-P1, breaking: this list was unauthenticated until 2026-07-27; see the amendment
block at the top of this document for what was exposed and why it changed.)*
```json
// 200
{ "items": [
    { "id": "<uuid>", "name": "Foo", "description": "...", "cover_url": "https://...",
      "kind": "game", "favourite": true,
      "default_width": 1920, "default_height": 1080, "default_fps": 60, "default_bitrate_kbps": 15000,
      "default_profile_id": "1440p60", "profile_policy": "prefer",
      "display_stream": { "width": 2560, "height": 1440, "fps": 60, "bitrate_kbps": 20000 } }
  ], "next_cursor": null }
```
`display_stream` is the user-facing stream advertised in the library: it resolves the
app/global profile policy when available, and otherwise falls back to the legacy
`default_*` stream fields. `runtime_spec` and resource defaults are **not** exposed to
clients (agent-internal / scheduler-internal). Disabled apps are omitted.

*(UI-P4)* `default_profile_id` now names a **launch profile**, and `profile_policy` is
`inherit | prefer | force` — **`custom` is gone** (see the UI-P4 amendment block). `display_stream`
resolves through the launch profile's **top rung** (`position = 1`), which is the advertised
setting; a launch may fall through to a lower rung and stream at a different resolution, and the
session's own `stream` block is the truth for that. The **`default_*` columns stay**: they remain
the COALESCE fallback here when no launch profile resolves (an `inherit` app while the global
default is unset), they are what `LaunchConsoleSession` uses unconditionally, and they are the
ceiling on the documented `POST /v1/sessions` stream-override escape hatch, which is reachable for
**any** app and not only a formerly-`custom` one.

*(UI-P1)* `kind` (`"game" | "desktop"`) is the library classification, **presentation only** —
it exists so the client can split and filter the library. Nothing in scheduling, admission,
profile/codec resolution, or the agent wire reads it. It is defined on `AppListItem`, so the
single-app read (`App`) and the admin read (`AdminApp`) inherit it — they are `allOf`
compositions over the same shape.

*(UI-P1)* `favourite` is **whether the calling user has favourited this app**, always
serialized. It is resolved per request from the **bearer identity** — it is never a stored
property of the app, never settable on the write shape, and never assertable by a client.
Cross-user isolation is structural rather than a check: every favourite read and write is
scoped to the bearer's `user_id`, and **no endpoint takes a `user_id`**, so there is no shape
in which one user can read or set another's favourites. Because `favourite` sits on
`AppListItem`, it is **required on every read shape that composes it** — including `AdminApp`
(`GET /v1/admin/apps`), which must therefore join the acting admin's own favourites rather than
serialize a placeholder. There is deliberately **no server-side `?kind=` filter** on this endpoint — the client filters the single page it already
holds; a filter parameter would be a second way to express the same view for no gain. That is
a decision, not an oversight.

### `GET /v1/apps/{id}`
Single app, same fields as a list item — including `kind` and the caller-resolved `favourite`.
`404` if absent or disabled.

> Creating/editing apps and managing hosts (`GET/POST/PATCH /v1/apps`, `GET /v1/hosts`) is the
> **admin** surface (`role=admin`), built in P1-3 against the same `schema.md`. The read shapes
> above are the public subset; admin write shapes are P1-3's to define within this contract's
> conventions (no frozen-interface change — they're additive, admin-gated).

**The app write shape and `kind` (UI-P1).** `kind` is **optional** on create and patch.
**Absent = the schema default on create, unchanged on patch. Absence is never a zero value.**
This is the `cb97bfb` trap made explicit: an omitted field that decodes to `""` / `0` and is
then written **clobbers the column default** — that is how four Tower apps reached
`default_encode_slots = 0` and silently bypassed admission. So `kind` must decode through a
pointer (or an equivalent presence-aware decode), exactly like the numeric `default_*` fields.
An explicit `kind: ""` is **not** "use the default": it is `400 validation_failed`, as is any
value outside `('game','desktop')`. The DB `CHECK` is the backstop, never the primary gate.

> **Rollout order: control plane before client.** `crud.decodeJSON` sets
> `DisallowUnknownFields()` (`control-plane/internal/crud/handler.go:540`), so a client that
> sends `kind` to a control plane **without** this amendment gets a hard `400 validation_failed`
> — not a silent ignore. Deploy the control plane first; a new client against an old control
> plane fails loudly on every app create/edit.

### Per-app launchable launch profiles *(UI-P5, admin)*

An app may constrain **which launch profiles a user can pick** from the menu beside Play. The
allow-list lives on the **admin** app shapes only — `AdminApp.launchable_profile_ids` on read,
`AppWrite.launchable_profile_ids` on write — because a client is served the already-filtered menu
by `GET /v1/me/profiles?app_id=…` and never needs to intersect anything itself.

```json
// GET /v1/admin/apps → 200 (excerpt)
{ "id": "<uuid>", "name": "Steam", "profile_policy": "prefer", "default_profile_id": "high",
  "launchable_profile_ids": ["balanced", "low-bandwidth"] }
```

- **`[]` = unrestricted** — any launch profile the device is eligible for. This is every pre-UI-P5
  app, so the feature is inert until an operator configures one.
- **Non-empty = INTERSECT with eligibility.** It can only ever *narrow* what eligibility already
  permits. A profile in the list that the device is ineligible for is still not offered.
- **The app's own default is implicitly always included and cannot be removed.** That is
  `default_profile_id` under `profile_policy: prefer`, and it is deliberately **absent from the
  array**: it is one field away, and storing a second copy would need syncing on every default
  change. Under `inherit` there is no app default, so a leftover `default_profile_id` is **not**
  folded in — the account or global default decides there, and folding it in would widen the list
  by one for no stated reason.
- **Only meaningful for `inherit` and `prefer`.** `force` pins the app's launch profile outright.
  Setting an allow-list on a `force` app is **`400 validation_failed`**, and switching an app *to*
  `force` **clears** any stored list even when the request says nothing about it. Both halves are
  the mirror of the admin UI hiding the control: a stored list that can never apply is a rule that
  does nothing today and silently takes effect the moment the policy changes back.
- **Write semantics.** On create, absent or `[]` = unrestricted. On patch, **absent = unchanged**,
  `[]` = clear, a non-empty array = replace wholesale (it is a **set**, not an ordered list — order
  is a launch profile's internal concern, not this one's). **Explicit `null` is `400`**: the
  contract gives it no meaning here (unlike `default_profile_id` and `runtime_preset_id`, where
  `null` means "clear"), `[]` already says clear, and reinterpreting it would silently act on a
  value the caller clearly meant something by.
- Every id must name a **user-visible launch profile** (`400` otherwise). A **rung id** (a stream
  profile, e.g. `1080p60-h264`) is not a launch profile and is rejected — the two id spaces look
  alike and differ only in table. Duplicates are deduped, not rejected.
- **Deleting a launch profile CASCADES** its allow-list rows. It is deliberately **not** added to
  `DELETE /v1/admin/launch-profiles/{id}`'s refuse-if-referenced set: an allow-list entry is a
  restriction naming a catalogue object, not a reference that would be left pointing at nothing,
  and refusing would make retiring a profile harder the more carefully an operator had curated
  their apps. The cost is that a cascade which empties a list turns it back into "unrestricted";
  the affected apps are recorded in the admin activity log so the widening is never silent.

**And the enforcement is at `POST /v1/sessions`, not here.** See §Sessions — the filtered read is a
convenience, the launch check is the gate.

### Favourites — owner-self surface *(UI-P1)*
`RequireAuth`, **any authenticated account — explicitly not admin**. The owner is the **bearer
identity**; `{app_id}` is the only path parameter and there is no `user_id` anywhere in the
surface. The **presence of the row is the fact** — there is no boolean column, no soft delete,
and the composite primary key `(user_id, app_id)` is simultaneously the uniqueness constraint
and the idempotency key (`schema.md` `user_app_favourites`).

**`PUT /v1/me/favourites/{app_id}` — favourite an app:**
```json
// 204 No Content — the app is now among the caller's favourites
```
- **Idempotent** (`INSERT … ON CONFLICT DO NOTHING`). A repeat `PUT` is another `204` and does
  **not** re-stamp `created_at` — favouriting twice must not look like favouriting again.
- `404 not_found` when the app id does not resolve **under the same visibility rule as
  `GET /v1/apps/{id}`** (non-admin: absent *or* disabled; admin: absent). A `204` on a disabled
  app would confirm the existence of something the caller cannot read.
- `400 validation_failed` on a malformed UUID. `401` on a missing/invalid token.

**`DELETE /v1/me/favourites/{app_id}` — unfavourite:**
```json
// 204 No Content — the app is not among the caller's favourites
```
- **Idempotent and unconditional** for a well-formed UUID: never-favourited, since-disabled and
  since-deleted all return `204`. It **deliberately never `404`s** — the caller's intent ("this
  is not one of my favourites") is satisfied either way, and an app delete cascades the row away
  on its own, so a `404` here would be unactionable noise. Same posture as
  `DELETE /v1/admin/invites/{id}` and `POST /v1/auth/logout`.
- `400 validation_failed` on a malformed UUID. `401` on a missing/invalid token.

> **No `GET /v1/me/favourites`.** `GET /v1/apps` already carries the joined per-item
> `favourite` state, so a list endpoint would be a **second source of truth** for the same fact
> — two shapes that can disagree, and a client that has to decide which one wins. The favourites
> view is `GET /v1/apps` filtered client-side. This is a decision, not an oversight; revisit it
> only if the catalogue outgrows a single page.

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

### Runtime presets *(UI-P3, admin)*
A **runtime preset** is a reusable container configuration many apps inherit instead of
repeating: image, launch arguments, environment, mounts, and the managed-home storage defaults.
`apps.runtime_preset_id` points at one; **`null` means the app carries everything itself, which
is exactly the pre-UI-P3 behaviour.**

> **Not a launch profile.** UI-P4's *launch profiles* are the quality/encode chain and *stream
> profiles* are its rungs. A runtime preset configures **what container runs**; a launch profile
> configures **how the stream is encoded**. They are unrelated objects that happen to sit one
> admin page apart, so the vocabulary must never be blurred — in the API, the docs, or the UI.

All five routes are **`RequireAuth → RequireAdmin`**, server-enforced.

**`GET /v1/admin/runtime-presets`** — list every preset:
```json
// 200
{ "items": [
    { "id": "<uuid>", "name": "Steam (Proton)", "description": "...",
      "image": "ghcr.io/quasar/steam:latest",
      "args": ["-silent"], "env": { "PROTON_VERSION": "9.0" },
      "mounts": ["/data/steam-cache:/cache"],
      "managed_home": true, "home_container_path": "/home/quasar",
      "used_by": [ { "id": "<uuid>", "name": "Weston Smoke" } ],
      "created_at": "...", "updated_at": "..." } ] }
```
`used_by` is **resolved per read, never stored** — the apps whose `runtime_preset_id` is this
preset. It is what makes the admin UI's "Used by" row and its disabled-Delete affordance
possible; it is not the enforcement (see `DELETE` below). It is `[]` for an unused preset.

**`GET /v1/admin/runtime-presets/{id}`** — one preset, same shape, wrapped in
`{ "runtime_preset": … }`. `404` if absent.

**`POST /v1/admin/runtime-presets`** — create. Only `name` is required; **every other field
absent falls through to the server default and is never written as a zero value**. `args` and
`mounts` must be arrays of strings and `env` an object of string values (`400` otherwise);
`home_container_path` must be absolute. A duplicate `name` is `409 conflict`. Returns `201` with
`{ "runtime_preset": … }`.

**`PATCH /v1/admin/runtime-presets/{id}`** — edit. **Absent means unchanged**, never "reset to
default". The edit **takes effect on the next launch of every app using the preset, with no app
edit** — that is the entire point of the object (see the merge rules below). `404` if absent,
`409` on a name collision.

**`DELETE /v1/admin/runtime-presets/{id}`** — delete, **refuse-if-in-use**:
- `409 conflict` while **any** app references the preset, *including a disabled one* — a disabled
  app still holds the reference and would still be silently reconfigured. Point the apps
  elsewhere (`PATCH /v1/apps/{id} {"runtime_preset_id": null}`) first.
- The DB foreign key is `ON DELETE RESTRICT` (`schema.md`) as a **backstop**, not the gate:
  `SET NULL` would silently strip an app's image/env/mounts and let it launch with a smaller
  spec instead of failing, which is precisely the class of silent misconfiguration this feature
  exists to prevent.
- `204` on success, `404` if absent.

**The app write shape and `runtime_preset_id`.** `AppWrite` gains an **optional**
`runtime_preset_id`. On create, absent/`null` = no preset. On patch it is **tri-state**: absent =
unchanged, explicit `null` = clear the reference (the app goes back to carrying everything
itself), a uuid = set it. A uuid that does not resolve is `400 validation_failed` at write time —
never an FK error surfacing at launch. It is returned on **`AdminApp`** only: a preset is
container configuration like `runtime_spec`, not library presentation, so no public read shape
carries it. The value returned is the app's **own stored column** — the read shape shows the
operator what they typed, never a pre-flattened merge.

**Merge rules — resolved server-side at launch.** The control plane flattens the preset and the
app's own runtime configuration into the single opaque spec the node agent receives, on the same
per-launch path that already assembles `runtime_spec`. **The agent sees no difference: exactly
one flattened `app` object, and `agent-api.md` is unchanged.** Flattening is deliberately *not*
done in the admin UI on save — that would freeze a copy per app and an edit to the preset would
reach nobody.

| field | rule |
|---|---|
| `env` | preset first, app second. **A key present on both takes the app's value** — the app is the more specific object. |
| `mounts` | **appended**, preset first, **no dedupe**. Two mounts on the same container path is a real misconfiguration and must **surface**, not be silently resolved by the server picking one. |
| `args` | appended, preset first. Argument order is meaningful; the app's follow the preset's. |
| `image` | the app overrides when set; **blank or absent inherits the preset's**. |
| `managed_home` / `home_container_path` | the preset provides the **default**; the app may override. `apps.managed_home` is `NOT NULL DEFAULT false` with no "unset", so an app can turn a managed home **on** when its preset has none but cannot turn a preset's **off** — a preset that provisions a per-user home is a storage guarantee for everything inheriting it. An app whose `home_container_path` is still the schema default has expressed no preference and takes the preset's path. |

Any other key in the app's `runtime_spec` (`gpu`, and anything added later) passes through
untouched. **An app with no preset dispatches a `runtime_spec` byte-identical to before this
feature existed** — the no-preset path returns the stored JSONB verbatim, without even a
decode/re-encode round trip.

---

## Stream profiles and launch profiles *(UI-P4, admin)*

Two objects, and keeping them distinct is the entire point of this section. A third noun,
**runtime preset** (above), configures *what container runs*; these two configure *how the stream
is encoded*. Blurring any of the three in a field name, a route or UI copy is the specific mistake
this naming exists to prevent.

- **Stream profile** = **one encode rung**: one codec, one resolution, one frame rate, one bitrate,
  one ABR floor, one playout0, one hardware-encoder requirement, one browser-support hint, plus its
  eligibility thresholds. "AV1 1440p60 at 9 Mbps" is one row. **Not user-facing** — no user ever
  picks one directly, and they are never returned outside an admin read or a launch profile's
  `rungs[]`.
- **Launch profile** = an **ordered list of stream profiles**, best first. This is what a user
  picks, what the global default points at, what a user preference points at, and what an app pins.
  "High" is AV1 1440p60, then HEVC 1440p60, then H.264 1080p60.

Resolution can rise with codec efficiency because each rung carries its **own** resolution **and**
its own bitrate. Falling through a rung can therefore change resolution as well as codec. That is
intended, and it is the whole reason for the two objects.

**Fall-through happens at LAUNCH ONLY, never mid-session.** A mid-session resolution change is a
WebRTC renegotiation and is deliberately out of scope; the one-offer-per-session invariant is
unaffected by this feature.

### Stream profiles — `/v1/admin/stream-profiles`

```json
// GET /v1/admin/stream-profiles → 200
{ "items": [
  { "id": "1440p60-av1", "display_name": "AV1 · 1440p60",
    "codec": "av1",
    "width": 2560, "height": 1440, "fps": 60,
    "nominal_bitrate_kbps": 9000, "abr_floor_kbps": 3500,
    "min_offer_bandwidth_kbps": 11000, "recommended_offer_bandwidth_kbps": 13500,
    "headroom_factor": 1.5, "max_startup_rtt_ms": 0, "min_decode_height": 1440,
    "high_refresh_display": "none", "hardware_encoder_required": true,
    "browser_client": "recommended", "playout0_ms": 50,
    "h264_profile": "high", "visibility": "internal",
    "used_by": [ { "id": "high", "display_name": "High" } ] }
] }
```

- **`codec`** is a single value from the **catalog** vocabulary `h264 | hevc | av1`. Note `hevc`,
  not the wire `h265`; the rename is bridged in exactly one place server-side and never on this
  surface. **This replaces the old `codecs[]` list and its `launchable | future | unsupported`
  status enum, both of which are gone** (amendment B3). A rung *is* a codec.
- **`h264_profile`** stays on the rung and applies to the `h264` codec only. The browser (WebRTC)
  receiver rejects High on both VA and NVENC, so a browser launch still negotiates down to the
  constrained-baseline floor; the rung records the preference for a capable client. Unchanged.
- `POST` creates, `PATCH /{id}` edits, `DELETE /{id}` removes. **`DELETE` is `409 conflict` while
  the rung is listed by any launch profile *or* while any session recorded it as the rung it
  resolved to.** The admin UI's disabled Delete button is a UX affordance, never the enforcement.
- `used_by` lists the launch profiles that list this rung, and is shown **inside the editor** as
  well as the list: editing a shared object changes every consumer, and that is worth seeing
  before you type.
- **`session_count`** (additive, admin read only, omitted when zero) is the **second** `used_by`
  dimension: the number of sessions whose `stream_profile_id` is this rung. It exists because
  `sessions.stream_profile_id` is a plain foreign key with **no `ON DELETE` clause** — deliberately
  `NO ACTION`, since `ON DELETE SET NULL` would erase which rung a historical session actually got,
  which is the whole reason the column exists. One historical session therefore refuses the delete
  at the database, and a rung named by session history is effectively **permanent**. A client must
  treat a non-zero `session_count` exactly like a non-empty `used_by` when deciding whether to
  offer Delete; a rung with only session references can be removed from every launch profile but
  never deleted.

### Launch profiles — `/v1/admin/launch-profiles`

```json
// GET /v1/admin/launch-profiles → 200
{ "items": [
  { "id": "high", "display_name": "High",
    "description": "The 1440p rung for modern codecs, 1080p on H.264.",
    "visibility": "user", "sort_order": 20,
    "rungs": [ { "position": 1, "stream_profile": { "id": "1440p60-av1",  "...": "..." } },
               { "position": 2, "stream_profile": { "id": "1440p60-hevc", "...": "..." } },
               { "position": 3, "stream_profile": { "id": "1080p60-h264", "...": "..." } } ],
    "used_by": { "apps": [ { "id": "<uuid>", "name": "Steam" } ],
                 "global_default": true, "user_preferences": 2 },
    "warnings": [] }
] }
```

- **Order is preference.** The write shape takes an **ordered array of stream-profile ids**; the
  server assigns `position` from that order. A client does not send positions.
- A stream profile may appear **at most once** in one launch profile (`400 validation_failed` on a
  duplicate).
- **The H.264 floor rule.** A launch profile **must** contain at least one rung whose `codec` is
  `h264`: `400 validation_failed` otherwise. **"And it must be last" is a warning, not a
  rejection** — see the amendment block for why. Two warning codes:
  - `h264_floor_not_last` — every rung after the H.264 rung is unreachable, because H.264 passes
    every clamp.
  - `floor_not_least_demanding` — the H.264 rung has a higher `min_offer_bandwidth_kbps` or
    `min_decode_height` than a rung above it, or requires a hardware encoder while a rung above it
    does not. A floor that is harder to satisfy than the rung above it is a misconfiguration.
- `DELETE /{id}` is **`409 conflict`** while the launch profile is referenced by any app
  (`apps.default_profile_id`), by the global policy
  (`stream_profile_policy.global_default_profile_id`), or by any user preference
  (`user_profile_preferences.default_profile_id`). All three are foreign keys as of migration 0036;
  the `409` is the application-layer gate and the FK is the backstop, not the other way round.
  *(UI-P5)* Membership of an app's **`launchable_profile_ids`** allow-list is deliberately **not**
  a fourth dimension: it is a *restriction* naming this profile, not a reference that would be left
  pointing at nothing, so the join row **cascades away** with the profile and the delete proceeds.
  The cost — a cascade that empties an app's list turns that app back into "unrestricted" — is
  bounded (the widened set is still only what the device is eligible for, which is the pre-UI-P5
  behaviour, and this list is stream-quality curation, never an authorization boundary) and the
  affected apps are written into the admin activity log so it is recorded rather than silent.

### `GET` / `PATCH /v1/admin/profile-policy`

```json
{ "global_default_profile_id": null, "user_overrides_allowed": true }
```
`global_default_profile_id` names a **launch profile** (`400 validation_failed` for an id that is
not a user-visible launch profile). `null` means there is no global default and the per-user
recommendation from `GET /v1/me/profiles` decides — which is the shipped state, and migration 0036
deliberately does **not** populate it, because doing so would change the effective resolution of
every `inherit` app for every user in one invisible step.

### `GET` / `PATCH /v1/me/profile-preferences`

```json
{ "default_profile_id": "high", "global_default_profile_id": null, "user_overrides_allowed": true }
```
`default_profile_id` names a **launch profile**, owner is the bearer identity, and it is honoured
only while `user_overrides_allowed` is true. The two policy fields are echoed read-only so a client
can render "your admin has pinned everyone to the default" without a second call.

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
    "stream_profile_id": "1080p60-h264",
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "constrained-baseline", "codec": "h264" },
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
    sessions (`state ∈ {pending, assigned, starting, running, stopping}`).
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

### `POST /v1/sessions/{id}/signaling-token` — reconnect signaling

Mints fresh signaling coordinates for an existing session owned by the authenticated caller.
Allowed states are `assigned`, `starting`, and `running`. The operation does not create a session,
reschedule it, or restart its container.

**Response `201`:**
```json
{
  "signaling": {
    "url": "wss://quasar.example/v1/signal",
    "token": "<single-use plaintext token>",
    "expires_at": "2026-07-12T01:02:03Z"
  }
}
```

- `404 not_found`: the session does not exist or belongs to another user.
- `409 session_not_reconnectable`: the session is terminal or stopping.
- Each successful call inserts a new `session_tokens` row. Concurrent calls are allowed and
  produce independent tokens; consuming one does not invalidate another.
- Normal authenticated API rate limits apply. Plaintext is returned once and never logged.

### `GET /v1/admin/activity` — administrative activity log

Admin-only, newest-first, cursor-paginated history of destructive and operational actions. Each
item contains `id`, `actor_user_id`, `action`, `target_type`, optional `target_id`, bounded
non-secret `details`, and `created_at`. Required actions include configuration changes, agent
restart, host drain/uncordon, console changes, storage GC, and destructive app/host/user/session
operations. Tokens, passwords, invite codes, and signaling payloads are never recorded.

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

*(UI-P5)* **The app's launchable allow-list is enforced HERE.** A `profile_id` outside the app's
allow-list is **`409 profile_not_launchable_for_app`**, and **no session row persists** (the check
runs before the scheduler, so there is nothing to roll back and no reservation to leak). The
allow-list is `AdminApp.launchable_profile_ids` (§Library — per-app launchable launch profiles); it
is empty and therefore inert for every app an operator has not restricted.

- **This is the enforcement, and the UI filter is not.** `GET /v1/me/profiles?app_id=…` returns the
  filtered menu so a client renders the right thing, but this endpoint checks the list itself,
  regardless of which client called. A client-side-only allow-list would be the same class of
  defect as a client-side admin flag.
- **The implicit path is closed too.** A launch with **no** `profile_id` resolves through the app
  pin → user preference → global default → recommendation chain, and the middle two know nothing
  about the app. A user preference or global default outside the allow-list is therefore
  **skipped**, falling through to the next source; the recommendation is computed over the
  **filtered** catalogue. Without this, the implicit path would silently grant what the explicit
  path is rejected for. The app's own pin is never skipped — it is implicitly always included.
- **An explicit `stream` override does NOT bypass it.** This is the one place the override hatch
  stops. `stream` is available to **every authenticated caller** — it carries no role gate on this
  endpoint — so letting it through would make the allow-list opt-out-able by exactly the party it
  constrains: one extra field would defeat the feature, and because an override also short-circuits
  profile resolution, the disallowed chain's rungs, codec and `profile_id` would all be persisted
  and dispatched. The override hatch beats the **eligibility gate** (a device-capability judgement
  an operator may want to force past); it does not beat operator **configuration** of which chains
  an app offers at all. The two claims are different: *"the user could already reach 1080p through
  an override"* is not *"the user can launch a chain the operator removed."*
- **Admin bypasses it**, exactly as admin bypasses the eligibility gate and the override policy.
  `role=admin` is read from the users table server-side, never a client assertion.
- **`force` apps are never restricted** by it: that policy pins the profile, so there is no menu
  for an allow-list to constrain.
- **Why `409`, and why its own code** rather than `400`, `403` or the generic `conflict`: see the
  UI-P5 amendment block. Short version — the id is valid and resolves (so not the `400` reserved
  for an unknown `profile_id`), the refusal says nothing about the caller's identity (so not the
  `403` this API reserves for the role gate), and it is the one 409 on this endpoint a client can
  actually recover from by re-reading its menu (so not the `conflict` shared with "overrides
  disabled" and "codec unsupported by host").
- **KNOWN GAP — an app SWAP does not re-resolve the profile.** `POST /v1/sessions/{id}/swap`
  validates only that the new app fits the held reservation; it does not re-check the allow-list,
  so a session launched on an unrestricted app at `1440p60` and then swapped into an app whose
  allow-list is `["720p60"]` keeps `profile_id = 1440p60` and the stream it was already running.
  **This is deliberate, not an oversight.** A swap must not renegotiate: P2-07's contract is
  explicitly no-resize (the reservation is fixed and the encode pipeline is never structurally
  touched — re-pointing `interpipesrc.listen-to` is the whole operation), and changing the stream
  mid-session would force a WebRTC renegotiation, which is the failure mode issue #68 exists to
  prevent. The allow-list is therefore a **launch-time** rule, and it is stated here rather than
  left to be discovered. It is not a privilege escalation: the wider stream was already granted at
  launch by an app that offered it, and the swap target's own reservation fit is still enforced.
  Narrowing it would mean either refusing such swaps or renegotiating, and both are contract
  changes well beyond this amendment.

*(UI-P4)* `profile_id` now names a **launch profile**, which is the object a user picks. The
**rung** the launch actually resolved to is the separate, additive **`stream_profile_id`** field,
also persisted and echoed in every session body, and also `null` on a legacy/tier/override launch.
`profile_id` answers "what did the user pick"; `stream_profile_id` answers "what did they get".
Because a rung carries its own resolution, those two can legitimately disagree about width, height,
fps and bitrate, and **`stream` is always the truth** for a live session.

#### Rung resolution *(UI-P4)*
> *Supersedes the codec-only clamp chain in §Codec resolution below, which described the same walk
> when the only thing that varied was the codec.*

The launch walks the resolved launch profile's **rungs**, in `position` order, and takes the first
that survives every clamp. Placement is deliberately **codec-blind**: the scheduler does not filter
hosts by rung, the session is admitted against the **top rung** (the highest-demand one, so
admission can never admit what it would have refused), and the rung resolves **after** placement,
where the host is known. The resolved rung is then written back in the single post-schedule stream
update the certification cap already performs.

| # | clamp | rule |
|---|---|---|
| 0 | admin/diagnostic `stream.codec` override | selects the first rung with that codec; no such rung ⇒ `400 validation_failed`; host cannot encode it ⇒ `409 conflict` (unchanged: host encoder capability is physics) |
| 1 | host encoder set | reject a rung whose codec is not in the placed host's reported wire codec set (`agent-api.md` `capacity.codecs`; an agent reporting nothing is h264-only) |
| 2/3 | client decode probe | reject a rung whose codec is hard-gated and unproven (`h265`/`av1` need an explicit `true`; a stale or absent probe means no). **Additionally** reject a rung whose `min_decode_height` exceeds the probe's measured decode height — new, because a rung now carries its own resolution |
| 4 | decode-failure history | reject a rung this (user, device) previously failed to decode. **Keyed by rung id** for rows written from UI-P4 onward; a pre-UI-P4 row keyed `(launch profile, codec)` bans every rung of that launch profile using that codec, which is exactly its old meaning |
| 5 | hardware encoder | reject a rung with `hardware_encoder_required` when the placed host has no hardware encoder. Explicit as a clamp for the first time: it was previously only an eligibility input, where host capability is usually unknown |
| — | **floor** | if **no** rung survives, dispatch the **last h264 rung**, bypassing clamps 0 through 5 **including its own `hardware_encoder_required`**. The user picked the launch profile, eligibility already approved it, and a session that cannot resolve anything is a failed launch rather than a degraded one. Same silent-downgrade-to-H.264 posture as Sunshine/Moonlight |

**Why clamp 4 had to change grain.** Two rungs may share a codec at different resolutions, which is
legal under the floor rule and is the normal shape of a real launch profile, and a
`(launch profile, codec)` key cannot tell them apart. Decode failure is also resolution-dependent
(which is why `min_decode_height` exists), so a failure recorded on an AV1 4K rung would otherwise
permanently and wrongly clamp the AV1 1080p rung of the same launch profile on that device.
Presentation-side failures and passes still record at **launch-profile** grain, because they are
genuinely codec- and resolution-independent, and they are what feeds the eligibility check.

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

#### Codec resolution (multi-codec)
> *Additive amendment — session bodies gain `stream.codec`; `POST /v1/sessions` gains an optional
> `stream.codec` override. Signed off 2026-07-25. Changes no existing shape or status code.*

Every session is one codec, chosen **server-side at launch** and reported as `stream.codec`
(`"h264" | "h265" | "av1"`; `h265` is HEVC on the wire). The client never selects or negotiates the
codec: it answers the single video codec the host offers. Resolution is, in order:

- **candidates** = the selected profile's `codecs` with status `launchable`, in catalog preference
  order (an `auto` launch with no profile, or a legacy/tier launch, has a single candidate: h264);
- **clamp 1, host encoder** = the placed host's reported wire `codecs` set (`agent-api.md`
  `capacity.codecs`; an old agent reporting nothing is h264-only);
- **clamp 2/3, client decode** = the launching device's capability probe
  (`user_devices.capabilities.codecs`, `POST /v1/me/devices`): h265 and av1 are **hard-gated** on a
  proven `true` (a stale/absent probe resolves to the h264 floor, because sending an undecodable
  codec is a black stream, not a quality drop);
- **clamp 4, decode-failure history** = a non-h264 codec this (user, device, profile) previously
  failed to decode is skipped (`schema.md` `user_device_profile_history.codec`, migration 0032), so
  the session degrades to the next candidate rather than the profile vanishing from the picker;
- **result** = the first candidate surviving all clamps, with a **guaranteed h264 floor** (h264 is
  launchable in every profile and decodable by every browser, so a session can never fail to resolve
  a codec, the same silent-downgrade-to-H.264 posture as Sunshine/Moonlight).

**Admin/diagnostic override.** `POST /v1/sessions` accepts an optional `stream.codec` (validated
against `h264|h265|av1`; a bad value ⇒ `400 validation_failed`). It is **orthogonal** to the
`stream.*` resolution envelope: a codec-only override does **not** bypass the profile eligibility
gate (unlike the other `stream.*` fields). It forces a concrete codec, bypassing clamps 2/3
(device decode) and 4 (failure history). This is exactly the forced re-test path by which a
previously-failed codec on a since-fixed encoder gets a fresh trial, whose sustained smooth run
records the clearing pass. It does **not** bypass clamp 1: forcing a codec the placed host
cannot encode returns **`409 conflict`** and no session persists (host encoder capability is
physics). Authorization is unchanged: codec resolution is server-side and the override is honoured
per the caller's role exactly like the other `stream.*` overrides; the codec field is never a
client-asserted capability or an access-control input.

> **UI-P4 retires this paragraph.** `codecs[]` and its `launchable | future | unsupported` status
> enum no longer exist. An admin enables a codec by **adding a rung** using it to a launch profile
> and disables it by **removing that rung** (§Stream profiles and launch profiles). The h264-floor
> requirement survives the change of mechanism: a launch profile must contain an h264 rung. The
> paragraph is kept below as the record of how the rollout switch worked before UI-P4.

**Enabling a codec (admin).** A profile ships with hevc/av1 at status `future` (not launchable), so
the default catalog resolves h264 for everyone. An admin turns a codec on per profile by setting its
`codecs[].status` to `launchable` via the `/v1/admin/stream-profiles` write path (admin-only). The
write validates the catalog codec vocabulary (`h264|hevc|av1`, note `hevc`, the catalog spelling,
not the wire `h265`), rejects a duplicate codec, and requires that **h264 stay launchable** in a
non-empty list (h264 is the unconditional resolution floor, so the catalog must never present it as
disabled while sessions still fall back to it); an empty list maps to SQL NULL and reads back as the
in-code default. A bad codec/status value or a missing h264 floor ⇒ `400 validation_failed`.

#### Codec decision *(UI-P6)*
> *Additive amendment — session bodies gain `codec_decision` and `negotiated_codec`;
> `POST /v1/sessions/{id}/stats` samples gain an optional `codec_mime_type`. Changes no existing
> shape, no status code, no route, and no authorization rule. Nothing on the agent wire
> (`agent-api.md`) is touched: this is a record of a decision the control plane already made.*

The clamp chain above answers "which rung did this session get". It did not answer **"why not the
one above it"** — the resolver's per-rung verdicts were logged and discarded, so after the fact the
only way to explain a fallback was to re-derive it by hand from the host codec set, the device
probe and the failure history, on the assumption all three still read the same as they did at
launch. `codec_decision` is that record, persisted at launch (`schema.md` `sessions.codec_decision`)
and echoed on **every** session body.

```json
"codec_decision": {
  "result_rung": "1080p60-h264",
  "result_codec": "h264",
  "override": null,          // the operator/diagnostic stream.codec that pre-empted the walk
  "floor": false,            // true ⇒ NO rung survived and the unconditional h264 floor fired
  "considered": [            // every rung WALKED, in position order
    { "rung_id": "1440p60-av1",  "codec": "av1",  "rejected_by": "client_decode",
      "selected": false, "clamps_bypassed": false },
    { "rung_id": "1440p60-hevc", "codec": "h265", "rejected_by": "host_encoder",
      "selected": false, "clamps_bypassed": false },
    { "rung_id": "1080p60-h264", "codec": "h264", "rejected_by": null,
      "selected": true,  "clamps_bypassed": false }
  ]
}
```

- `rejected_by` is the clamp that killed the rung, `null` when it passed:
  `host_encoder` (1) · `client_decode` (2/3, codec) · `decode_height` (2/3, resolution) ·
  `decode_history` (4) · `hardware_encoder` (5) · `unknown_codec` (a rung whose catalog codec does
  not map — hand-edited data; `codec` then carries the raw catalog value). Treat the set as
  **open**: a client must render an unrecognised reason rather than assume the list is closed.
- `considered` holds only the rungs actually **walked**. The walk stops at the first survivor, so a
  clean top-rung win lists exactly one entry — that is not a truncated record, it is the whole
  decision.
- `codec_decision` is `null` for every pre-UI-P6 session and for any launch that walked no rung
  chain (console; a legacy/tier launch that resolved no launch profile). Always serialized.

**The three outcomes are deliberately distinguishable, and a client must not collapse them.** The
`codec` a session ends up with is identical in all three; only this record tells them apart:

| outcome | how it reads |
|---|---|
| **won on merit** | `selected: true`, `rejected_by: null`, `clamps_bypassed: false`, `floor: false`, `override: null` — it was measured against every clamp and survived |
| **operator override** (clamp 0) | `override` names the forced codec and the selected rung has `clamps_bypassed: true` with `rejected_by: null`. It **skipped** clamps 2/3, 4 and 5 rather than surviving them — clamp 1 is the only one an override honours, so a `rejected_by: "host_encoder"` here is the `409` path and no session persists |
| **the floor** | `floor: true`, and the selected rung is `clamps_bypassed: true` **while still carrying the `rejected_by` that killed it during the walk**. That pairing is the point: the terminal rung was dispatched *despite* being rejected. Recording it as an unqualified pass would misinform an operator about a session that is, for example, running a codec this device has already failed to decode |

`floor` and `clamps_bypassed` answer different questions — "did anything survive?" versus "was
*this* rung measured?" — and the override case sets the second without the first. Do not merge them.

**`negotiated_codec` — what the receiver actually decoded.** The session body also carries the wire
codec the **client** reports it is decoding (`"h264" | "h265" | "av1"`, or an unrecognised
lower-case token, or `null` until reported), beside `stream.codec`, which is what the **server**
resolved. They should agree. When they do not, that is how a silent fallback or a mis-negotiated
m-line presents, so **both are kept and neither is reconciled away** — a value the server never
resolves (`vp9`) is preserved rather than dropped precisely because it is the loudest case there
is. Comparison is the reader's: the API states two facts and flags nothing.

The value arrives on the existing telemetry path: each sample in `POST /v1/sessions/{id}/stats`
accepts an optional **`codec_mime_type`** (the `getStats()` codec `mimeType`, e.g. `"video/H264"`),
a **sibling string field** alongside `client_health` for the same reason — the `schema.md` browser
metrics dictionary is numeric, and a codec name has nowhere to live inside it. The server takes the
newest sample in the batch that carries a usable value (a batch legitimately ends with samples
posted before `getStats()` resolved the codec), normalises it to the wire vocabulary, bounds it,
and writes it to the session **only when it changes** and **only while the session is non-terminal**
— a late flush from a torn-down client never rewrites the historical record. An unparseable value is
dropped, never stored, and never masks an earlier good one; the POST still returns `202`.
Authorization is unchanged (owner-or-admin, the bearer identity): this is telemetry, and a
non-owner's `403` writes nothing.

### Admission control (P2-01)
> *Additive amendment — the rule the launch path enforces. Defines wire behaviour only; the
> implementation (atomic, no-double-admit-under-concurrency) is P2-03.*

A `POST /v1/sessions` launch is admitted only if **both** gates pass, evaluated in this order:

1. **Per-user session quota.** Let `active` = the caller's sessions in `state ∈ {pending,
   assigned, starting, running, stopping}` (the non-terminal set — see `schema.md`
   `users.max_concurrent_sessions`). If `count(active) ≥ user.max_concurrent_sessions`, reject
   with **`409 session_quota_exceeded`** and persist no row. A `max_concurrent_sessions` of `0`
   blocks every launch for that user. This gate is per-user and independent of host capacity.

2. **Per-GPU capacity (the governor).** The request's resource ask comes from the app row —
   `requested_encode_slots = apps.default_encode_slots` (clients never set this; the `stream`
   block carries resolution/bitrate, not resource reservations). A GPU admits the launch iff
   **both** of the following hold, with availability derived per `schema.md` §gpus
   (`total − Σ reservations of sessions in {assigned, starting, running, stopping}`):

   **(a) The encode-slot reservation.** This is the real budget and the only race-safe one:
   ```
   encode_slots_available ≥ requested_encode_slots
   ```

   **(b) The live free-VRAM veto** *(#383, amended)*. A GPU whose most recent live sample
   (`gpus.vram_mb_free`, reported on the agent heartbeat — `agent-api.md` §heartbeat) shows less
   free memory than the server's floor is not a candidate, even with slots free. The floor is
   **server policy, not a client or per-app input**.

   The veto is **advisory**: it refuses a GPU that is *already* out of memory; it does not
   allocate or reserve memory, and slots remain the reservation. Because a sample lags reality,
   sessions admitted too recently for the sample to reflect their allocation are debited against
   it.

   The veto **abstains** — admission falls through to (a) alone — whenever the answer is not
   trustworthy: no sample, no sample timestamp, a sample older than the server's freshness
   window, a GPU whose reported total is at or below the floor (an AMD APU's UMA carve-out is not
   the pool the workload lives in), or the floor configured to zero. **Unknown telemetry must
   never reduce availability**: an agent whose sampler breaks cannot be allowed to strangle its
   own host.

   > **Declared per-app VRAM is no longer part of admission.** `apps.default_vram_mb` and
   > `sessions.reserved_vram_mb` are **deprecated** (`schema.md`). The API still accepts
   > `default_vram_mb` on app write and it is still returned on read, but it no longer influences
   > placement, and new sessions record `reserved_vram_mb = 0`. It was never enforceable — no
   > VRAM cap is applied to a session — so it only ever encoded a guess that under-declaring
   > could silently oversubscribe.

   - If **no online host** has any GPU that could serve the request — no host is `online`, or none
     has a GPU whose **encode-slot totals** could ever satisfy the ask — reject with
     **`503 no_host_available`**. This is the "fleet is empty/down" condition; it is distinct from
     "full". Note the VRAM floor is deliberately *not* part of this test: a GPU whose total is at
     or below the floor is abstained from the veto and is therefore servable, so counting the
     floor here would report a merely-busy host as one the fleet can never serve.
   - If a candidate GPU exists (its totals suffice) but **no** candidate currently has enough
     **available** slots, or none currently shows enough **free VRAM**, reject with
     **`503 capacity_exhausted`**. This is the "up but full right now" condition; it is
     **retryable** — a later launch may succeed once an active session ends, or once memory is
     released and the next sample reflects it.
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
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "main", "codec": "h264" },
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
- **Reservation rule (Phase 2, amended by #383): the swap must fit within the held reservation.**
  The new app's `default_encode_slots` must be ≤ the session's `reserved_encode_slots`. If not,
  `409 swap_exceeds_reservation` and **no** swap is attempted (the session is untouched).
  Reservation *resize* on swap is deferred past Phase 2. The VRAM half of this rule is
  **removed**: declared per-app VRAM no longer participates in admission, so it cannot bound a
  swap either. The live free-VRAM veto is an *admission*-time check and is deliberately not
  applied to a swap — the session already holds its GPU and its memory, and refusing the swap
  would not release anything.
- **The stream is NOT re-resolved** *(stated for UI-P5)*. Reservation fit is the only thing checked;
  the session keeps its `profile_id`, `stream_profile_id` and `stream` across the swap. So a
  session launched on an unrestricted app can end up running an app whose
  `launchable_profile_ids` would not have offered that profile at launch. That follows directly
  from this endpoint's no-resize / no-renegotiate contract — see §Sessions (UI-P5) for the full
  reasoning and why narrowing it is a larger change than this amendment.
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

> **Amendment — host-observability, additive.** The host body (`GET /v1/hosts`,
> `GET /v1/hosts/{id}`, and the host list) gains `storage` — the agent-reported storage
> volumes from its latest `capacity` report (`agent-api.md` `host.storage`): an array of
> `{ label, path, total_mb, available_mb }`, serialized always (`null` until an
> amendment-aware agent reports). Canonical schema: `openapi.yaml` `Host` / `StorageVolume`.

> **Amendment — host-observability-2, additive.** The host body additionally gains
> `cpu_model` (agent-reported CPU marketing name, `null` until reported). The GPU capacity
> read (`GET /v1/hosts/{id}/gpus`) gains `render_node` per GPU — the stable by-path device
> path from the capacity report (`agent-api.md` `gpus[].render_node`, `null` until reported);
> the admin UI uses these to offer a render-node picker (`software` + one entry per reported
> GPU) instead of a free-text field. Also adds `POST /v1/admin/hosts/{id}/restart` (below).

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
                   // RVFC captureTime capability (not abs-capture-time wire proof):
                   "rvfc_capture_time_available": 1, "abs_capture_time_negotiated": 0,
                   // RVFC capture-to-display estimate (legacy key name retained):
                   "glass_to_glass_ms": 71, "network_pacing_ms": 7.5,
                   "decode_display_ms": 30.9 },
      // optional sibling STRING fields (the metrics dictionary is numeric):
      "codec_mime_type": "video/H264" }   // UI-P6; see §Codec decision
  ] }
// 202 — accepted (no body); samples are written with source = the `client` value (default 'browser')
```
- Each sample becomes a `session_metrics` row with `source =` the request's `client`
  (default `'browser'`; `'native'` for the native client — P9-01; an unknown `client` value
  is rejected, not silently coerced). `rvfc_capture_time_available` records whether this browser
  yielded a valid RVFC `captureTime` sample; it returns to `0` and staged keys stop when valid
  captureTime becomes stale, even if RVFC callbacks continue with null/invalid metadata.
  `abs_capture_time_negotiated` remains `0` until SDP/RTP-extension wire
  proof exists. The legacy staged keys (`glass_to_glass_ms`, `network_pacing_ms`,
  `decode_display_ms`) are emitted only after valid, fresh RVFC capture-to-display samples. They
  are not a strict abs-capture-time measurement. *(Supersedes the removed
  deep-trace toggle / pixel-overlay instrument.)*
- *(UI-P6, additive)* `codec_mime_type` is an optional per-sample **string** carrying the
  `getStats()` codec `mimeType` the receiver is actually decoding. It is a sibling of the numeric
  `metrics` object, not a key inside it. The server normalises it to the wire codec vocabulary and
  records it on the session as `negotiated_codec` so an operator can compare it against the
  server-resolved `stream.codec` — see §Codec decision for the full rule (newest usable value wins,
  written only on change, never for a terminal session, junk dropped).
- Best-effort: a malformed sample is dropped, not fataled. Accepting telemetry never affects
  session state — with the single, deliberate exception of the two observability fields the client
  is the only source of truth for: the AS10-11 `client_health` class and the UI-P6
  `codec_mime_type`. Neither is a lifecycle transition, an admission input, or an access-control
  input.

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
                   "rvfc_capture_time_available": 1, "abs_capture_time_negotiated": 0,
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
  `display_refresh_unknown`, `display_refresh_too_low`, `browser_playout_unsupported`,
  `historical_client_performance_failed`, `probe_missing`, `probe_stale`.
  Network/decode/codec checks are skipped for an unmeasured probe field (unknown → allow);
  host-encoder and historical-failure inputs are not yet populated in AS10-02 (reserved for
  later issues) and are part of the contract from the start.
  `display_refresh_too_low` is advisory: a measured display below 98% of a profile's
  nominal frame rate makes that profile risky and prevents recommendation, but does not
  make an otherwise eligible profile unlaunchable.
- Debug/internal profiles (`720p30`) are **never** returned.

#### What this endpoint returns after UI-P4 *(BREAKING — read this before the shape above)*

It returns **launch profiles**, not stream profiles. The body above is the pre-UI-P4 shape and is
retained as the record of what changed. The new shape:

```json
// 200
{
  "recommended_id": "balanced",
  "confidence": "high" | "low",
  "notes": [ { "code": "probe_missing", "message": "..." } ],
  "profiles": [
    {
      "id": "high", "display_name": "High",
      "description": "The 1440p rung for modern codecs, 1080p on H.264.",
      "nominal": { "width": 2560, "height": 1440, "fps": 60, "bitrate_kbps": 9000 },
      "eligibility": "risky",
      "reasons": [ { "code": "decode_height_too_low", "message": "..." } ],
      "rungs": [
        { "position": 1, "id": "1440p60-av1",  "codec": "av1",
          "width": 2560, "height": 1440, "fps": 60, "...": "...",
          "eligibility": "ineligible", "reasons": [ { "code": "codec_not_supported", "message": "..." } ] },
        { "position": 2, "id": "1440p60-hevc", "codec": "hevc", "...": "...",
          "eligibility": "ineligible", "reasons": [ { "code": "codec_not_supported", "message": "..." } ] },
        { "position": 3, "id": "1080p60-h264", "codec": "h264", "...": "...",
          "eligibility": "eligible", "reasons": [] }
      ]
    }
  ]
}
```

- **A launch profile is `eligible` if ANY rung is.** This inverts the old semantics on purpose: a
  4K launch profile no longer vanishes for a client that cannot decode 4K, it is offered and
  resolves to its H.264 floor rung. Refusing to start is worse than starting lower.
- **A launch profile whose TOP rung is ineligible while a lower one is not is `risky`, not
  `eligible`.** `recommended_id` only ever picks a fully eligible entry, so a launch profile that
  is *going* to fall through can never become the recommendation. Without this rule the inverted
  semantics would silently recommend a "4K" profile that always streams 1080p.
- **`nominal` echoes the TOP rung's numbers and is advertised, not resolved.** It exists so a
  picker and the admin app-editor preview have something to render. It is **not** what the session
  will stream if a rung falls through; the session's own `stream` block is. Clients must not treat
  it as a promise.
- Each entry in `rungs[]` is a full stream profile plus its own `eligibility` and `reasons`, so a
  client can say *"you will get the H.264 1080p rung on this device, because your browser cannot
  decode HEVC"* instead of implying 1440p. Reason codes are the same stable, append-only set.
- Debug/internal **launch profiles** are never returned. A rung's own `visibility` is `internal`
  by construction and is not a filter here: rungs are returned as part of the launch profile that
  lists them, never standalone.
- **Client impact.** Any consumer reading `profiles[].width` breaks. Known consumer:
  `quasar-client` (`ffi_control.rs`, `ffi.rs`) hand-projects `width`/`height`/`fps` off the
  evaluation. Its protocol pin is already five tags stale; the re-pin is deferred to its own piece
  of work rather than done blind alongside this change.

#### `?app_id=` — the per-app launch menu *(UI-P5, additive)*

`GET /v1/me/profiles?app_id=<uuid>` narrows `profiles[]` to the launch profiles **that app offers**
(its `launchable_profile_ids` allow-list, plus its implicit default), intersected with eligibility
exactly as before. `recommended_id`, `confidence` and `notes` are computed over the **narrowed**
set, so the recommendation can never name a profile the app does not offer.

- The parameter is **optional**. Without it the response is exactly the pre-UI-P5 one, and an app
  with an empty allow-list produces the same body either way.
- An `app_id` that does not resolve is **`404 not_found`**, under the same visibility rule as
  `GET /v1/apps/{id}` (non-admin: absent *or* disabled). Falling back to the full catalogue on a
  typo would silently widen the menu, and a `200` for an app the caller cannot read would confirm
  its existence.
- A `force` app is never narrowed: its policy pins the profile, so no allow-list applies.
- **This is a convenience, not the gate.** `POST /v1/sessions` enforces the same allow-list
  independently (§Sessions), so a client that omits this parameter — or ignores what it returns —
  cannot launch anything extra. Prefer it over filtering client-side anyway: the server is the only
  party that knows the allow-list, and re-deriving the intersection in a client is a second
  implementation of a rule that has exactly one correct answer.

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
    "client.rvfc_capture_time_available": [ { "ts_unix_ms": 1735689600100, "v": 1 } ],
    "client.abs_capture_time_negotiated": [ { "ts_unix_ms": 1735689600100, "v": 0 } ],
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

Records the measured, sustainable encode envelope of a concrete (host, GPU, encoder, **rung**,
bench-bitrate) configuration, so the scheduler can avoid default-starting something a host cannot
hold in real time.

> **Migration 0041 re-keyed this on the RUNG (`stream_profile_id`), not the launch profile.**
> Encode cost is codec- and resolution-dependent and a Phase-4 launch profile chains rungs for up
> to all three codecs, so a verdict measured on the h264 rung was being applied to an AV1 or HEVC
> launch at the same launch profile and bitrate. Wire effect, all **additive**: `stream_profile_id`
> (and `codec` where a codec is implied) appear on the certification row, the run cell plan, and the
> cell/finalize responses; `stream_profile_id` is accepted on the `/cells` and `/finalize` request
> bodies and on the list filter. `profile_id` keeps its place and its meaning as *the launch profile
> the bench ran under* — a request that sends only `profile_id` still works and resolves that
> chain's first h264 rung, which is exactly what the bench streamed before. Backing DDL:
> `schema.md` §`host_encoder_certification`, migration `0041`.

**As-built (SPT-06): script-orchestrated, not control-plane-autonomous** — a
bench session needs a real WebRTC peer to drive frame flow, so the harness
(`deploy/run-spt06-certify.sh`) supplies a Chrome-for-Testing peer and drives the loop: open a run
→ per-cell (launch → CFT peer drives → finalize) → complete. The control plane still owns the
measurement-to-verdict logic (read `session_metrics` → derive verdict → upsert); it does not spawn
a browser itself.

### `GET /v1/admin/hosts/{id}/encoder-certification` — read verdicts (admin)
Returns the **latest** `host_encoder_certification` row per configuration for the host (the
upsert-latest table means "latest" is just "the row"). Optional filters
`?gpu_index=&encoder=&profile_id=&stream_profile_id=` narrow the set; `?max_age_s=` treats anything
older as omitted (the staleness horizon below).
```json
// 200
{ "host_id": "<uuid>",
  "certifications": [
    { "gpu_index": 0, "encoder": "va", "profile_id": "1080p60",
      "stream_profile_id": "1080p60-h264",
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
**1. `POST .../runs` — open a run.** Body optionally scopes launch profiles/encoders/bitrates;
absent ⇒ the host's default matrix. `profiles` names LAUNCH profiles and each is **expanded into
its rungs** (0041), so a multi-codec chain is measured per codec rather than at whichever rung the
bench happened to pick; a rung id may also be named directly to certify one codec. An id that
resolves to neither is skipped. Reserves the per-host lock, returns the cell plan:
```json
// request (all fields optional)
{ "gpu_index": 0, "encoder": "va", "profiles": ["1080p60","1080p45","720p60"],
  "bitrates_kbps": [4000,6000,8000,12000] }
// 202 — run opened
{ "run_id": "<uuid>", "host_id": "<uuid>", "status": "running", "started_at": "...",
  "cells": [ { "profile_id": "1080p60", "stream_profile_id": "1080p60-h264",
               "codec": "h264", "bitrate_kbps": 8000 } ] }
```
**2. `POST .../cells` — launch one cell.** Body
`{run_id, stream_profile_id, bitrate_kbps}` (`profile_id` alone is still accepted and resolves that
launch profile's first h264 rung). Launches a Diagnostics session pinned to the target host/GPU,
**streaming the rung's own codec** — a verdict must be measured on the codec it is filed under — and
returns its `session_id` + signaling token so the harness can attach a CFT peer. Subject to normal
admission control (a fully reserved GPU → `409`, not a stolen slot). A rung whose codec the pinned
host cannot encode is refused up front rather than failing opaquely; a stream profile that is not a
rung, or a rung no launch profile lists, is `400` (neither can ever be launched, so neither can be
meaningfully certified).

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
0. **(0041) Resolve the RUNG first.** The cap is keyed on the rung, so it cannot be evaluated
   before the launch profile's rung walk has said which rung is being started. If the cap then
   fires, the rung is re-resolved over the lower launch profile. The walk consumes no
   certification input, so this ordering cannot change which rung a given chain produces, and the
   SPT-07 probe envelope is still applied to the FINAL rung's bitrate in the single
   post-placement write.
1. Look up the latest `host_encoder_certification` row for `(host_id, gpu_index, encoder,
   stream_profile_id)`, at the bitrate that rung would actually dispatch at. A row older than a
   staleness horizon (default 7 days, tunable) counts as **uncertified**. A verdict for a
   *different* rung of the same launch profile — another codec, another resolution — is **not** a
   verdict about this launch and is never read as one.
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
  "effective": {
    "encoder": "va",
    "render_node": "/dev/dri/renderD128"
  },
  "pending_restart": false
}
```
- **`resolved`** — the full set of effective knob values for this host: for each catalog knob,
  the override value if one exists, otherwise the catalog default. **A display view**
  (`catalog-default ← overrides`) — it cannot see the agent's env baseline; for what the agent
  process is actually running with, read `effective`.
- **`effective`** *(NEW, host-observability, additive)* — the agent-reported settings map from
  its latest `capacity` report (`agent-api.md` `effective_settings`): the true
  `env ← overrides` overlay, values stringified, restart-class knobs latched to the running
  process. `null` when the agent has never reported one (pre-amendment agent). A mismatch
  between `resolved` and `effective` on a restart-class knob means a restart is pending, or the
  agent's env differs from the catalog default the display view assumes.
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
  host has one or more sessions in `state ∈ {assigned, starting, running, stopping}`:
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

### `POST /v1/admin/hosts/{id}/restart` — restart a host's agent *(host-observability-2)*
> *Additive, admin-gated. New endpoint; no change to any existing shape.*

Sends the `restart` downstream command (`agent-api.md`) to the host's agent without changing
any override — the standalone lever for applying an already-persisted restart-class change
(e.g. `pending_restart: true`, or `effective` ≠ `resolved` because the env baseline changed).
```json
// request — confirm mirrors PATCH's restart_confirm and is required only with live sessions
{ "confirm": false }
// 200
{ "restart_triggered": true }
```
- **Live-session guard.** Same semantics as the PATCH restart guard: with sessions in
  `state ∈ {assigned, starting, running}` and `confirm` absent/`false` →
  `409 restart_required` with `live_sessions: N`; with `confirm: true` the restart proceeds
  (sessions are not forcibly stopped — drain first for a clean cut).
- Sets `pending_restart` on the host row; it clears when the agent reconnects.
- **Errors:** `404 not_found`; `409 conflict` — host `offline` (agent not connected, nothing
  to restart); `409 restart_required` — as above.

---

## Console mode (CM-01)
> **Amendment — CM-01 (console-mode), additive, requires sign-off.** Adds the admin
> per-host **console-config** surface driving local display + local audio + local input
> ("use the host like a console"). It changes no existing shape. Storage: `schema.md`
> `console_config`. Delivery to the agent: `agent-api.md` `config_update.console_config`
> (additive) + capability enumeration in `capacity.console_capabilities` (additive). The
> node-agent reads this instead of the spike's `QUASAR_LOCAL_DISPLAY` env hardcode.

### `GET /v1/admin/hosts/{id}/console-config` — read a host's console config + capabilities (admin)
`RequireAuth → RequireAdmin`. Returns the resolved config (defaults applied) plus the host's
latest reported capabilities so the UI can populate selectors.
```json
// 200
{
  "config": {
    "enabled": false, "connector": "auto", "compositor": "weston",
    "output_id": null, "mode": null,
    "audio_output": null, "stream": false, "stream_audio": false,
    "input_devices": "auto", "grab": true,
    "auto_start_on_display": false, "auto_connect_controller": false,
    "default_app": null, "default_user": null, "fullscreen": true
  },
  "capabilities": {
    "connectors": ["DP-4", "HDMI-A-1"],
    "audio_sinks": [ { "id": "hw:1,3", "label": "GPU HDA (DP-4)" }, { "id": "hw:0,0", "label": "Motherboard" } ],
    "input_devices": [ { "path": "/dev/input/event4", "label": "Keyboard" }, { "path": "/dev/input/event5", "label": "Mouse" } ]
  }
}
```
- **`config`** — the resolved console-config object (`schema.md`): every field with its
  override-or-default value. Console-mode is **off** (`enabled:false`) and **local-only**
  (`stream:false`) by default; **`audio_output` has no default** (`null` ⇒ no local audio
  until an admin picks a sink — fail-safe/quiet).
- **`capabilities`** — the host's latest `console_capabilities` report (`agent-api.md`
  `capacity`); empty arrays if the agent hasn't reported (older/offline agent) — the UI then
  offers only `auto`.
- **Errors:** `404 not_found` — no such host; `403` for non-admin (precedes lookup).

### `PATCH /v1/admin/hosts/{id}/console-config` — update a host's console config (admin)
`RequireAuth → RequireAdmin`. Partial/sparse update; only present keys change; `null` clears a
key to its default (except `audio_output`/`default_app`, where `null` is the meaningful
"unset/quiet/no-app" value).
```json
// request — partial
{ "enabled": true, "connector": "DP-4", "compositor": "cage",
  "audio_output": "hw:1,3", "default_app": "<uuid>" }
// 200 — same shape as GET (resolved config + capabilities)
```
- **Validation.** `compositor` ∈ `{weston, cage}`; `connector`/`audio_output`/`input_devices`
  validated against the host's reported `console_capabilities` (unless `auto`/`null`);
  `default_app` FK-checked against `apps(id)`; `default_user` FK-checked against
  `users(id)` (CM-06 — the owner of auto-started console sessions; required when
  `auto_start_on_display` is true). Bad value → `400 validation_failed`.
  `enabled:true` with `audio_output:null` is valid (console runs quiet).
- **Persist + push.** Upserts `console_config.config`, stamps `updated_by`, and **pushes the
  resolved `console_config` to the agent immediately** via `config_update` (`agent-api.md`).
  Takes effect on the next session build; the agent re-arms its hotplug watcher live. **No
  `restart_confirm`** — console config is not restart-class (it is not `gst::init`-latched).
- **Errors:** `404 not_found` — no such host; `400 validation_failed` — bad enum / unknown
  device / unknown `default_app`; `403` for non-admin.

---

## Cover artwork (UI-P7)

Populates `apps.cover_url` — present since Phase 1 and **never written until now** — plus a new
`apps.hero_url`. **Additive and admin-gated**; every route below is new, and no existing shape
changes except the two extra read-only fields on `AppListItem`.

### The rule that governs the whole feature: it ships dark

The artwork provider is configured per deployment (`QUASAR_STEAMGRIDDB_API_KEY`, see
`docs/configuration.md`). **With no key set the control plane makes no third-party request,
starts no background sweep, writes no artwork row for a game, and every app keeps
`cover_url`/`hero_url` `null`** — which renders the gradient tile, byte-for-byte the pre-UI-P7
behaviour. `provider_configured: false` on every artwork response is the shipped default, not a
degraded state, and the provider-backed routes answer `409 conflict` rather than `500`: nothing
is broken, the deployment has simply not opted in to a third party.

The **local** half — upload, override, clear, and serving cached images — works regardless,
because none of it contacts anything. That is deliberate: it is what makes an app a games
database will never carry a solved case rather than a permanent gradient.

### Two crops, not one image scaled

`cover_url` is the **TILE** crop for the 16:10 library-tile frame; `hero_url` is the **HERO**
crop, a much wider banner for the detail/hero panels. They are **different source assets**. A
~2.1:1 tile stretched into a ~3:1 hero reads as a blown-up thumbnail, which is the specific
failure this split exists to avoid. Either may be null independently; a client falls back
`hero_url` → `cover_url` → the gradient tile.

### Cached locally, never hotlinked

Every stored URL is a **local** path, `/v1/artwork/<sha256>.<ext>`. Art is fetched once into the
deployment's own storage and served from there forever. Two reasons, both load-bearing: a
self-hosted box must not depend on a third party being reachable at browse time, and a hotlinked
`<img>` would report the deployment's entire library to that third party on every page view.

### `GET /v1/artwork/{asset}` — serve a cached image

**Deliberately unauthenticated.** A browser cannot attach a bearer token to an `<img src>`, so
the URL is a **capability** instead: the path is the lowercase hex SHA-256 of the image bytes,
which is unguessable (256 bits) and reveals nothing about the app, the user or the catalogue.
The name is content-addressed, so the bytes at a given URL can never change and the response is
`immutable`-cached; it also means nothing a remote supplies influences the on-disk path, and
anything not matching `^[0-9a-f]{64}\.(jpg|png|webp)$` is `404`, never `500`. Responses carry
`X-Content-Type-Options: nosniff` — this is the one route that returns bytes a third party
supplied, and the bytes were already verified to sniff as JPEG/PNG/WebP before being stored.

### `GET /v1/admin/apps/{id}/artwork` *(admin)*

Returns `{ artwork, provider_configured, provider_name, provider_origin, provider_problem? }`.
`artwork` is `null` when the app has no record — the gradient-tile state. `artwork.source` is
one of:

| `source` | meaning |
|---|---|
| `provider` | matched and fetched automatically |
| `manual` | an admin picked, supplied or uploaded it |
| `none` | we looked and there is nothing — a **negative cache**, not an error |

`none` is a first-class outcome. `apps.kind = 'desktop'` (Phase 1) is checked **before** any
provider call, so a desktop app is never queried at all: a games database will not have Blender
or Firefox, and asking costs a request and leaks the app name for a guaranteed miss. An
unmatched *game* records `none` too, so the next sweep does not re-ask. A provider **error**,
by contrast, records nothing — a transient outage must never be cached as "this app has no art".

### Where the provider API key comes from *(2026-07-28 amendment)*

The key is **not** read once at boot. It is resolved on **every use** from the encrypted
secrets store (`artwork.steamgriddb.api_key`, see *Encrypted secrets* above), falling back to
`QUASAR_STEAMGRIDDB_API_KEY`. Consequences worth stating:

- A key set from the admin UI takes effect **with no control-plane restart** — the search
  controls become available on the next request, and the background sweep picks it up on its
  next tick.
- An existing deployment that already set the env var keeps working untouched.
- `provider_origin` (`database` | `environment` | `static` | `none`) says which one is live, so
  an operator never has to guess. `provider_problem` explains a configured-but-unusable key
  (e.g. the master key does not match it) instead of it reading as "no provider configured".
- `QUASAR_ARTWORK_PROVIDER=none` still switches the provider off outright, and a key typed into
  the admin UI does **not** override it.

### `POST /v1/admin/apps/{id}/artwork/search` *(admin)*

Candidate matches for a title (body `{"query"}`, defaulting to the app's own name). An empty
candidate list is a normal result. `409 conflict` when no provider is configured.
`candidates[].thumb_url` is a **remote** preview URL — the one place a provider URL is loaded
directly, and only for an admin who explicitly opened the picker. It is never stored and never
rendered to an end user; the no-hotlinking rule governs the library, and a picker an operator
opened is not the library.

### `PUT /v1/admin/apps/{id}/artwork` — the override *(admin)*

**Fuzzy matching will be wrong sometimes, so this is part of the feature, not a follow-up.**
Exactly one intent per request:

- `{"provider_ref"}` — accept a candidate from the search results.
- `{"tile_url"}` / `{"hero_url"}` — fetch operator-supplied art. Either may be omitted, which
  leaves that crop untouched, so an operator can replace just the hero.
- `{"rematch": true}` — re-run automatic matching now instead of waiting for the sweep. Wanted
  right after a key is first configured, when every app carries a `none` row from before.

Any override sets `locked: true`, and **the automatic sweep never touches a locked record** — a
correction must not be silently re-broken.

An operator-supplied URL is **attacker-adjacent input**: an operator pastes what they were sent.
It goes through exactly the same guards as a provider URL, with **no exception for admin
privilege** — `http(s)` only; the *resolved* address must be publicly routable (loopback,
RFC1918, link-local, CGNAT, and cloud-metadata addresses are refused, and the check is re-applied
to every hop so a public URL cannot redirect into the deployment's own network); at most 3
redirects; a byte cap; and the stored bytes must **sniff** as JPEG/PNG/WebP whatever the
`Content-Type` claimed. SVG is refused outright as an executable document format. A blocked or
mislabelled fetch is `400 validation_failed` and stores nothing.

### `POST /v1/admin/apps/{id}/artwork/upload?crop=tile|hero` *(admin)*

Raw image bytes with an `image/*` `Content-Type`. Needs no provider and no network. The body is
capped by the server's own reader — not by the client-declared `Content-Length` — and oversize is
`413`. Sets `locked: true`.

### `DELETE /v1/admin/apps/{id}/artwork` *(admin)*

Clears the record and NULLs both URLs: back to the gradient tile. Cached blobs are
content-addressed and possibly **shared** with another app, so they are deliberately not deleted
here; unreferenced ones are reclaimed by the service's orphan sweep.

### Authorization

Every route except `GET /v1/artwork/{asset}` is `RequireAuth → RequireAdmin`, enforced by the
middleware at route registration. Hiding the Artwork panel from a non-admin UI is **not** the
access control (invariant #6): a valid non-admin token is `403` on all five admin routes, and a
missing token is `401`.

### The app write shape and `cover_url` (UI-P7 amendment, 2026-07-28)

`AppWrite.cover_url` (`POST /v1/apps`, `PATCH /v1/apps/{id}`) predates this feature and stays a
plain optional string — no schema shape change — but its write semantics are now **enforced**,
closing a gap a review found between this contract and the code:

- **Ownership.** Once an app has an `app_artwork` record (any `source`, including `none`),
  `cover_url` is owned **exclusively** by the routes above. A direct write in the same `PATCH` is
  refused **whole-request** with `409 conflict` — never silently dropped, and never a partial
  update that applies the request's other fields while quietly keeping the old `cover_url`. An app
  with **no** artwork record is unaffected: the pre-existing direct-write allowance (`AppListItem
  .cover_url`'s "honoured only while the app has no artwork record") still works exactly as
  documented there, and this endpoint never touches `app_artwork` — the two write paths cannot
  race each other into a torn state.
- **Format.** A non-null, non-empty direct `cover_url` must be `http`/`https` or a schemeless
  (same-origin) path; anything else (`javascript:`, `data:`, `file:`, ...) is `400
  validation_failed`. This does not change what a *legitimate* direct write looks like — an
  operator was always pointing this at a real browsable URL or their own static asset — it only
  closes a scheme-injection surface the contract left unstated. It is narrower than, and does not
  replace, the SSRF/redirect/content-sniff guard the artwork routes themselves apply to a
  provider-supplied or admin-pasted URL (above): a direct `AppWrite` write is never fetched
  server-side, so there is nothing to dial, only a string the browser later loads as an `<img
  src>` — the "never hotlinked" goal this whole feature exists for (§Cached locally, never
  hotlinked) applies here too, which is why an off-box `http(s)` value remains an admin's own
  choice to make, not one the server can silently launder into a local copy.

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
