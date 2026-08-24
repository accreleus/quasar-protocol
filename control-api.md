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

> **Amendment — session-display-update (live render resolution / UI scale), additive, pre-approved
> 2026-08-15.** Adds the `PATCH /v1/sessions/{id}/display` endpoint (§Sessions), one error code —
> `display_update_rejected` (409) — and one Authorization-table row (owner-or-admin, same rule as
> `DELETE`/`swap`). **Purely additive:** a new endpoint + new code, no change to any existing shape.
> Modelled directly on `session_swap_app` (P2-02): a best-effort relay to the assigned host's agent
> (`agent-api.md` `session_display_update`) that never transitions session state and whose rejection
> is always a no-op. **Render resolution and UI scale are EPHEMERAL** — agent-held only, never
> written to the `sessions` table and never present on the `Session` resource; the encoded stream
> `WxH` is unaffected (§Sessions `stream` block is unchanged by this endpoint). Clients read the
> live values back via `session_metrics` (`agent-api.md`), or keep their own last-acked value. See
> `agent-api.md` §`session_display_update`. **Stops at the contract.**

> **Amendment — session-display-stream (live external/encoded resolution), additive, approved 2026-08-16 (PR #15)
> 2026-08-16, approved 2026-08-16 (PR #15).** Sibling of `session-display-update` (2026-08-15), for the
> **other** half of the resolution vocabulary this repo now needs two words for:
> - **INTERNAL resolution** — the app-facing `wl_output` logical mode `session-display-update`
>   already controls (`render_width`/`render_height`). What the composited scene is *produced* at.
> - **EXTERNAL resolution** — what is actually **encoded and streamed** to the client: the new
>   `stream_width`/`stream_height` this amendment adds. What the composited scene is *upscaled
>   into* before it reaches the encoder.
> Both are independent, live, ephemeral knobs, and both stay `≤` the session's **launch** size
> (the `stream` block as `session_assign` first set it — `session-display-update` never moved it
> and still doesn't). **(2026-08-16 amendment) INTERNAL and EXTERNAL are independent axes, full
> stop** — INTERNAL is bounded ONLY by the launch size, never by the current or any past EXTERNAL
> size. EXTERNAL may sit below the current INTERNAL size; the encoder downsamples the (unchanged)
> render framebuffer, and the app never sees a mode change. This supersedes an earlier draft of
> this amendment that also clamped INTERNAL to the current EXTERNAL size. Extends
> `PATCH /v1/sessions/{id}/display`
> (§Sessions) with `stream_width`/`stream_height` (both-or-neither, must resolve to one of the
> session's `stream.rungs` — see §Sessions §`GET /v1/sessions/{id}`) and one error code,
> `external_resize_unsupported` (409). **Purely additive:** new request/response fields and a new
> code on an already-additive endpoint; no existing field, status code, or shape changes. Unlike
> `session-display-update`, this changes what the **encoder** actually produces — the coded size
> moves at the **next IDR**, with **no WebRTC renegotiation** (`signaling.md` unchanged; the
> client `<video>` element follows the new coded size on its own) — so it is gated on the
> assigned host's encoder supporting a live resize (readback: `stream.external_resize_supported`,
> below). **The in-session ABR governor does not drive this yet** — it is a manual/API lever
> only; a later amendment may add automatic external-size stepping, and this message shape is
> what it would reuse. See `agent-api.md` §`session_display_update` / §`session_metrics`.
> **APPROVED (Michael, 2026-08-16); merged PR #15.**

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

> **Amendment — ST-09 (Verdict value: falsifiers, window, clock, tier), additive, approved by
> Michael 2026-08-23.** Turns the diagnostic bundle's `classifier` object into the **Verdict** —
> one value that carries not only the state and its prose `evidence` (both **unchanged**) but the
> numbers that would overturn it: `reason`, `window` (with per-source sample counts), `clock`
> (quality + offset/uncertainty), `evidence_tier`, a `falsifiers` array, and
> `thresholds_version`. Adds **two** observability-only reads that return that value alone:
> `GET /v1/admin/sessions/{id}/verdict` (**admin**) and `GET /v1/sessions/{id}/verdict`
> (**owner-or-admin**, the same ownership check as `DELETE /v1/sessions/{id}`), both reusing the
> bundle's window query parameters and clamps. **Purely additive:** every existing key, status
> code, and endpoint is unchanged; a consumer that reads only `classifier.verdict` +
> `classifier.evidence` is unaffected. The verdict carries **no session authority** — it is
> observational, exactly as the ST-01 classifier was. A **falsifier** is a named, estimator-
> qualified number (`name`, `estimator`, `value`, `op`, `threshold`, `unit`, `n`, `holds`) that
> the verdict relies on; `holds=false` means the data does not satisfy the condition, and a
> series with `n=0` reports `value: null`, `holds: false` and a `note`. `evidence_tier` states
> which sides of the measurement were actually present. **An unknown `verdict` string is DATA to
> a consumer** — the control plane owns that vocabulary and grows it; a client must report an
> unrecognised value verbatim, never fail on it. Shape and thresholds:
> `docs/session-trace/trace-format.md` §8 and `docs/session-trace/thresholds.json` (the single
> golden threshold file, versioned by `thresholds_version`).

> **Amendment — clock-aligned series + ingest validation, additive, approved by Michael
> 2026-08-23 (additive, admin-only read).** Makes the measured clock offset **load-bearing**
> rather than merely reported. (1) `Verdict.clock` gains `applied` (whether the offset was
> applied to the client-clock series before the rules ran) and `age_ms` (now minus
> `measured_at`), and `Verdict.window` gains `warmup_excluded_ms`. (2) `Falsifier.estimator`
> gains the value `count_ge_threshold` — a count of samples at or above a threshold, alongside
> the existing `max` row, so one outlier sample can no longer carry a verdict. (3) The
> diagnostic bundle gains an optional **`ingest`** object (`rejected_ts`,
> `last_rejected_ts_unix_ms`, `last_rejected_reason`): a client sample or trace event whose
> `ts_unix_ms` is outside ±24 h of server now is **dropped** at ingest — it would otherwise be
> stored where every read window silently excludes it — counted per session in memory, and
> logged at most once per session per minute naming the offending value and its likely domain
> ("looks like seconds" / "looks like performance.now" / "looks like nanoseconds"). The batch
> still returns `202`; a rejection is never a client-visible error. **Purely additive:** no
> existing key, status code, or endpoint changes, and every new field is optional to read. The
> agent's own ingest is unvalidated (it is the trusted host reporter) and `agent-api.md` is
> unchanged. Sign convention and the tolerance rule live in
> `docs/session-trace/trace-format.md` §4.

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
> (`"game" | "desktop"` — **widened to add `"launcher"` by the Steam library discovery Phase 3
> amendment below; the presentation-only promise in this sentence is unchanged and is what that
> amendment leans on**, backed by `apps.kind`, `schema.md`), a **presentation-only** library
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

> **Amendment — Steam library discovery, Phase 1 (`external_source` / `external_id`), additive,
> requires sign-off.** The app object gains **two fields** that say *"this app **is** provider X's
> title Y"* — today only `("steam", <appid>)`. `AppListItem` gains both on read (so `App` and
> `AdminApp` inherit them, exactly as they inherit `kind`), **always serialized**; `AppWrite` gains
> both as **optional** request fields on `POST /v1/apps` and `PATCH /v1/apps/{id}`. **Purely
> additive:** two new fields on one read shape and one write shape, **no existing shape, status
> code, endpoint or behaviour changes**, and no new error code (`400 validation_failed` is reused).
>
> **`""` is a value, not an absence, and that distinction is the whole design.** `external_source:
> ""` / `external_id: ""` means *"this app is not a provider title"* — the state of every app that
> exists today — so both fields are serialized on every read (never `omitempty`: a client must be
> able to tell `""` from absent) and an explicit `""` on the write path is a deliberate **clear**,
> **not** `400`. That is the one place this differs from `kind`, where `""` is a hard `400`. What
> the two share is the presence rule: **absent = the schema default on create, UNCHANGED on patch**
> (the `cb97bfb` trap — an omitted field decoding to `""` and being written clobbers the stored
> value; here it would silently un-tag an app's appid on any unrelated `PATCH` and send its artwork
> back to the fuzzy matcher).
>
> **The appid grammar is argument-injection containment, not tidiness.** `external_id` is `""` or
> `^[1-9][0-9]{0,9}$` — a bare positive integer, no leading zero, no sign, no whitespace, no
> separators. The value is destined for `STEAM_STARTUP_FLAGS`, which the `quasar-steam` entrypoint
> **word-splits** with `read -r -a`, so a stored `"480 -foo"` would arrive at the Steam client as
> two extra arguments. **The handler is the primary gate; the DB `CHECK` is the backstop** — the
> same division of labour as `kind`, and here the `CHECK` is additionally the only one of the two
> that survives an admin editing the value directly later.
>
> **The two fields are validated INDEPENDENTLY.** No pairing rule ("a source requires an id", or
> the reverse) is imposed: none is needed, an admin setting one field in one request and the other
> in the next would be rejected by a rule nothing asked for, and a half-set pair is **inert** — the
> only reader requires *both* before it does anything different.
>
> **The only reader in Phase 1 is artwork resolution** (§Cover artwork). An app carrying
> `("steam", <appid>)` resolves its two crops **by id** and never enters the fuzzy title matcher;
> an exact id beats a fuzzy title match by construction. The `app_artwork` provenance shape is
> **unchanged** — `provider_ref` simply now carries the appid for these apps. **Nothing in
> scheduling, admission, profile/codec resolution, or the agent wire reads either field.**
>
> Backed by `schema.md` (`apps.external_source`, `apps.external_id`, two `CHECK`s and the partial
> index `apps_external_ref_idx`, **migration 0042**). **`agent-api.md` is byte-identical** — the
> whole Steam library discovery spec deliberately avoids touching the agent contract (spec §14:
> `agent-api.md` appears in no phase's contract row). `signaling.md`, `input.md`,
> `native-client.md` are unchanged. Rollout order, as for UI-P1/UI-P3/UI-P5: **control plane before
> client** — `crud.decodeJSON` sets `DisallowUnknownFields()`, so a client sending either field to
> a control plane without this amendment is a hard `400`. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §4.1, §10, §12 and §13 "Phase 1".

> **Amendment — Steam library discovery, Phase 2 (entitlements). NOT ADDITIVE. Requires Opus +
> explicit human sign-off.** Recorded honestly rather than dressed up as additive, in the same
> idiom as UI-P1 (5) and UI-P4.
>
> **Read the blast radius before the shape.** This introduces an **authorization object** —
> *"which subjects may see and launch which apps"* — and then makes **five existing endpoints
> obey it**. Nothing about that is a new field on a new route:
>
> | endpoint | before | after |
> |---|---|---|
> | `GET /v1/apps` | every enabled app, for every caller | **only apps the caller is entitled to**, for **every role including admin** |
> | `GET /v1/apps/{id}` | `404` iff absent or disabled | non-admin: **`404` also when not entitled**. The pre-existing admin branch is **unchanged** (see below) |
> | `PUT /v1/me/favourites/{app_id}` | `204` / `404` | **`403 forbidden`** when the caller is not entitled |
> | `POST /v1/sessions` | `409`/`503` refusals | **new terminal `403 forbidden`** when the caller is not entitled to `app_id` |
> | `POST /v1/sessions/{id}/swap` | `404`/`409` refusals | **new terminal `403 forbidden`** when the **session owner** is not entitled to the target app |
>
> **A client that reads `GET /v1/apps` and shows what it gets keeps working and needs no change.**
> What changes for a client is that the list can now be *shorter than the catalogue*, and that
> two writes it previously only had to defend against `404`/`409` can now answer `403`. `403` on
> `POST /v1/sessions` and on the swap is a **new terminal status on an existing endpoint** — the
> reason this amendment is not additive, and the reason it is signed off rather than glanced at.
>
> **`GET /v1/admin/apps` is untouched and stays the unfiltered god view.** That is what makes
> filtering `GET /v1/apps` for admins acceptable: nothing becomes unreachable, it moves to the
> route that already means "the whole fleet's catalogue".
>
> **Day one, nothing changes for anybody.** Migration 0043 creates the table **and backfills an
> `('all', granted_by='migration')` row for every existing app in the same transaction**, so the
> filter's first evaluation returns exactly what the unfiltered query returned. That is not a
> convenience: turning filtering on against an empty table blanks **every** user's library on
> **every** deployment simultaneously, and it would ship — the migration applies, the service
> boots, `go-test-db` passes (tests create their own entitlements) and the web build passes.
> **The backfill is the only gate between those two outcomes.** See `schema.md`.
>
> **Four new admin routes**, `RequireAuth → RequireAdmin` like every other admin route, and they
> are the **only** way to widen visibility — which is precisely what lets the list and the launch
> stay filtered for admins too. `GET`/`POST /v1/admin/apps/{id}/entitlements`,
> `DELETE /v1/admin/apps/{id}/entitlements/{entitlement_id}`, and
> `GET /v1/admin/users/{id}/entitlements`.
>
> **One new optional request field**, `entitle` on `POST /v1/apps` (`"all" | "none"`, default
> `"all"`). It is the mirror of the backfill: without a default grant, *"I made an app and nobody
> can see it"* becomes the new default experience.
>
> **There is no role bypass anywhere in the enforcement path, and there must never be one.** An
> admin who cannot see something **grants themselves the entitlement**, which is one audited call.
> An `if isAdmin { skip }` arm in the filter or the launch check is the exact shape of the
> client-side-admin-flag defect class `CLAUDE.md` invariant #6 exists to forbid, and it is the one
> code path nobody tests.
>
> **The filtered list is UX. `POST /v1/sessions` is the authorization boundary** — the check runs
> **inside the scheduling transaction, before placement**, so a client that ignores the list and
> posts an app id directly is refused there, and nothing is reserved for a launch that will fail.
>
> Backed by `schema.md` (the `entitlements` table, its two **partial** unique indexes, its shape
> `CHECK`, and the 0043 backfill) and `openapi.yaml` (four paths + the `Entitlement` shapes +
> `AppWrite.entitle`). **`agent-api.md` is byte-identical** — the whole Steam library discovery
> spec deliberately avoids touching the agent contract (spec §14: `agent-api.md` appears in no
> phase's contract row). `signaling.md`, `input.md`, `native-client.md` are unchanged. **Rollout
> order: control plane before client**, and the client re-pin is scheduled **with** this phase
> rather than batched to the end, because the client's library listing narrows here. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §6 and §13 "Phase 2".

> **Amendment — Steam library discovery, Phase 3 (derived tiles + the `launcher` kind).
> Additive, but requires Opus + explicit human sign-off on TWO counts** — it widens a **frozen
> enum** and it adds an app shape whose rules are enforced by a database `CHECK` rather than by
> anything a later reader can renegotiate. No existing field changes meaning, no existing status
> code is removed, and no route is added or removed. **Migration 0044.**
>
> **(1) `AppKind` gains `launcher`: `("game" | "desktop" | "launcher")`.** The Steam provider app
> stays in the user's library once its games are discovered, filed under a Launcher category,
> rather than being hidden (operator decision). Widening an enum is additive **for writers and
> not for readers** — every consumer holds a *closed* set — which is why this needs sign-off
> despite adding nothing.
>
> > **The forward-compatibility consequence, stated plainly.** A client pinned to a protocol
> > **before** this amendment has `AppKind` as a closed two-value union and **will receive
> > `kind: "launcher"`** from a newer server. Nothing crashes: a TypeScript union is erased at
> > runtime, and the library filter's predicate (`kind !== "all" && a.kind !== kind`) simply never
> > matches — so the Launcher tile appears under **All** and under **no segment**. **That is a
> > benign degradation, but it is a degradation**, and it is why the `quasar-client` pin bump is
> > scheduled **with this phase** rather than batched to the end.
>
> **`kind` STAYS PRESENTATION-ONLY, AND THAT PROMISE IS NOW LOAD-BEARING.** A third value invites
> `AND kind = 'launcher'` in a scan query or a launch path. **No server path may branch on
> `apps.kind`.** Discovery is triggered by `library_provider = 'steam'` **and nothing else**;
> gating it on `kind` would turn an additive enum widening into a **semantics change on an
> existing field** — a materially larger sign-off — and would create a failure mode where an
> operator flips a presentation dropdown and a background job silently stops. An admin editor may
> *suggest* `launcher` when `library_provider` is set; it must not enforce it.
>
> **The one exception is the artwork short-circuit, and it stays exactly as narrow as it is.**
> `launcher` joins `desktop` in it (§Cover artwork): a games database has no entry for the Steam
> client, and the evidence is direct — the fuzzy matcher's 7-for-7 failure list on the live
> catalogue is **headed by "Steam (Dev)" matching "Steam Dev Days"**. This is presentation
> deciding presentation, not `kind` acquiring teeth.
>
> **(2) The derived-tile app shape.** `AppListItem` gains **`parent_app_id`** (uuid or null, always
> serialized) and `AdminApp` gains **`origin`** (`"manual" | "discovered"`, read-only) and
> **`library_provider`** (`"" | "steam"`). `AppWrite` accepts `parent_app_id` (tri-state on patch,
> like `runtime_preset_id`) and `library_provider` (explicit `""` valid, like `external_source`) —
> **but never `origin`**, which is provenance the server sets and the Phase 4 reconciler reads.
> Sending it is `400` — enforced by `DisallowUnknownFields()` rather than a field-specific branch,
> so the message is the generic malformed-body one (§Derived tiles).
>
> **`DELETE /v1/apps/{id}` gains a second refusal**, reusing the generic `conflict` code:
> deleting an app that has derived tiles is `409` with the tiles **listed** in
> `error.derived_tiles`, unless the caller sends `?delete_derived=true`. The FK cascade is the
> integrity backstop under that confirmation, not the UX. And **the new constraints answer `4xx`,
> never `500`** — `apps_derived_shape_ck` is `400 validation_failed`,
> `apps_parent_external_uk` is `409 conflict`.
>
> **A derived tile carries identity and presentation only and borrows everything executable from
> its parent at launch.** Its shape is a database `CHECK` (`apps_derived_shape_ck`):
> `runtime_spec = '{}'`, `managed_home = false`, `runtime_preset_id IS NULL`,
> `library_provider = ''`, and a non-empty `external_source` + `external_id`.
>
> > **Why the tile stores no runtime of its own.** Merging at launch rather than flattening at
> > save is what makes an edit to the Steam app — an image bump, a new GPU flag, a new mount —
> > propagate to **every** derived tile with no re-sync and no stale copies. It is the same
> > decision UI-P3 made for runtime presets, for the same reason. It is a `CHECK` and not a
> > convention because **a validated Tower experiment hardcoded a host path into a tile's
> > `runtime_spec.mounts`**; the constraint exists so that cannot ship, and so it survives an
> > admin editing the row directly.
>
> The tile's contribution to execution is exactly one thing: an env override,
> `STEAM_STARTUP_FLAGS = "-bigpicture -applaunch <external_id>"`, merged over the parent's
> `runtime_spec.env` at dispatch (parent first, tile second, **tile wins** — the same order
> UI-P3 established). **The tile's own resource columns are never read**: admission resolves
> `default_vram_mb` / `default_encode_slots` from the **parent**, because `cb97bfb` is a live
> incident of exactly the opposite and a derived tile is precisely the shape that took.
> `apps_parent_external_uk` makes one tile per `(parent_app_id, external_source, external_id)`
> **fleet-wide**, which is what keeps the catalogue bounded regardless of user count.
>
> **(3) Three `409`s on the launch paths — one widened, two new.** `home_in_use` is **not** a new
> code (P5-01 has emitted it since managed homes shipped); what changes is that it now keys on the
> **home-owning** app rather than on the tile, and that its body carries **`session_id`**.
> `home_not_provisioned` and `parent_app_disabled` **are** new. All three are `409` and all three
> are additive (`409` is already declared on both endpoints and `Error.code` is an open string),
> and §Derived tiles below carries the rules and the client requirement.
>
> **Both new codes reach `POST /v1/sessions/{id}/swap` as well as `POST /v1/sessions`**, and for
> `home_not_provisioned` the swap's constraint is the **narrower** one: a swap is pinned to the
> live session's host and has no placement step, so the parent's home must exist **on that host**
> specifically. A launch can pin to whichever host holds the home; a swap cannot move.
>
> **`parent_app_disabled` is a behaviour change ruled in at review, not a shape the spec asked
> for**, and it is recorded as such rather than as a footnote. **A tile's own `enabled` and its
> parent's `enabled` are ANDed at launch.** The implementation spec's field table lists `enabled`
> under *"lives on the tile"* and genuinely reads the other way — **the contract is the durable
> artifact here, and it is this**. That table says which *row* a field is read **from**; it does
> not say the parent's `enabled` is irrelevant. See §Derived tiles for the reasoning and for what
> a reviewer observed before it was fixed.
>
> Backed by `schema.md` (migration 0044: the five `apps` columns, the widened `apps_kind_check`,
> `apps_derived_shape_ck`, `apps_parent_external_uk`, `apps_external_ref_idx`) and `openapi.yaml`
> (`AppKind`, the new `AppOrigin` / `LibraryProvider`, `AppListItem.parent_app_id`,
> `AdminApp.origin` / `.library_provider`, `AppWrite.parent_app_id` / `.library_provider`,
> `Error.session_id`). **`agent-api.md` is byte-identical** — a derived tile is invisible from the
> agent side by construction, exactly as a runtime preset is: the agent still receives one opaque,
> already-merged `app` object in `session_assign` / `session_swap_app`. `signaling.md`, `input.md`,
> `native-client.md` are unchanged. **Rollout order: control plane before client**
> (`crud.decodeJSON` sets `DisallowUnknownFields()`, so a client sending `parent_app_id` to a
> control plane without this amendment is a hard `400`, not a silent ignore), and the client
> re-pin is scheduled **with this phase** because of the `AppKind` widening above. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §1–§5, §4.5 and §13 "Phase 3".

> **Amendment — Steam library discovery, Phase 4 (discovery, the denylist, and the operator
> setting). Additive, requires sign-off. Migration 0045.** Adds the §Library discovery section:
> **seven new routes** — two **agent-authenticated** (`GET /v1/agent/library/scan-pending`,
> `POST /v1/agent/library/scan-report`) and five **admin** (`GET /v1/admin/library/status`,
> `GET /v1/admin/apps/{id}/library/unpublished`, `GET`/`PUT`/`DELETE` on
> `/v1/admin/apps/{id}/library/rules`) — plus **one field**, `library_discovery_enabled`, on
> `GET`/`PATCH /v1/admin/settings`. **No existing shape, status code, endpoint or behaviour
> changes, and no new error code** (`validation_failed` 400, `unauthorized` 401, `not_found` 404
> and `conflict` 409 are all reused).
>
> **`{id}` on the four `/v1/admin/apps/{id}/library/*` routes must be a library provider, and the
> refusal is `400`, not `404` and never `200`** (§Library discovery). These routes are keyed on
> `(parent_app_id, external_source, external_id)` and the reconciler only reads rules whose parent
> carries `library_provider='steam'`, so a rule written against an ordinary app would store, be
> acknowledged, and be **permanently inert** — a success response for a request that can never have
> an effect, handed to the one operator already hunting for why a rule "isn't working". `400`
> rather than `404` because the app is **real**: those are two different mistakes with two
> different fixes, and collapsing them sends someone looking for a missing app that is sitting
> right there. The check reads **`library_provider`, never `kind`** — §Derived tiles' promise that
> no server path branches on `kind` cuts both ways here, so an app *styled* as a Launcher without a
> provider set must fail it.
>
> **This one really is additive, and the contrast with Phase 2 is the point.** Phase 2 introduced
> an authorization object and then made five existing endpoints obey it; that is why it was
> recorded as breaking and signed off as such. Phase 4 adds a surface and changes nothing that
> already existed. What it *does* is start **writing** the `granted_by='provider'` entitlement
> rows Phase 2 put in the `CHECK` and left unwritten — a value the contract already permitted,
> arriving through a new route, with no shape change anywhere.
>
> **`agent-api.md` IS BYTE-IDENTICAL, and that is the headline architectural result of the whole
> phase rather than an absence worth skipping.** Discovery has to run agent-side — the control
> plane never touches a host filesystem (invariant #1) — so the obvious design is a new WebSocket
> message on the frozen agent contract. It reuses the existing **`/v1/agent/storage/*` pull
> channel** instead, exactly as the #175 GC amendment did, and §7.2 of the spec weighed both
> before choosing:
>
> | | new `agent-api.md` WS message | HTTP pull, here |
> |---|---|---|
> | contract | frozen interface, Opus + sign-off, pin bumps here **and** in `quasar-client` | additive HTTP surface; #175 is the precedent that records `agent-api.md` byte-identical |
> | push vs pull | the control plane must know the agent is connected and hold the correlation | the agent asks when it is ready; **nothing to correlate across a reconnect** |
> | restart | an in-flight scan dies with the socket | a claim is a **database row**; a stale one is reaped by timeout |
> | prior art | none for a long-running batch job | `spawn_gc_reaper` does exactly this shape today |
> | does the agent learn a user | it would have to, or carry an opaque token that amounts to one | **never**, by construction |
>
> The whole feature — a filesystem walk on every user's home on every host, a fleet-wide
> suppression ladder, and per-user entitlement grants and revokes — is implementable with the
> agent contract untouched. `signaling.md`, `input.md`, `native-client.md` are likewise unchanged.
>
> **The agent never learns a user, and that is a guarantee of the interface rather than a property
> of the current implementation.** Neither direction of the pull channel carries a user id, a
> username, or any user-derived field: the job payload is a scan id, an opaque home path, two
> relative roots and two bounds, and the report is a scan id, an `ok` flag and a list of installed
> titles. The control plane holds the `scan_id → (user, app, host)` mapping in `library_scans`
> and resolves it on receipt. `agent-api.md` records the **P2-01 verdict that per-user concerns
> never reach the agent**; this honours it literally, and a change to this surface that put a user
> on the wire would be the wrong change even though the JSON would validate.
>
> **PII: the report shape is closed, and the reason it is closed is `LastOwner`.** Every
> `appmanifest_*.acf` carries a **SteamID64** — a persistent, globally unique, externally
> resolvable identifier for a real person's Steam account. The agent parses with a **key
> allow-list** (`appid`, `name`, `installdir`, `SizeOnDisk`, `StateFlags`) and **never a
> denylist**, because Steam adds keys over time and a denylist leaks every future key by default;
> and the report entry has exactly those five fields, so **there is no field for a sixth value to
> travel in** even if the parser were wrong. Widening either of those two things is the change
> this paragraph exists to make someone stop and think about.
>
> **Two of the five report fields have no consumer, and a reader should not assume otherwise.**
> `install_dir` and `size_on_disk` are **collected, not used**. `install_dir` existed to feed
> launch verification, which operator decision 3 dropped (spec §16.1); `size_on_disk` was never
> consumed by anything. They stay because narrowing a PII containment allow-list buys nothing
> while widening it later means re-touching the one piece of code whose job is to touch as little
> of the manifest as possible. `state_flags` is likewise unread — on the live capture it was `4`
> for all five Valve tools *and* for three of the four real games, so it distinguishes nothing.
>
> **The second-order PII, said out loud.** `library_observations` and the provider-written
> `entitlements` rows are, together, **a per-user record of which games a person has installed**.
> That is inherent to the feature the operator asked for, admins can see it by design (the
> entitlement UI and the "Seen, not published" read both expose it), and it is **new** — nothing
> in Quasar recorded a per-user game inventory before this phase. It is only appids and titles,
> and it cascades away with the user (`ON DELETE CASCADE` on `user_id` throughout), but it is
> worth naming rather than discovering.
>
> **Suppression is a hide, never a delete** (§Library discovery). An `ignore` rule writes the rule
> row, disables the tile, and revokes that tile's `granted_by='provider'` entitlements — three
> things, none of them a `DELETE`, because `apps` cascades to `user_app_favourites` and
> `app_artwork`, so deleting a junk tile destroys **every user's favourite of it and its artwork
> row, irreversibly** — and the next scan re-creates a bare row anyway, because the appid is still
> on disk.
>
> **`library_discovery_enabled` is the master switch and the only switch.** Under operator
> decision 1, auto-publish is the **behaviour**, not a mode: there is no review queue, so a second
> "publish" toggle would only select between publishing and a path that was never built. Two env
> knobs sit beside it and neither is a second switch: **`QUASAR_LIBRARY_SCAN_INTERVAL`** (default
> `6h`; `0` forces discovery dark regardless of the database) and the opt-in, **default-off**
> `QUASAR_STEAM_APPDETAILS_LOOKUP`, which discloses this instance's installed appids to a third
> party — the same trade the artwork work rejected for hotlinking, which is why it is an
> operator's decision and never a default.
>
> **P5 side effect (image management):** flipping `library_discovery_enabled` `false→true`
> **auto-installs** every catalog image with a non-null `library_provider` (the canonical Steam
> image) that is not already installed — idempotent, via the P3 install path (adopt +
> ensure-everywhere + runtime-preset materialization), so enabling discovery guarantees the
> provider's image and preset are present rather than failing at a later launch. A provider image
> that cannot resolve its digest (private registry) stays `digest_unresolved` and is surfaced in
> `GET /v1/admin/images`, never fatal to the setting flip. Disabling discovery does **not**
> uninstall (destructive; an admin uninstalls explicitly). See the image-management P5 spec.
> `GET /v1/admin/images` per-image gains `runtime_preset_id` — the managed preset materialized
> from the manifest `runtime` block at install (`schema.md` migration 0058), null until installed.
>
> **Amendment — admin-libraries (2026-08-01), additive, signed off.** The two env knobs above
> gain **database-settable counterparts** so the admin UI's Steam library page can configure
> them: `instance_settings.library_discovery_interval_minutes` (integer, 15–10080, default 360;
> out-of-bounds `PATCH` is 400 `validation_failed`) and
> `instance_settings.library_discovery_appdetails_enabled` (boolean, default false). Both appear
> on `GET /v1/admin/settings` (always) and are accepted by `PATCH /v1/admin/settings` (absent =
> unchanged, the same pointer-decode rule as `library_discovery_enabled`). **The env vars are
> overrides, not defaults**: when set, `QUASAR_LIBRARY_SCAN_INTERVAL` wins — its `0` = dark
> regardless-of-the-database semantics unchanged — and `QUASAR_STEAM_APPDETAILS_LOOKUP` wins,
> so a privacy-hardened deployment can pin the lookup off in the environment.
> `LibraryStatus.scan_interval_secs` and `.appdetails_lookup` stay the **resolved** values, and
> `LibraryStatus` additionally reports `interval_overridden_by_env`,
> `appdetails_overridden_by_env` (so a UI greys a control the environment pinned) and
> `last_scan_completed_at` (nullable; when the most recent scan finished, the one-glance
> companion to the counters). **No route, status code, or error code changes; `agent-api.md`
> untouched.**
>
> **Amendment — scan observability + backfill (2026-08-01, same-day follow-on), additive,
> signed off.** Two things one idea: make a scan's work **visible** and make it **complete-able**.
> (1) Per-scan outcome counts (`observed, suppressed, created, disabled, granted, revoked,
> rejected, backfilled`) are **stored on the scan row** at reconcile (migration 0048 — rows from
> before it read zero, which a UI presents as "not recorded") and surfaced as
> `LibraryStatus.recent_scans` (last 20 terminal scans, newest first, with `user`/`host` names
> and the failure `error`). Under auto-publish, "nothing appeared" and "nothing ran" were
> indistinguishable to an operator whose library was already fully published; this closes that
> at the per-scan grain, the same failure `inert_reason` closes at the instance grain.
> (2) Reconcile gains a **backfill** step: existing tiles of the scanned parent whose enrichable
> fields are **empty** (`description`, initially) are filled from the same appdetails source the
> suppression rung uses — gated by the **same** appdetails switch (off ⇒ no backfill, no
> third-party call), bounded per scan, and **fill-blanks-only** (a non-empty field is never
> overwritten; operator edits survive every scan). "Scan now" thereby also becomes "fetch
> newly-supported data for tiles I already have" as enrichment grows — no delete-and-rediscover.
> **No route, status code, or error code changes; `agent-api.md` untouched** (the agent's report
> body is unchanged; backfill is server-side reconciliation behaviour).
>
> Backed by `schema.md` (migration 0045: `library_scans`, `library_observations`,
> `library_appid_rules`, and `instance_settings.library_discovery_enabled`) and `openapi.yaml`
> (the seven paths, the `Library*` shapes, and the settings field). **Migration numbering:** the
> spec says 0044 for this phase throughout; Phase 3 shipped first and took 0044, so Phase 4's
> tables land at **0045**. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §7, §8, §9, §10, §11 and §13
> "Phase 4".

> **Amendment — Steam library discovery, the force scan (Phase 4 follow-on). Additive,
> admin-gated, requires sign-off. No migration.** Adds **one route** to §Library discovery —
> `POST /v1/admin/library/scan` — and documents one **behaviour** on an endpoint whose shape does
> not move: a `PATCH /v1/admin/settings` that flips `library_discovery_enabled` **false→true** now
> causes a janitor pass promptly rather than at the next six-hourly tick. **No existing shape,
> status code, endpoint or error code changes** (`validation_failed` 400, `unauthorized` 401,
> `forbidden` 403 and `not_found` 404 are all reused), no new table and no new column. **`agent-api.md` is
> byte-identical**, verified by blob hash against the published Phase 4 contract.
>
> **What this closes is two timers that were never one idea.** Discovery shipped paced by two
> independent six-hour clocks, and **each is anchored to its own process's boot rather than to the
> moment an operator enables the feature**: the control-plane janitor decides when a scan is
> *enqueued*, and the node-agent's poll decides when a queued scan is *claimed*. Enabling discovery
> could therefore take **up to twelve hours** to produce a single tile. And because this is a
> **pull** channel by deliberate design (§Library discovery, "Where it runs, and why it is not a
> WebSocket message"), **the control plane cannot push a scan to an agent** to shorten it — the
> property that bought `agent-api.md` byte-identical is the same property that makes latency the
> operator's problem. Half a day of an unchanged library is indistinguishable from the feature being
> broken, which is precisely the failure `inert_reason` exists to close, arriving through **timing**
> rather than configuration.
>
> **Two changes fix it, and they are one idea rather than two.** First, **the agent's poll drops
> from 6h to 60s**. **Poll cadence is not scan cadence, and conflating them was the defect**: a poll
> is one indexed query against `library_scans` scoped to the calling host, returning an empty list
> almost always, while the expensive half — a filesystem walk over every user's home on every host —
> stays paced by `QUASAR_LIBRARY_SCAN_INTERVAL` exactly as before. That is **agent behaviour, not
> wire shape**: the two payloads, the claim semantics, the bounds and the node-secret auth are all
> untouched, which is why `agent-api.md` does not move and no `quasar-client` pin does either.
> Second, the route below — which is only *worth* having because of the first. A "scan now" button
> in front of a 6h poll would have made an operator wait six hours for their own button.
>
> **`POST /v1/admin/library/scan` bypasses pacing, and never a gate.** It enqueues `pending` scans
> immediately, dropping the janitor's *"no successful scan inside the interval"* recency check —
> **that bypass is the entire point**, because an operator pressing this has a reason the recency
> rule cannot know about (they just installed a game, or just fixed a home mount). Everything the
> janitor *gates* on it still respects **and reports**: the `library_discovery_enabled` switch, the
> `volume` storage provider (§Storage-driver limitation), `QUASAR_LIBRARY_SCAN_INTERVAL=0` as the
> hard kill switch, and the per-home `local` driver filter. A force scan overrides how *often*
> discovery runs, never *whether* an operator allowed it to.
>
> **Inert is a `200` carrying `inert_reason`, not a 4xx**, mirroring `GET /v1/admin/library/status`
> — whose `inert_reason` exists for exactly this — so a client renders one code path for "here is
> what happened" and "here is why nothing did". The server extracts a **shared helper** for the
> reason strings, so the two surfaces cannot drift in wording; an operator reading `queued 0` on the
> button and the sentence on the status panel must not be reading two different stories.
>
> **`eligible` is the non-obvious field and it is contractual.** `queued: 0` with `eligible > 0`
> means *everything is already queued — wait*; `queued: 0` with `eligible: 0` means *your scope
> matched nothing — fix the scope*. **Two zeros with opposite remedies**, and without `eligible` a
> client cannot tell them apart. This is the same class of problem §Storage-driver limitation exists
> for: under auto-publish, silence is ambiguous, so every path through this endpoint says something.
>
> Scoping by `app_id` and/or `user_id` is optional and **an empty body is valid**, meaning
> "everything, now" — the common case and what the admin button sends. A non-provider `app_id` is
> `400`, reusing the rule the four `/v1/admin/apps/{id}/library/*` routes already apply rather than
> inventing a second vocabulary for the same mistake. Double-press is **idempotent**: the open-scan
> unique index is partial on `pending`/`claimed`, so a second press inserts nothing, reports it as
> `skipped`, and is not an error. **Audited as `library.scan.force`**, identifiers and counts only:
> this makes the whole fleet walk every user's home directory on demand, which is exactly the kind
> of action someone later asks *"who did that"* about.
>
> **The on-enable nudge is documented because a client author would otherwise assume the opposite.**
> `PATCH /v1/admin/settings` does not change shape, but flipping `library_discovery_enabled`
> false→true now runs a janitor pass promptly instead of at the next 6h tick; true→true and
> false→false do not, so re-saving a settings form does not re-walk the fleet. The **transition** is
> detected by reading the previous value `FOR UPDATE` in the same transaction as the write, so two
> admins saving concurrently cannot both observe false→true. Left undocumented, a client author
> reasonably assumes enabling is inert until the next tick — which is the wrong expectation, and the
> one this amendment exists to fix.
>
> Backed by `openapi.yaml` (the one new path and the `LibraryForceScan*` shapes). No `schema.md`
> change: the force path writes `library_scans` rows the Phase 4 DDL already defines.

> **Amendment — a fourth `inert_reason`: no app is marked as a library provider. Additive,
> admin-gated, requires sign-off. No migration, no new route, no new field, no new error code and
> no shape change of any kind** — the only thing that moves is the **set of values** the existing
> `inert_reason` string on `GET /v1/admin/library/status` and `POST /v1/admin/library/scan` may
> take, which this document previously enumerated as exactly three. **`agent-api.md` is
> byte-identical**, verified by blob hash.
>
> **This is the first-run state, and it produced total silence on a real deployment.** An operator
> switched `library_discovery_enabled` on and *nothing happened* — no scan rows, no observations,
> no log line beyond the janitor's startup message. The cause was that no app carried
> `library_provider='steam'`, so the janitor's enqueue joined against nothing and matched zero
> rows. That is the **single most likely first-run state**, because an operator naturally flips the
> settings toggle first and only afterwards wonders what else was needed — and §Storage-driver
> limitation's principle already covers exactly this: under auto-publish, *"nothing appeared"* and
> *"nothing ran"* are indistinguishable to an operator, so **the reason has to be surfaced**. Three
> inert causes already were. This fourth one is the one that actually bit.
>
> **It is a REASON, not a gate, and the distinction is contractual.** The three existing causes mean
> no work *may* be done; this one only predicts that the eligibility query will match nothing, which
> stays a **normal, non-error outcome**. Enqueue behaviour does not change, a force scan against
> such an instance is still a `200`, and **zero eligible triples is still not an error**. The
> control-plane janitor likewise *reports* it and carries on with its pass rather than returning
> early — the case where an operator **un**marks their last provider app is precisely the case that
> needs the stranded-scan expiry an early return would skip.
>
> **Ordering is observable and therefore contractual: it is reported LAST of the four.** With the
> switch off *and* no provider app, the reported reason is **the switch**. The three above it are
> instance-level facts that make the provider question moot, and telling someone who has the feature
> switched off that they also have no provider app would send them configuring an app on a control
> plane that would have ignored it anyway. Same rule, same reason, as *"inertness is answered before
> the scope"* above.
>
> **The message names the remedy**, as the non-provider-`app_id` `400` already does: *set Library
> provider to Steam on your Steam app (Identity section of the app editor)*. An operator told only
> that discovery is inert has been moved from silence to a sentence, which is barely an improvement.
> The usual rule still holds — **clients must not parse `inert_reason`**; it is human-readable prose
> rendered verbatim, and only its presence is contractual.
>
> Both admin surfaces answer with the **identical string**, computed by the one shared server-side
> helper the force-scan amendment introduced, and the janitor logs the same reason **once** rather
> than every six hours. Backed by `openapi.yaml` (two description updates; no schema, path, method
> or required-field change, so `TestOpenAPIDrift` sees nothing move — which is correct, and is the
> known limit of that gate rather than a gap in this amendment).

## Conventions
- Base path **`/v1`**. JSON request and response bodies; `Content-Type: application/json`.
- TLS in deployment (architecture: control plane is the public ingress). The web client derives
  the API origin from `location.origin`, mirroring the `signaling.md` rule for the WS URL.
- **Auth:** all endpoints except `/v1/auth/register` and `/v1/auth/login` require
  `Authorization: Bearer <access_token>` (the opaque token from `auth_tokens`, see `schema.md`).
  Missing/invalid/expired/revoked ⇒ `401`.
- **Tokens, never passwords, on the wire** beyond the two auth endpoints (P1-2). The password is
  sent only to register/login over TLS; everything else is bearer-token.
- **Client version (#380, additive):** a bearer-authenticated request MAY carry
  `X-Quasar-Client-Version: <MAJOR.MINOR.PATCH>`. Absent ⇒ no gate (web/legacy behaviour,
  exactly as an omitted `client_version` at login). Present and below the operator floor ⇒
  `426 client_too_old`, enforced in the shared auth middleware. See
  §Client version gate on bearer-authenticated endpoints.
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
  single-writer. *Steam library discovery Phase 3:* the key is the **home-owning** app, so a
  derived tile collides with its parent and with its siblings, and the body carries `session_id`,
  the conflicting session — see §Derived tiles), `home_not_provisioned` (409, *Steam library
  discovery Phase 3* — a derived tile was launched by a user who has no home for the **parent**
  app on any host; the tile provisions nothing, so no home is created and no `user_homes` row is
  written. **Also emitted by `POST /v1/sessions/{id}/swap`**, where the constraint is narrower:
  the swap is pinned to the *live session's* host and has no placement step, so the parent's home
  must exist **on that host**), `parent_app_disabled` (409, *Steam library discovery Phase 3* —
  a derived tile was launched or swapped into while the **parent** app it borrows its runtime and
  home from is disabled; the message names the parent, and the caller cannot act on it), 
  `profile_ineligible` (409, *AS10-03* — a user-facing launch selected a stream
  profile that is `ineligible` for the caller's device, or a non-user-facing profile without an
  admin/explicit-override bypass), `profile_not_launchable_for_app` (409, *UI-P5* — the selected
  launch profile is valid and eligible but is not in the app's `launchable_profile_ids` allow-list;
  the caller's menu is stale, so re-read `GET /v1/me/profiles?app_id=…` and re-pick),
  `restart_required` (409, *host-runtime-settings* — a `PATCH` to
  `/v1/admin/hosts/{id}/settings` changed a restart-class knob while the host has live sessions
  but `restart_confirm` was not `true`; includes `{ "live_sessions": N }` in the error body),
  `rate_limited` (429), `client_too_old` (426, *P9-08 / #380* — the presented client version is
  below the operator-configured floor; the body carries `min_client_version` (+
  `latest_client_version` when configured) as top-level siblings of the `error` envelope),
  `no_host_available` (503,
  *P2-01* — no online host/GPU can serve the request), `capacity_exhausted` (503, *P2-01* — a
  matching GPU is online but its free encode slots / VRAM cannot satisfy the request right now),
  `internal` (500). **Retryable:** `no_host_available`, `capacity_exhausted`, `rate_limited`
  (and `session_quota_exceeded` / `home_in_use` once one of the caller's sessions ends).
  **`home_not_provisioned` is NOT retryable** — retrying reproduces it forever. It is resolved by
  a user *action* (launch the parent app once, so its home exists), which is why the message names
  the parent rather than saying "try again". **`parent_app_disabled` is not retryable either, and
  is not even resolvable by the caller** — it takes an *operator* re-enabling a different app.
  These two are the only refusals on the API whose remedy lies outside the caller's reach, which
  is exactly why neither is folded into the generic `conflict`.
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
`404`). *(#380)* The client-version gate sits **between** those two checks — `401`, then
`426 client_too_old`, then `403` — for the same reason the `403` precedes the lookup: each
refusal is decided by the cheapest fact that settles it, and neither tells an unauthenticated
caller anything. See §Client version gate on bearer-authenticated endpoints.

| endpoint | required role | notes |
|---|---|---|
| `POST /v1/apps` | **admin** | create an app — *(UI-P5)* including its `launchable_profile_ids` allow-list |
| `PATCH /v1/apps/{id}` | **admin** | edit an app — *(UI-P5)* including its `launchable_profile_ids` allow-list. **UI-P5 adds no new route**: the allow-list rides these two, which are already `RequireAuth → RequireAdmin`, so a non-admin bearer is `403` before the field is ever looked at |
| `DELETE /v1/apps/{id}` | **admin** | *(admin-delete)* remove an app from the catalog — refuse-if-in-use |
| `GET /v1/admin/apps/{id}/entitlements` | **admin** | *(Steam library discovery Phase 2)* who may see and launch this app — the `all` row first, then the personal grants |
| `POST /v1/admin/apps/{id}/entitlements` | **admin** | *(Phase 2)* grant — `{subject_type, subject_id}`; `409 conflict` if that subject already holds one |
| `DELETE /v1/admin/apps/{id}/entitlements/{entitlement_id}` | **admin** | *(Phase 2)* revoke — **scoped to the app in the path**, so an entitlement id belonging to another app is `404`, not a cross-app delete |
| `GET /v1/admin/users/{id}/entitlements` | **admin** | *(Phase 2)* **this user's personal grants only** — deliberately *not* the `all` rows they also benefit from; see §Entitlements for why |
| `GET /v1/admin/library/status` | **admin** | *(Steam library discovery Phase 4)* is discovery actually doing anything, **and if not, why** — the switch, the interval, the storage provider, the opt-in lookup, an `inert_reason` and the per-state scan census |
| `POST /v1/admin/library/scan` | **admin** | *(Steam library discovery, force scan)* **"Scan now"** — enqueue `pending` scans immediately, **bypassing the janitor's recency pacing and nothing else**. Optional `app_id` / `user_id` scope; an **empty body is valid** and means everything. Reports `queued` / `skipped` / `eligible` / `inert_reason`; an inert instance is a `200` with the reason, not a 4xx, mirroring `GET /v1/admin/library/status`. A non-provider `app_id` is `400` and a non-existent one `404`, same rule as the four routes below; **inertness is answered before the scope**, so a bad `app_id` on a switched-off instance is that `200`. Idempotent. Audited as `library.scan.force` |
| `GET /v1/admin/apps/{id}/library/unpublished` | **admin** | *(Phase 4)* **"Seen, not published"** — every appid observed under this provider app that has no enabled tile, and which layer suppressed it. A read of one table and a button, not a review queue: nothing waits on it. **`{id}` must be a library provider: `400` for a real app that is not one, `404` only for one that does not exist** |
| `GET /v1/admin/apps/{id}/library/rules` | **admin** | *(Phase 4)* the operator-written layer-2 rules on this provider app. Same `{id}` rule: `400` for a real non-provider app, `404` for a missing one |
| `PUT /v1/admin/apps/{id}/library/rules/{external_id}` | **admin** | *(Phase 4)* **Ignore and un-ignore, one route, two directions** — `rule:"ignore"` suppresses fleet-wide (rule row + disable the tile + revoke its provider entitlements, in one transaction), `rule:"allow"` outranks the built-in denylist. The primary key is the idempotency key, so a repeat is a replace. Audited as `app.library.rule.set` |
| `DELETE /v1/admin/apps/{id}/library/rules/{external_id}` | **admin** | *(Phase 4)* drop a rule, returning the appid to whatever the built-in denylist says about it. Re-enables and disables nothing by itself — the next scan applies the ladder afresh. Audited as `app.library.rule.delete` |
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
| `GET /v1/admin/images` | **admin** | *(image management P1)* the app-image catalog with per-instance install + per-host presence state |
| `POST /v1/admin/images/sync` | **admin** | *(image management P1; P3 adds digest resolution + policy application)* re-fetch the manifest and refresh the cached catalog |
| `POST /v1/admin/images/{id}/install` | **admin** | *(image management P3)* adopt a catalog image + ensure-everywhere unless `lazy` — `409` already-installed / digest-unresolved |
| `DELETE /v1/admin/images/{id}/install` | **admin** | *(image management P3)* uninstall — best-effort `image_remove` to every host that has it |
| `POST` / `DELETE /v1/admin/images/{id}/pin` | **admin** | *(image management P3)* freeze/unfreeze the installed version against every update path — idempotent `204` |
| `POST /v1/admin/images/{id}/update` | **admin** | *(image management P3)* explicit update-now — `200` applied or no-op, `409` if pinned |
| `GET /v1/hosts`, `GET /v1/hosts/{id}` | **admin** | host/capacity oversight |
| `POST /v1/hosts/{id}/drain`, `POST /v1/hosts/{id}/uncordon` | **admin** | *(P3-01)* host lifecycle — cordon a host out of service / return it |
| `DELETE /v1/hosts/{id}` | **admin** | *(admin-delete)* forget an offline host — refuse-if-online-or-in-use |
| `GET /v1/apps`, `GET /v1/apps/{id}` | user | the library — **both reads require auth** *(UI-P1: the list was public until 2026-07-27; see the breaking-change amendment. `favourite` is resolved from the bearer identity, so an anonymous read could not answer it anyway)*. **(Phase 2: both are now entitlement-scoped — the list for every role including admin, the single read for non-admins as a `404`. This is not a role gate and produces no `403`; an entitlement is a per-subject grant, not a role. The unfiltered catalogue is `GET /v1/admin/apps`.)** |
| `GET /v1/sessions/{id}`, `GET /v1/sessions`, `DELETE /v1/sessions/{id}` | **owner or admin** | resource-ownership check (`403` otherwise), not a blanket admin gate |
| `POST /v1/sessions/{id}/swap` | **owner or admin** | *(P2-02)* same ownership check as `DELETE`. **(Phase 2: additionally `403 forbidden` when the **session owner** is not entitled to the target app — keyed on the owner, never on the caller, so an admin swapping someone else's session cannot launder their own entitlements into it)** |
| `PATCH /v1/sessions/{id}/display` | **owner or admin** | *(session-display-update; session-display-stream (approved 2026-08-16) adds external/stream resolution)* live render resolution / UI scale (and, DRAFT, external/encoded resolution) change — same ownership check as `DELETE`/`swap`; best-effort relay to the host agent, no session-state transition |
| `POST /v1/sessions/{id}/stats` | **owner or admin** | *(P4-01)* the client posts its own session's browser telemetry — same ownership check as `DELETE` |
| `GET /v1/admin/sessions/{id}/metrics` | **admin** | *(P4-01)* per-session telemetry read (oversight) |
| `POST /v1/me/devices` | user (self) | *(P4-01)* upsert the caller's own device capability; owner is the bearer identity, never a body field |
| `GET /v1/me/devices` | user (self) | *(AS10-08; **LP-SEC-01**)* read the caller's own devices — **now the full list** (was AS10-08 latest-only); owner is the bearer identity |
| `GET /v1/me/profiles` | user (self) | *(AS10-02; **UI-P4**: now evaluates **launch profiles**, each with its per-rung verdicts; **UI-P5**: optional `?app_id=` narrows the result to that app's allow-list — a convenience, **never the gate**, which is `POST /v1/sessions`)* eligibility + recommendation for the caller's device; owner is the bearer identity |
| `PATCH /v1/me/profile-preferences` | user (self) | *(AS10-03; prose added UI-P4)* the caller's preferred **launch profile**; honoured only while the global policy allows user overrides |
| `GET` / `PATCH /v1/me/ui-preferences` | user (self only) | client UI presentation preferences, synced across the caller's devices. No admin variant exists: these are keyed on the authenticated caller and there is no `{id}` form, so one user's preferences are unreadable by anyone else. |
| `GET /v1/me/highlights` | user (self only) | *(home-rail amendment, 2026-08-05)* the caller's server-ranked home rail, derived from their own session history. No `user_id` parameter and no admin variant. **Entitlement-filtered**, and that filter is load-bearing rather than cosmetic — see below. |
| `GET /v1/admin/storage/homes` | **admin** | *(P5-01)* list managed homes (storage oversight) |
| `DELETE /v1/admin/storage/homes/{id}` | **admin** | *(P5-01)* tombstone a home for GC |
| `GET /v1/me/storage` | user (self) | *(P5-01)* the caller's own per-app storage usage |
| `PUT /v1/me/favourites/{app_id}` | user (self) | *(UI-P1)* favourite an app; owner is the bearer identity — no endpoint takes a `user_id`. Idempotent `204`; `404` under the same visibility rule as `GET /v1/apps/{id}`. **(Phase 2: `403 forbidden` when the caller is not entitled — checked *after* the `404`, and applied to every role including admin, because `/v1/me/*` is the user surface by definition)** |
| `DELETE /v1/me/favourites/{app_id}` | user (self) | *(UI-P1)* unfavourite; idempotent **and unconditional** `204` for a well-formed UUID — deliberately never `404` |
| `POST /v1/me/password` | user (self) | *(CP-01)* change the caller's own password; subject is the bearer identity, never a body field. Revokes all active tokens on success — client must re-authenticate |
| `GET /v1/admin/settings` | **admin** | *(LP-SEC-01)* read instance settings (`registration_mode`, `storage_provider`, **`library_discovery_enabled`** *(Phase 4)*, …) |
| `PATCH /v1/admin/settings` | **admin** | *(LP-SEC-01)* update instance settings — how invites are enabled/disabled from the UI. *(Phase 4: also the discovery master switch. Every field is an **optional pointer**, so a `PATCH` that omits one leaves it alone — a plain bool would decode an absent `library_discovery_enabled` to `false` and silently switch discovery off every time an admin changed the registration mode)* |
| `GET /v1/admin/secrets` | **admin** | *(secrets facility)* list the declared secrets + configured/readable/origin + a **masked** hint — never a value |
| `PUT /v1/admin/secrets/{name}` | **admin** | *(secrets facility)* set/replace a secret's value; write-only on the wire, response is the status shape |
| `DELETE /v1/admin/secrets/{name}` | **admin** | *(secrets facility)* clear a stored secret; any declared env-var fallback takes effect again |
| `GET /v1/admin/access-check` | **admin** | *(wizard v2 §S6b)* diagnose **this request's** reachability — cert SAN coverage, fingerprint, days-to-expiry, secure-context state, and whether the origin would pass `/v1/signal`. Reflected `Host`/`Origin` are **length-capped**; `X-Forwarded-Proto`/`Forwarded` are read to detect an upstream-terminating proxy and **may only soften advice, never authorise** |
| `GET /v1/tls/certificate.pem` | **none — public** | *(wizard v2 §S6a)* **the one deliberate exception in this table.** A client that does not yet trust the certificate frequently cannot log in *in order to* fetch it, so auth would make the route useless exactly when it is needed. It serves the **public** half every TLS handshake already transmits, and it is structurally incapable of emitting the key (the response is re-encoded from the parsed leaf's DER) |
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
| `GET /v1/admin/sessions/{id}/diagnostic-bundle` | **admin** | *(ST-01/ST-06)* the assembled bundle — metadata + clock + aligned series + events + derived windows + classifier verdict + `ingest` rejection counters |
| `GET /v1/admin/sessions/{id}/verdict` | **admin** | *(ST-09)* the **Verdict** alone — state + evidence + reason + window + clock + tier + falsifiers. Observability only; no session authority. Same window parameters and clamps as the bundle |
| `GET /v1/sessions/{id}/verdict` | **owner or admin** | *(ST-09)* the same Verdict for the caller's **own** session — resource-ownership check (`403` otherwise), the same one `DELETE /v1/sessions/{id}` applies; rate-limited per session like the other owner-scoped telemetry routes |
| `POST /v1/admin/sessions/{id}/trace/annotations` | **admin** | *(ST-01)* an operator annotation marker on the trace timeline |
| `POST /v1/admin/sessions/{id}/capture` | **admin** | *(session-capture)* arm ONE bounded observation of a live session — `202` with the `capture_id`. Admin-only and **observability-only**: it moves no session state, and `403` precedes the lookup as everywhere else in this table. `409` busy / not running, `422` unknown or unsupported kind, `501` the host's agent predates captures, `503` its agent is not connected |
| `GET /v1/admin/sessions/{id}/captures/{capture_id}` | **admin** | *(session-capture)* read that capture's stored result — `404` until it arrives, which is the poll signal, not an error |
| `POST /v1/sessions/{id}/trace/events` | **owner or admin** | *(ST-01)* the client posts its own session's browser trace events — same ownership check as `POST .../stats`; `202` on accept |
| `POST /v1/sessions/{id}/trace/clock` | **owner or admin** | *(ST-01/ST-05)* the client posts its own session's client↔host clock-offset estimate — same ownership check as `POST .../stats`; `202` on accept |
| `GET /v1/admin/hosts/{id}/encoder-certification` | **admin** | *(SPT-05)* read a host's encoder-certification verdicts (latest per configuration) |
| `POST /v1/admin/hosts/{id}/encoder-certification/runs` | **admin** | *(SPT-05)* open a certification run (reserve the per-host lock, return the cell plan) |
| `POST /v1/admin/hosts/{id}/encoder-certification/cells` | **admin** | *(SPT-06)* launch one pinned bench cell |
| `POST /v1/admin/hosts/{id}/encoder-certification/cells/{sid}/finalize` | **admin** | *(SPT-06)* derive the verdict from real agent metrics, upsert + teardown |
| `POST /v1/admin/hosts/{id}/encoder-certification/runs/{run_id}/complete` | **admin** | *(SPT-06)* close the run (release the per-host lock) |
| `GET /v1/admin/hosts/{id}/encoder-certification/runs/{run_id}` | **admin** | *(SPT-05)* poll a run's status/progress |
| `GET /v1/admin/jobs` | **admin** | *(jobs framework)* every registered background job with its resolved schedule and run state — **including `managed: false` rows**, work that exists in code but is not adopted, so the page cannot hide what it exists to show |
| `GET /v1/admin/jobs/{job_id}` | **admin** | *(jobs framework)* one job. `{job_id}` is the code-owned dotted id (`artwork.sweep`), **not a uuid** |
| `PATCH /v1/admin/jobs/{job_id}` | **admin** | *(jobs framework)* edit the **admin-owned** half of the schedule (`enabled`, interval, window, timezone, history limit) — code-owned identity is never writable here. **`409 job_unmanaged`** for an unadopted job, **`409 schedule_locked`** when an env override is authoritative, `422 validation_failed` for a value the schedule model refuses, `400 validation_failed` for an **unknown key**. Audited as `job.update` |
| `POST /v1/admin/jobs/{job_id}/run` | **admin** | *(jobs framework)* queue a manual run (`202`) — **bypasses the window, never the job's own gates**. `409 job_already_running` / `409 job_disabled` / `409 job_unmanaged`. `host_id` required for a host-scoped job, refused for an instance-scoped one. Audited as `job.run` |
| `GET /v1/admin/jobs/{job_id}/runs` | **admin** | *(jobs framework)* that job's bounded run history, newest first; optional `host_id` narrows to one target |
| everything else (`/v1/me`, `POST /v1/sessions`, …) | user | any authenticated account. *(UI-P5: `POST /v1/sessions` additionally refuses a `profile_id` outside the app's allow-list with `409 profile_not_launchable_for_app` — a per-app configuration rule, not a role check, which is why it is `409` and not the `403` this table's admin rows produce. It is refused for **every** non-admin caller, including one supplying an explicit `stream` override, which carries no role gate here.* ***Phase 2:** `POST /v1/sessions` additionally refuses an app the caller holds no entitlement for with `403 forbidden`. That one **is** `403` and not `409`, because unlike the allow-list it is a statement about the **caller** rather than about the request — and unlike this table's admin rows it is refused for **every** role, admin included)* |

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
> client owns; `contract_version` is the `protocol/` version tag the client built against.
> **#380 extends the hard gate past login onto every bearer-authenticated endpoint, carried by
> the optional `X-Quasar-Client-Version` header — see §Client version gate on
> bearer-authenticated endpoints. The login shape above is unchanged.***

> *(LP-SEC-01, additive) `device_key` is an **optional** request field. When present, the server
> upserts the `(user_id, device_key)` `user_devices` row and **stamps the minted
> `auth_tokens.device_id`** so the token is revocable per-device (`schema.md`). When absent
> (legacy/native clients), behaviour is exactly as today: the token is minted with `device_id =
> NULL` and is not device-revocable until a device-declaring re-login. The response shape is
> unchanged.*

### Client version gate on bearer-authenticated endpoints

> **Amendment — #380 (P9-08 follow-up), additive, requires sign-off.** P9-08 gates only
> `POST /v1/auth/login`, so a client holding a cached token is never re-checked: raising the
> floor has no effect on it until the token expires. This amendment gives the version a
> carrier on non-login requests so the floor is enforced *server-side* on every
> authenticated call, per the server-enforced-authorization rule in §Authorization.
> **Purely additive: one new OPTIONAL request header, no new endpoint, no new error code
> (`client_too_old` already exists from P9-08), and no change to any existing request body,
> response body, or status code.** `agent-api.md`, `signaling.md`, `input.md` and `schema.md`
> are byte-identical — the header carries no state and nothing is persisted.

**The header.** A client MAY send, on any endpoint that requires
`Authorization: Bearer <access_token>`:

```
X-Quasar-Client-Version: 1.2.0
```

The value is a strict `MAJOR.MINOR.PATCH` semver string the client owns — the same grammar and
the same meaning as `client_version` in the login body (pre-release / build metadata is not
part of the grammar). There is no header twin of `contract_version`; that field stays a
login-only handshake value.

**Normative rule (server-enforced, uniform).** The gate is applied **in the shared auth
middleware** — the same `RequireAuth` layer every bearer endpoint is wired through, exactly as
`RequireAdmin` is the one place the admin role is checked. It is **never** re-implemented per
handler, and no endpoint is exempt. Evaluation order on a bearer request is:

1. missing / invalid / expired / revoked token ⇒ `401 unauthorized` (unchanged);
2. **version gate** ⇒ `426 client_too_old` (this amendment);
3. role gate ⇒ `403 forbidden` (unchanged, §Authorization);
4. handler.

The version gate runs **after** authentication and **before** the role check, so an
unauthenticated caller can never probe the configured floor, and a too-old client gets the
actionable `426` rather than a `403` it cannot act on.

**Gate decision** — identical to login's, given the operator floor `min_client_version`:

| header | floor configured? | outcome |
|---|---|---|
| absent | either | **proceed** — legacy / web client, no gate (same as an omitted `client_version` at login) |
| present, ≥ floor | yes | proceed |
| present, < floor | yes | **`426 client_too_old`** |
| present, any value | no floor | proceed |
| present, **not valid semver** | yes | **proceed** (treated as absent); the server SHOULD log a warning |

> *The malformed case deliberately diverges from login, where an unparseable `client_version`
> is gated. At login the client presents itself once, at a credential boundary, and a refusal
> costs it one retry. On the bearer path the header rides **every** request, so gating a
> malformed value would take a client that is already signed in and, on a single typo or an
> unexpected suffix in its version string, brick every call it makes — including the ones it
> would use to update itself. It also buys nothing: the gate is cooperative, and a client that
> wants to evade it can simply omit the header. So a malformed header is treated as absent and
> logged, not gated.*

**`426` body** — byte-identical to login's `426`, so a client parses it from one place
regardless of which call produced it:

```json
{
  "error": { "code": "client_too_old",
             "message": "client version is below the minimum supported version; please update" },
  "min_client_version": "1.0.0",
  "latest_client_version": "1.3.0"
}
```

`min_client_version` is always present on a `426` (the gate only fires when a floor is
configured); `latest_client_version` appears only when the operator configured one. Both are
top-level siblings of the `error` envelope, matching the login `426` and the login `200`
advisory.

**Client obligation.** A native client SHOULD send the header on every authenticated request
once it knows its own version, and SHOULD treat a `426` from any endpoint the way it treats a
`426` from login: stop, surface the update prompt, and do not retry — the refusal is not
retryable and no other endpoint will answer differently. The web client sends nothing and is
unaffected.

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

## Dev-only agent auth (amendment #399, additive, dev-gated — signed off 2026-08-07)

> **Never part of the production surface.** The route below is registered ONLY when
> `QUASAR_DEV_AGENT_AUTH=1`; absent the flag it does not exist (404 from the mux, not a 403
> guard). The control plane **refuses to boot** (`fatal`) with the flag set while
> `QUASAR_ENV=production`. Every boot with the flag on emits a `WARN` banner naming the flag.
> In `openapi.yaml` the path carries `x-dev-only: true` and is excluded from the route-coverage
> drift test; its registration semantics (absent when off, present when on) are asserted by
> dedicated control-plane tests instead.

### `POST /v1/dev/agent-session`
```json
// request — header X-Quasar-Dev-Key: <per-boot secret>; body optional
{ "role": "user", "ttl_seconds": 1800 }
// 200 — login-shape + storage_keys
{
  "access_token": "<opaque>",
  "token_type": "Bearer",
  "expires_at": "<RFC3339 — token AND identity expiry>",
  "user": { "id": "<uuid>", "email": "agent-<uuid>@dev.invalid", "username": "...", "role": "user", "created_at": "..." },
  "storage_keys": {
    "quasar.auth.token": "...",
    "quasar.auth.expires_at": "...",
    "quasar.auth.user": "{...}"
  }
}
```

Mints a **throwaway, auto-reaped** identity for automated UI validation (unattended test
agents), so no shared long-lived credential exists and no real operator password is ever
handed to tooling. Semantics:

- **Auth** is the per-boot random secret (generated at startup, never persisted across
  boots, written to the container log and to `/run/quasar/dev-agent-key`). Wrong or missing
  key → `401` with no message or timing distinction (constant-time compare).
- **The identity is real.** A real `users` row (random email `agent-<uuid>@dev.invalid`,
  random username, random password that is **never returned** — the token is the only
  credential) and a real bearer token minted through the normal issuance path.
  `role=admin` mints a real admin via the same enforcement path as everything else
  (`RequireAuth → RequireAdmin` is untouched); every admin mint is logged at `WARN` with
  the request's source address. This endpoint is provisioning, not a bypass.
  **Durable side effects are the operator's to weigh:** because the minted admin is real,
  admin writes it performs (e.g. flipping `instance_settings.registration_mode`, granting
  entitlements) persist after the identity is reaped, with `updated_by`/`granted_by`
  nulled rather than attributed. Point automated admin-surface tests at throwaway stacks,
  not production instances.
- **TTL**: `ttl_seconds` default 1800 (30 min), hard cap 28800 (8 h), minimum 60. The token
  TTL is clamped to the identity TTL — a token cannot outlive its user. `expires_at` in the
  response is both.
- **Reaping**: `users.ephemeral_expires_at` (nullable; non-null = throwaway) marks the row;
  a reaper deletes expired ephemeral users at boot and on an interval, and the existing
  user-delete cascade removes their sessions and device bindings. A reaped identity's token
  stops working immediately. A crashed test run cannot leave an identity behind.
- **Response shape** is compatible with `POST /v1/auth/login`, plus `storage_keys` — the
  three web-SPA localStorage entries ready for injection by browser-automation tooling.

---

## First-run setup (amendment — first-run wizard, additive — signed off 2026-08-07)

> Two additive routes let a fresh instance be claimed through the UI instead of the
> `BOOTSTRAP_ADMIN_*` env dance (which is retained for unattended provisioning). Both are
> always registered; `claim` self-disables once an admin exists. Server-enforced, never
> UI-gated. Full design: `docs/design/plans/2026-08-07-first-run-wizard-spec.md`.

### `GET /v1/setup/status`
```json
// 200 — unauthenticated; only routing booleans, nothing an attacker can use
{ "admin_exists": false, "setup_completed": false }
```
Lets the SPA route a virgin instance to `/setup` rather than an unsatisfiable login screen.

### `POST /v1/setup/claim`
```json
// request — header X-Quasar-Setup-Token: <per-boot token>
// device_key is OPTIONAL and absent is accepted (unchanged behaviour)
{ "email": "...", "username": "...", "password": "<plaintext, TLS only>", "device_key": "<opaque>" }
// 201 — login-shaped; the caller is now authenticated as the new admin
{ "access_token": "...", "token_type": "Bearer", "expires_at": "...", "user": { "...": "role=admin" } }
```
Creates the **first** admin. Auth is the per-boot **setup token** (minted at boot when no admin
exists; written to the CP log at `WARN` and to `/run/quasar/setup-token`, 0600; not persisted
across boots). Gating, fail-closed: (1) `409 setup_already_complete` if any admin exists —
checked in the insert transaction under the **same advisory lock** as `BOOTSTRAP_ADMIN_*`, so
the two paths can never both create an admin; (2) wrong/missing token → `401`, constant-time,
no missing-vs-wrong distinction; (3) password obeys the `/v1/auth/register` strength rule; (4)
every attempt logged at `WARN` with source address. The token is never returned by any endpoint, and (amended 2026-08-07) is **never written to the log** either — only its file path is, because log aggregators routinely expose logs to principals who have no host access and any of them could otherwise claim a fresh instance. A failed token-file write fails the boot rather than degrading to log-only.

`device_key` (optional, additive) binds the founding admin's token to a device exactly as `POST /v1/auth/login` does, so that token is revocable from Account > Devices like any other; omitted, the token carries no `device_id` and behaves exactly as before.

### `POST /v1/setup/complete`
```json
// 200 — RequireAuth → RequireAdmin; idempotent
{ "admin_exists": true, "setup_completed": true }
```
Marks the wizard finished **or skipped** by setting `instance_settings.setup_completed_at`
when it is null; a second call is a no-op returning the same body. Completion is **instance
state**, not a per-browser marker: a skip must be permanent for every admin on every device,
which is exactly what `GET /v1/setup/status.setup_completed` advertises. The wizard's
furthest-step position deliberately stays client-side — it is per-operator convenience, not
instance truth — so `instance_settings.setup_state` remains reserved for a later cross-device
resume rather than being written here.

---

## App-image catalog + management (amendment — image management, additive — signed off 2026-08-07)

> The catalog mirrors the `accretion-io/quasar-images` **manifest** (Quasar-owned, versioned;
> Quasar refuses an unknown `manifest_version`). Installing an image also lands its **runtime
> preset** (the `runtime_presets` object, `schema.md`). This section documents the **P1 surface
> only** — read + sync; install/update/remove/pin land in later phases with their own routes.
> Full design: `docs/design/plans/2026-08-07-image-management-spec.md`. All routes
> `RequireAuth → RequireAdmin`.

### `GET /v1/admin/images`
Returns the cached catalog: each image's manifest metadata (`id`, `display_name`, `kind`,
`version`, `registry_ref`, `artwork`, `library_provider`), its per-instance install state
(`installed`, `installed_version`, `pinned`, `update_available`), and its per-host presence
(`hosts[]`: `state ∈ {absent,pulling,building,ready,failed}`, `version`, `error`, `bytes`).

### `POST /v1/admin/images/sync`
Re-fetches the manifest at `instance_settings.image_catalog_ref`, validates `manifest_version`,
upserts the cached catalog, and returns it. **A fetch failure never affects launches** — the
cached catalog keeps serving and the error is reported as `sync_error` in the envelope.
*(P3, additive)* Sync also resolves each entry's digest (§Digest pinning below) and applies
the instance's update policy (§Update-policy semantics below); `fetched_at` / `sync_error` are
now backed by `instance_settings` (§Sync-state persistence below) rather than an in-process
variable — the wire shape is unchanged.

## App-image management P3 — install / uninstall / pin / update (amendment, additive)

> **Amendment — additive, admin-gated route family per §Authorization's documented
> exception; delegated sign-off 2026-08-08 overnight campaign, flagged for review.** Adds
> five admin routes closing the P1 catalog surface's placeholder ("install/update/remove/
> pin land in later phases") plus the update-policy application and digest pinning (#440)
> described below. All five are `RequireAuth → RequireAdmin`, same as the P1 catalog
> routes. No existing shape changes: `GET /v1/admin/images`'s `pinned` field (declared in
> P1, always `false` until now) is real, and each catalog entry gains `lazy` and
> `registry_digest`. Full design: `docs/design/plans/2026-08-08-image-management-p3-spec.md`.

### `POST /v1/admin/images/{id}/install`
```json
// request — lazy optional, default false
{ "lazy": false }
// 201 — the now-installed catalog entry
{ "id": "steam", "display_name": "Steam", "kind": "prebuilt", "version": "1.4.0",
  "registry_ref": "ghcr.io/accretion-io/quasar-steam:1.4.0",
  "registry_digest": "ghcr.io/accretion-io/quasar-steam@sha256:<64hex>",
  "installed": true, "installed_version": "1.4.0", "pinned": false, "lazy": false,
  "update_available": false, "hosts": [] }
```
Seeds `installed_images` with the catalog's **current** `(version, registry_ref)` — captured
at adoption per the P2 amendment (`schema.md`), where the adopted `registry_ref` is the
**resolved digest form** (`name@sha256:...`, see §Digest pinning below), never the floating
tag. Unless `lazy:true`, the control plane immediately kicks ensure-everywhere
(`agent-api.md` image-management amendment) so every connected host starts pulling. A lazy
install seeds the adoption row and dispatches nothing — hosts pull on first launch
placement instead.
- **Errors:** `404 not_found` (no such catalog id); `409 already_installed` (an
  `installed_images` row already exists for this id — use `POST .../update` to move it to a
  newer version, not a re-install); `409 digest_unresolved` (the catalog's
  `registry_digest` is empty because the last sync could not resolve it — re-sync and
  retry).

### `DELETE /v1/admin/images/{id}/install`
```json
// 204 No Content
```
Uninstalls: sends `image_remove` (`agent-api.md`) to every connected host that reports the
image present (best-effort — the agent never force-removes an image backing a live
session), then deletes the `installed_images` row. `host_images` rows are **not**
FK-cascaded from `installed_images` (they FK `image_catalog`, not the adoption row); the
control plane deletes that image's `host_images` rows itself after dispatching the
removes. A late `image_state` report for an id no longer installed re-upserts
`host_images` if the id is still in the catalog — harmless, reconciled at the host's next
register.
- **Errors:** `404 not_installed` — no `installed_images` row for this id (this includes an
  id absent from the catalog entirely: either way there is nothing to uninstall).

### `POST /v1/admin/images/{id}/pin`
```json
// 204 No Content
```
Freezes the installed `(version, registry_ref)` against every update path, including the
`auto` policy and an explicit `.../update` call (both then answer `409 conflict`).
**Idempotent** — pinning an already-pinned image is a `204` no-op.
- **Errors:** `404 not_installed`.

### `DELETE /v1/admin/images/{id}/pin`
```json
// 204 No Content
```
Unpins. **Idempotent** — unpinning an already-unpinned image is a `204` no-op. Does **not**
itself trigger an update; the next `auto`-policy sync or an explicit `.../update` call does.
- **Errors:** `404 not_installed`.

### `POST /v1/admin/images/{id}/update`
```json
// 200 — applied distinguishes an actual re-adopt from a no-op; same 200 either way
{ "applied": true, "image": { "id": "steam", "installed_version": "1.4.0", "...": "..." } }
```
The `notify` policy's action and `manual`'s escape hatch (§Update-policy semantics below):
re-adopts the catalog's current `(version, registry_digest)` and re-ensures everywhere,
identically to a fresh install's dispatch. `applied:false` (still `200`) when
`installed_version` already equals the catalog `version` — a no-op, not an error, so a UI
button click never has to branch on status code to tell "worked" from "was already
current".
- **Errors:** `404 not_installed`; `409 conflict` when the image is pinned (unpin first).

### Update-policy semantics (`instance_settings.image_update_policy`)
*(DDL default is `notify` — migration 0054; an earlier draft of this section said `manual`.
The settings envelope carries the field from P3, optional for pre-P3 conformance.)*
`POST /v1/admin/images/sync` applies the instance-wide policy after refreshing the catalog:
- **`manual`** — sync only refreshes the catalog; `update_available` is
  recomputed per image; nothing installs or re-adopts on its own.
- **`notify`** — identical effect to `manual`. The distinction is UI-only today (the badge
  is surfaced more prominently); a notification channel is a documented future hook, not
  part of this contract. `POST .../update` is how an admin acts on it.
- **`auto`** — after the catalog refresh, every installed **and unpinned** image whose
  catalog `version` now differs from `installed_version` is re-adopted and re-ensured, the
  same as an explicit `.../update` on each. A **running session is never affected** — it
  keeps whatever image its host already pulled; the new image applies from that app's
  **next launch** (operator decision 2026-08-07). Pinned images are skipped regardless of
  policy.

Policy is read/written through the existing `GET` / `PATCH /v1/admin/settings`
(`image_update_policy: "manual"|"notify"|"auto"`, LP-SEC-01 surface) — no new settings
route.

### Digest pinning (#440)

At **sync**, the control plane resolves each manifest image's `registry_ref` **tag** to its
content digest via the registry's HTTP API (GHCR: an anonymous, token-less pull token for
public repos; `HEAD /v2/<name>/manifests/<tag>` with the Docker/OCI manifest `Accept`
headers → `Docker-Content-Digest`). The catalog stores **both**: `registry_ref` (the
human-readable tag, display only) and a new `registry_digest` (`name@sha256:<64hex>`, the
form used for every dispatch). **Install and update adopt the digest form** —
`installed_images`'s `registry_ref` column (P2, `schema.md`) holds `registry_digest` at the
moment of adoption, not the mutable tag, closing the standing critical (#440) that adopting
straight from the live catalog could otherwise silently split the fleet across builds
sharing one version label.

Resolution failure at sync **never fails the sync**: `registry_digest` is left empty and the
per-image note is carried the same way `sync_error` already is (below); installing an
unresolved image is refused (`409 digest_unresolved`) until a later sync resolves it. No CI
change is required — `quasar-manifest.json` keeps publishing the human-readable tag; digest
resolution is a control-plane-side, sync-time lookup, not a build-time one.

**SSRF containment.** Because the registry host derives from remote catalog data (and a
registry's `WWW-Authenticate` realm from the registry's own response), the resolver is
constrained: HTTPS only; the registry host and any token realm must match an allowlist
(`QUASAR_IMAGE_REGISTRY_HOSTS`, default `ghcr.io`); HTTP redirects are not followed; and the
dialer refuses connections to loopback/private/link-local/multicast/unspecified addresses
(DNS-rebind guard). A ref or realm failing these checks resolves to an empty digest (the
image is then un-installable until the catalog names an allowed registry), never an outbound
request to an internal address. **Private/credentialed registries are out of scope for P3**
(anonymous pull tokens only) — tracked separately; a private image simply stays
`digest_unresolved`.

### Sync-state persistence

`GET /v1/admin/images`'s `fetched_at` / `sync_error` fields (P1) are now backed by
`instance_settings.image_synced_at` / `image_sync_error` (`schema.md`, migration `0056`)
instead of an in-process variable — the wire shape is **unchanged**, only the storage moved,
so a control-plane restart no longer forgets the last sync's outcome. Closes the P1
"in-memory sync-state" note (invariant #5 cleanup deferred from P1/P2).

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

## Remote access, TLS certificate, and signaling origins (first-run wizard v2 §S6, 2026-08-09)

> **Amendment — additive; Opus + operator sign-off granted 2026-08-09 for exactly this
> surface.** Adds `GET /v1/tls/certificate.pem` (public), `GET /v1/admin/access-check`, and an
> `allowed_origins` field on the existing `GET`/`PATCH /v1/admin/settings`. **No existing shape
> changes.** The fourth route in the original amendment,
> `POST /v1/admin/tls/certificate`, was withdrawn before shipping — see below.

### The failure chain this closes

An operator reaches the instance by a name the certificate does not cover → the browser
refuses or warns → the page is **not a secure context** → the **microphone is unavailable**,
with a message that correctly says "needs HTTPS" but cannot say *why theirs is not trusted*.
Separately, the signaling origin allow-list returns a `403` that is **structurally invisible
to browser JS**, so a `Host`-rewriting reverse proxy reaches the user as "signaling
connection failed" with no cause. Nothing in the product explained either.

### Three supported TLS topologies — none of them mandatory

| topology | TLS terminated by | operator does | the panel must **not** say |
|---|---|---|---|
| **A** self-signed (default, LAN) | Quasar's own listener | download the cert, trust it, **verify the fingerprint** | — |
| **B** own certificate | Quasar's own listener | mount cert + key at `QUASAR_TLS_CERT`/`QUASAR_TLS_KEY` | — |
| **C** external reverse proxy (e.g. their own Caddy + Let's Encrypt) | the proxy | set allowed origins; Quasar keeps its self-signed cert internally and **that is correct** | must **not** report the cert as a problem — its SANs are irrelevant when the browser never sees it |

**Normative:** A, B and C all complete setup, and a client MUST NOT present any of them as a
required step.

**Topology detection.** When a request arrives carrying `X-Forwarded-Proto` or RFC 7239
`Forwarded`, a proxy is in front of the control plane, so its certificate is an internal
detail: `certificate.in_use` is `false` with a reason saying the setup is supported and
complete. **Forwarded headers may only SOFTEN advice — they authorise nothing.** They are
trivially spoofable; the worst a hostile client achieves is making an admin-gated diagnostics
page tell *them* that their own certificate is not in use.

### `GET /v1/tls/certificate.pem` — public

Returns the **leaf certificate currently being served**, PEM, as an attachment with
`Cache-Control: no-store` and an `X-Quasar-Certificate-Fingerprint` header.

**Unauthenticated, and that is correct rather than lax:** a client that does not yet trust the
certificate frequently cannot complete a login *in order to* fetch it. It discloses SANs
(internal hostnames, LAN IPs) to anyone who can reach the port — **so does the TLS handshake**.
Equivalent exposure, not new exposure.

**Never the private key.** The server re-encodes PEM from the **parsed leaf's DER** with a
compile-time `CERTIFICATE` block type: it does not read the key path and no key bytes exist in
scope on this path. This is covered by an explicit test asserting no `PRIVATE KEY` block can
appear in the response — **not by a comment**.

**The honest caveat.** Downloading a certificate over the connection you do not yet trust is
trust-on-first-use; a MITM can serve their own. The mitigation is the fingerprint, and **it
only works out of band** — the control plane logs the SHA-256 at startup. A client **MUST**
tell the operator to compare the two. Without that instruction the download button is security
theatre; with it, it is a real verification step.

### `GET /v1/admin/access-check` — admin

Reports, **for the request that called it**: the `Host` and `Origin` seen, whether that host is
covered by the served certificate's SANs, the fingerprint and days-to-expiry, whether the
origin would pass the `/v1/signal` allow-list, and whether that allow-list is configured at
all. Shape: `AccessCheck` in `openapi.yaml`.

Admin-gated by the existing `RequireAuth → RequireAdmin`. It reflects `Host` and `Origin`, so
both are **length-capped at 256 characters** and are only ever rendered as JSON string values —
a client **MUST NOT** put them through `dangerouslySetInnerHTML`. It discloses configuration an
admin can already read.

`secure_context` is the field that links this panel to the microphone: it is the precondition for
`getUserMedia`. **A forwarded protocol, when present, is authoritative for the browser-facing hop
and overrides this control plane's own transport** — a proxy speaking HTTPS to us while serving
the browser in the clear is not a secure context, however encrypted our own hop was. The loopback
exception is taken from the browser's `Origin`, never from a `Host` a proxy may have rewritten to
a loopback backend authority.

### `POST /v1/admin/tls/certificate` — **specified, deliberately not shipped**

§S6d specified an admin upload for the operator's own certificate. **It is not on this
contract**, and no server implements it. It is the only surface that would accept a private
key, and protecting that key requires knowing whether the browser's hop was encrypted — a
property that cannot be reconstructed from request headers on a router shared with the
plaintext listener. Five review rounds produced five distinct holes in successive attempts,
each fix correct and each followed by a new gap in the same inference. The route returns when
there is a listener-level answer (a dedicated management listener, mTLS, or similar), not a
sixth iteration of the gate. Implementation and full review history live on
`feat/tls-certificate-upload`.

**Topology B is meanwhile served by mounting the certificate** at `QUASAR_TLS_CERT` /
`QUASAR_TLS_KEY`, which is unchanged and fully supported.

### Allowed origins — on `/v1/admin/settings`, not a route of its own

The allow-list is an **instance-wide singleton setting**, exactly like `registration_mode`,
`mic_capture_enabled` and `storage_provider`. It rides the existing settings envelope and PATCH
body as `allowed_origins` rather than earning its own endpoint: a separate route would fork the
admin settings surface, duplicate its auth and audit wiring, and force a client to make two
calls to render one form. Nothing about the value's semantics differs from its neighbours.

- **Absent = unchanged; an explicitly-sent `[]` clears the list.** Those are different requests
  and the server distinguishes them.
- Entries are validated as **scheme + host only** (http/https, no path, query, credentials or
  trailing slash), and the **normalized** form is stored — so what is saved is exactly what
  `/v1/signal` compares against.
- **`*` is rejected outright** (`400`). A wildcard is indistinguishable from having no
  allow-list at all, and would discard the defense-in-depth layer entirely.
- **`QUASAR_ALLOWED_ORIGINS` remains an OVERRIDE.** When the variable is **set** — including to
  the empty string, which is how a hardened deployment pins the list off — it wins and the
  column is not consulted. This is what makes the change a behavioural **no-op on upgrade** for
  every existing deployment. `access-check` reports which source won so a UI can grey out an
  editor the environment has pinned.
- **An empty list is not "deny all".** `/v1/signal` still admits a same-origin request and a
  request carrying no `Origin` header at all.

**Why "always seed the IP as an allowed origin" is deliberately NOT implemented.** The
same-origin check already ends with `Origin.Host == Host`, so browsing to `https://<ip>:8443`
passes with **no configuration**. Seeding it would be a no-op that looks load-bearing. The case
that genuinely breaks is a reverse proxy that rewrites `Host`: the browser sends the public
origin, the proxy presents an internal `Host`, the exemption misses, and the operator meets the
invisible 403. `same_origin_exemption` on `AccessCheck` is what makes that visible *before* it
bites.

**Why admin-editability is safe.** `/v1/signal` authenticates with a single-use `token` **query
parameter**, not a cookie. A malicious page cannot obtain that token, so it cannot open a
signaling socket whatever origin it claims. The allow-list is defense-in-depth, **not the
primary CSRF control**.

---

## Library

### `GET /v1/apps`
Lists enabled apps (the library the user can launch). **`RequireAuth`** — `401` without a valid
bearer. *(UI-P1, breaking: this list was unauthenticated until 2026-07-27; see the amendment
block at the top of this document for what was exposed and why it changed.)*
**(Steam library discovery Phase 2, non-additive: the list is now `enabled = true` **AND**
entitled — see "Entitlement scope" below.)**
```json
// 200
{ "items": [
    { "id": "<uuid>", "name": "Foo", "description": "...", "cover_url": "https://...",
      "kind": "game", "external_source": "steam", "external_id": "570", "favourite": true,
      "default_width": 1920, "default_height": 1080, "default_fps": 60, "default_bitrate_kbps": 15000,
      "default_profile_id": "1440p60", "profile_policy": "prefer",
      "display_stream": { "width": 2560, "height": 1440, "fps": 60, "bitrate_kbps": 20000 } }
  ], "next_cursor": null }
```
`display_stream` is the user-facing stream advertised in the library: it resolves the
app/global profile policy when available, and otherwise falls back to the legacy
`default_*` stream fields. `runtime_spec` and resource defaults are **not** exposed to
clients (agent-internal / scheduler-internal). Disabled apps are omitted.

**Entitlement scope *(Steam library discovery Phase 2 — this is the non-additive part)*.** An app
appears here only if it is `enabled` **and** the caller holds an entitlement for it: either an
`all` row (the app is visible to everyone) **or** a `('user', caller)` row. On day one every app
carries an `all` row from the 0043 backfill, so the list is byte-identical to the pre-Phase-2 one;
it narrows only as an admin deliberately narrows it.

- **The filter applies to every role, admins included, and there is deliberately no bypass.** An
  admin browsing the library is a user browsing *their own* library — a fleet-wide god view here
  would be actively wrong once library discovery is on, since it would show every other user's
  games. **The god view is `GET /v1/admin/apps`**, which is unfiltered and unchanged, so nothing
  becomes unreachable; it moves to the route that already means "the whole catalogue". An admin
  who wants an app in their *own* library grants themselves the entitlement — one call, and it
  leaves an audit row, which `if isAdmin { skip }` does not.
- **A caller holding both an `all` and a personal entitlement for one app sees it exactly once.**
  The two are independent facts and the schema deliberately permits both to exist (revoking one
  must not revoke the other), so this endpoint matches on **existence**, never by joining the
  entitlement rows in. A join would emit the app twice, and because this list pages by offset with
  a `limit + 1` overfetch, a duplicated row does not merely look untidy — it consumes a slot,
  shifts the page boundary, and silently drops an app off the far end. It is stated here because
  it is the kind of defect every single-user test passes.
- **This is UX, not the authorization boundary.** The boundary is `POST /v1/sessions`. A client
  that ignores this list and launches an app id directly is refused there, not here.
- **A library-ordering consequence, stated because it is visible.** `apps` has no `sort_order` and
  this list is `ORDER BY created_at DESC`, so newly entitled apps arrive at the top of an entitled
  user's library all at once. Phase 2 alone cannot produce a batch (an admin grants one at a
  time); Phase 4's discovery can. No ordering column is added here — that is a library-UX change
  with its own design.

*(UI-P4)* `default_profile_id` now names a **launch profile**, and `profile_policy` is
`inherit | prefer | force` — **`custom` is gone** (see the UI-P4 amendment block). `display_stream`
resolves through the launch profile's **top rung** (`position = 1`), which is the advertised
setting; a launch may fall through to a lower rung and stream at a different resolution, and the
session's own `stream` block is the truth for that. The **`default_*` columns stay**: they remain
the COALESCE fallback here when no launch profile resolves (an `inherit` app while the global
default is unset), they are what `LaunchConsoleSession` uses unconditionally, and they are the
ceiling on the documented `POST /v1/sessions` stream-override escape hatch, which is reachable for
**any** app and not only a formerly-`custom` one.

*(UI-P1; widened by Steam library discovery Phase 3)* `kind`
(`"game" | "desktop" | "launcher"`) is the library classification, **presentation only** —
it exists so the client can split and filter the library. Nothing in scheduling, admission,
profile/codec resolution, or the agent wire reads it. It is defined on `AppListItem`, so the
single-app read (`App`) and the admin read (`AdminApp`) inherit it — they are `allOf`
compositions over the same shape. `"launcher"` (Phase 3, migration 0044) is the category the
Steam **provider app** is filed under once it has derived children; it does **not** mean
"launchable" and it is **not** the discovery trigger — that is `AdminApp.library_provider`, and
nothing server-side branches on `kind`. A pre-Phase-3 client holding a closed two-value union
receives `"launcher"` harmlessly: the tile shows under "All" and under no segment.

*(Steam library discovery, Phase 3)* `parent_app_id` (uuid or `null`, **always serialized**) is
the app this tile is **derived** from — `null` for a normal app, which is every app predating
migration 0044. It is on the public read shape, unlike `origin` and `library_provider`, for one
concrete reason: the single-writer lock belongs to the **parent's** home, so while any tile in a
family is live every sibling answers `409 home_in_use`. The client needs this field to mark those
siblings blocked rather than let the user find out by clicking. **That is presentation; the
enforcement is the server's `409`** (§Derived tiles).

*(Steam library discovery, Phase 1)* `external_source` (`"" | "steam"`) and `external_id` say
**which provider title this app is** — "this app *is* Steam appid 570". Both are **always
serialized**, and **`""` on both is the meaningful default**, not a missing value: it means "this
app is not a provider title", which is the state of every app that predates migration 0042. They
sit on `AppListItem`, so `App` and `AdminApp` inherit them the same way they inherit `kind`; they
are **identity**, not operator configuration, which is why they are not admin-only like
`runtime_preset_id`. **Phase 1's only reader is artwork resolution** (§Cover artwork): an app
carrying an appid resolves its two crops by that id and never enters the fuzzy title matcher.
Nothing in scheduling, admission, profile/codec resolution, or the agent wire reads them.

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
Single app, same fields as a list item — including `kind`, `external_source`/`external_id`, and
the caller-resolved `favourite`. `404` if absent or disabled.

**Entitlement scope *(Steam library discovery Phase 2)*: for a non-admin caller this is `404`
when the caller is not entitled**, folded into the existing no-such-app answer rather than given
its own status. This endpoint is already an existence check for a non-admin — a disabled app is
`404` too — so collapsing "not entitled" into it leaks nothing and is the correct posture for a
per-user library: the caller cannot tell "no such app" from "not yours". `403` is reserved for the
two *writes* that follow (`PUT /v1/me/favourites/{app_id}`, `POST /v1/sessions`), where the caller
has deliberately named a specific app and needs an actionable answer.

> **The admin branch of this route is NOT entitlement-filtered, and that is deliberate. Do not
> "fix" it.** This route has always had two behaviours: for a non-admin it is the library read
> above; **for an admin it returns the full admin shape** — disabled apps included, `runtime_spec`
> included. **It is the admin app-editor's loader** (`web/src/api/admin.ts:160`), and **there is no
> `GET /v1/admin/apps/{id}` route** to move that load onto. Filtering it would make an app
> **un-editable the instant an admin restricted it** — an operator lockout with no recovery inside
> the product, reachable by using the feature exactly as intended.
>
> **The asymmetry, stated plainly so a future reader does not read it as a bug:** an admin can read
> one app's detail here while that same app is **absent from their own `GET /v1/apps`**. That is
> consistent, not contradictory — §Authorization's admin surface is "the whole fleet's
> configuration", the library is "what *I* may launch", and this route serves both. Nothing an
> admin can read here is anything `GET /v1/admin/apps` would not already hand them, so the
> asymmetry concedes no information. **What it does not do is grant a launch:** `POST /v1/sessions`
> and the swap are entitlement-checked for every role, so an admin who can *read* an app they have
> restricted still cannot *run* it without granting themselves the entitlement.

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
value outside `('game','desktop','launcher')` *(Steam library discovery Phase 3 widened the enum;
`'launcher'` is accepted on create and patch and round-trips on every read shape)*. The DB `CHECK`
is the backstop, never the primary gate.

> **Rollout order: control plane before client.** `crud.decodeJSON` sets
> `DisallowUnknownFields()` (`control-plane/internal/crud/handler.go:540`), so a client that
> sends `kind` to a control plane **without** this amendment gets a hard `400 validation_failed`
> — not a silent ignore. Deploy the control plane first; a new client against an old control
> plane fails loudly on every app create/edit.

**The app write shape and `external_source` / `external_id` (Steam library discovery, Phase 1).**
Both are **optional** on create and patch, both admin-gated like every other `AppWrite` field, and
both carry the same presence rule as `kind`: **absent = the schema default (`""`) on create,
unchanged on patch.** An absent field must never be written as a zero value — that is the
`cb97bfb` trap again, and on this pair it has a specific cost: an unrelated `PATCH` that says
nothing about them would silently **un-tag** an app's Steam appid and send its artwork back to the
fuzzy title matcher on the next sweep.

- **An explicit `""` IS valid here, and this is the one place the rule differs from `kind`.**
  `""` is the real domain value for "this app is not a provider title", so `{"external_id": ""}`
  is a deliberate **clear** — a real operation an admin needs — and not `400`. For `kind`, by
  contrast, `""` is outside the enum and is a hard `400`. Absent and `""` therefore mean different
  things and must stay distinguishable on the wire, which is why they decode through a
  presence-aware pointer.
- **`external_source` must be `""` or `"steam"`** (`400 validation_failed` otherwise).
- **`external_id` must be `""` or `^[1-9][0-9]{0,9}$`** — a bare positive integer, no leading zero,
  no sign, no whitespace, no separators. `"0"`, `"007"`, `"1 2"`, `"1;rm -rf /"` and
  `"-applaunch 480 -foo"` are all `400 validation_failed`.
- **Why the id is regex-constrained: argument-injection containment** (spec §10), not a format
  preference. The stored appid is eventually rendered into `STEAM_STARTUP_FLAGS`, whose value the
  `quasar-steam` entrypoint **word-splits** with `read -r -a`. A stored `"480 -foo"` would
  therefore arrive at the Steam client as *two extra arguments*. The flags are built from a fixed
  template with the validated integer interpolated, never by concatenating stored free text.
- **The handler is the primary gate; the DB `CHECK` is the backstop** — `apps_external_source_ck`
  and `apps_external_id_ck` carry the same value set and the same regex (`schema.md`, migration
  0042). Same division of labour as `kind`, with one addition worth stating: because the value is
  written by an automated job and read by a shell-adjacent consumer, the `CHECK` is also the only
  one of the two layers that survives an admin editing the column directly later.
- **The two fields are validated independently.** There is deliberately **no pairing rule** — no
  "a source requires an id", and no reverse. None is needed: an admin who sets one field in one
  request and the other in the next would be rejected by a rule nothing asked for, and a half-set
  pair is **inert**, because the only reader (artwork resolution) requires *both* before it takes
  the by-id path. That is a decision, not an oversight.

> **Rollout order: control plane before client**, for the same reason as `kind` — `crud.decodeJSON`
> sets `DisallowUnknownFields()`, so a client that sends either field to a control plane **without**
> this amendment gets a hard `400 validation_failed`, not a silent ignore.

**The app write shape and `entitle` (Steam library discovery, Phase 2).** `POST /v1/apps` gains one
**optional** request field, `entitle`: `"all" | "none"`, **default `"all"`**.

- **`"all"` (the default, and what an absent field means)** creates the app with an
  `('all', granted_by='admin')` entitlement, so it is immediately visible to everyone — i.e.
  **exactly the pre-entitlements behaviour of creating an app**. `"none"` creates it entitled to
  nobody, for an admin who wants to configure access before anyone sees it.
- **The default is the whole point and is not negotiable.** Once `GET /v1/apps` is
  entitlement-filtered, a newly created app is invisible until something entitles it — so without
  a default grant, *"I made an app and nobody can see it"* becomes the **default** experience. It
  is the same failure as an un-backfilled migration (`schema.md`), one app at a time.
- **CREATE-ONLY.** `entitle` is not accepted on `PATCH /v1/apps/{id}`; sending it there is
  `400 validation_failed` (`crud.decodeJSON` sets `DisallowUnknownFields()`). It describes how an
  app is *born*, not a property it carries — after creation, access is edited through the
  entitlement routes above, which is the surface that produces an audit row and can express a
  per-user grant. A patchable `entitle` would be a second, lossy way to say the same thing.
- **`entitle` is not a stored column and is never returned on any read shape.** It is an
  instruction to the create path; the resulting entitlement row is what persists, and
  `GET /v1/admin/apps/{id}/entitlements` is where it is visible.
- Any value other than `"all"`, `"none"` or absent is `400 validation_failed` — **rejected rather
  than treated as "none"**, because a typo (`"nome"`, `"None"`) that quietly created an invisible
  app would be diagnosed as "the catalogue is broken".
- **The default grant is part of the create, and a failure to write it removes the app.** An app
  that exists with no entitlement is not a security problem but an undiagnosable one — the admin
  sees a created app, every user sees nothing, and no field in the editor explains it. The create
  therefore fails whole (`500`, no app), rather than succeeding into that state.

> **Rollout order: control plane before client**, same reason again — a client sending `entitle`
> to a control plane without this amendment gets a hard `400`, not a silent ignore. Note the
> asymmetry with the *read* path: a client that never sends `entitle` is unaffected by this field,
> but **every** client is affected by the filtered `GET /v1/apps`, which is why this phase's
> client re-pin is scheduled with it.

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
- **`403 forbidden` when the caller is not entitled to the app** *(Steam library discovery Phase
  2)*, checked **after** the `404` above rather than folded into it. This is the one read-adjacent
  place the contract gives a distinguishable answer instead of a uniform `404`, and it is a
  considered trade: `GET /v1/apps/{id}` is a *read*, so collapsing "not entitled" into "not found"
  costs the caller nothing, but this is a **write** the client offers as a toggle on a tile — an
  app the caller can no longer launch should say so rather than appear to have evaporated. The
  existence signal it concedes is bounded, because the caller must already hold a valid app UUID
  to reach it. **Applied to every role, admin included**, unlike the visibility rule above:
  `/v1/me/*` is the user surface by definition, an admin favouriting an app that is not in their
  own library is meaningless, and a role arm here would be the first crack in the no-bypass rule.
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

### Entitlements — who may see and launch an app *(Steam library discovery Phase 2, admin)*

An **entitlement** is one fact: *"this subject may see and launch this app"*. It is the object the
roadmap's library-provider model already owns, deliberately built here rather than a narrower
Steam-shaped shortcut (a `visible_to_user_id` column on the app could express neither "everyone"
nor an admin grant nor a future group, and would have to be torn out when a second provider
arrives). **Presence of the row is the fact** — no `revoked` boolean, no soft delete.

`RequireAuth → RequireAdmin`, like every other admin route. **These four are the only way to widen
visibility**, which is exactly what lets `GET /v1/apps` and `POST /v1/sessions` stay filtered for
admins too: an admin who cannot see something grants it here, and the grant is written to
`GET /v1/admin/activity` (`app.entitlement.grant` / `app.entitlement.revoke`).

**Two subject shapes, and only two in Phase 2.** `subject_type: "all"` (with **no** `subject_id`)
means everyone; `subject_type: "user"` (with a user UUID) is a personal grant. `"group"` is
additive later — a new value and a third uniqueness key, no shape change — and is deliberately not
shipped now.

**The entitlement object** (identical in all three read shapes):
```json
{ "id": "<uuid>",
  "subject_type": "user",          // "user" | "all"
  "subject_id": "<uuid>",          // null when subject_type is "all"
  "subject_username": "alice",     // null when subject_type is "all"; joined for display
  "app_id": "<uuid>", "app_name": "Foo",
  "granted_by": "admin",           // "admin" | "provider" | "migration"
  "granted_by_user": "<uuid>",     // the acting admin, null otherwise
  "source_ref": "",
  "created_at": "..." }
```
`granted_by` is **provenance, not authority** — every one of the three grants the same access.
`"migration"` is the 0043 backfill and only ever that, so *"who made this app public"* answers
"it was public before entitlements existed" rather than falsely attributing it to whichever admin
happened to act first. `"provider"` is written by library discovery in **Phase 4** — nothing
writes it yet; it is in the contract now so revoking one is already a working path the day the
first one appears. `source_ref` is free-form provenance for a provider grant and is `""` for
everything Phase 2 writes.

**`GET /v1/admin/apps/{id}/entitlements` — who can see this app:**
```json
// 200
{ "items": [ { "subject_type": "all", "subject_id": null, ... },
             { "subject_type": "user", "subject_username": "alice", ... } ] }
```
Ordered **`all` first, then by username** — the "Visible to" control reads top-down and "everyone"
is the fact that subsumes the rest. `404 not_found` if the app does not exist; `400
validation_failed` on a malformed UUID.

**`POST /v1/admin/apps/{id}/entitlements` — grant:**
```json
// request
{ "subject_type": "user", "subject_id": "<uuid>" }
// 201
{ "entitlement": { ...the object above... } }
```
- `subject_type: "all"` must **omit** `subject_id` (or send it empty); anything else is
  `400 validation_failed`. `subject_type: "user"` **requires** a user UUID. Any other
  `subject_type` is `400`.
- **`409 conflict` when that subject already holds an entitlement for this app.** Not a silent
  idempotent `201`: the two are genuinely different outcomes for an admin who believes they are
  granting access to someone new, and the underlying uniqueness is per **shape**, so the honest
  answer to "grant again" is "there is already one".
- `404 not_found` if the app or the named user does not exist.
- The grant is always recorded as `granted_by: "admin"` with the acting admin in
  `granted_by_user`. A client cannot assert either.

**`DELETE /v1/admin/apps/{id}/entitlements/{entitlement_id}` — revoke:**
```json
// 204 No Content
```
- **Scoped to the app in the path**, not a bare delete by id: the entitlement must belong to
  `{id}`, and a well-formed pair that does not match is `404 not_found`. So a stale admin page
  cannot revoke an entitlement on some *other* app by replaying an id it still holds, and the URL
  means what it reads as.
- `400 validation_failed` if either id is malformed.
- **Revoking an `all` row does not revoke anybody's personal grant, and vice versa.** They are
  independent facts about the same app and the schema deliberately allows both to exist.
- **A `granted_by: "provider"` revoke is not permanent** and a client showing this surface should
  say so: the next library sync re-grants it if the title is still installed. Permanent fleet-wide
  suppression is the ignore-rule path (Phase 4), not per-user revocation — otherwise an admin
  revokes the same junk tile once per user, forever.

**`GET /v1/admin/users/{id}/entitlements` — the per-user direction:**
```json
// 200
{ "items": [ { "subject_type": "user", "app_name": "Foo", ... } ] }
```
Ordered by app name. `404 not_found` if the user does not exist.

> **This returns the user's PERSONAL grants only — deliberately not the `all` rows they also
> benefit from.** It is the one place in this surface where the obvious implementation is the
> wrong one, so the reasoning is recorded rather than left to be rediscovered. Including the `all`
> rows would make this screen **a per-user copy of the entire catalogue**, on which every Revoke
> button has a **fleet-wide** effect — an admin looking at *alice's* page, clicking Revoke on a
> row shown as alice's, and removing the app from **everyone**. The narrower answer is also the
> one that matches the question this route exists to answer: *"what has been granted to this
> person specifically?"* The fleet-wide view of an app's access is the app-direction route above,
> where an `all` row is unmistakably an app-level fact. A future reader who wants "everything this
> user can see" should note that it is already answerable — it is `GET /v1/apps` as that user —
> and that widening *this* route to produce it is the change being warned against.

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
  migration `0014`). *(Phase 2: its **entitlements** cascade away too — `entitlements.app_id` is
  `ON DELETE CASCADE`. An entitlement to an app that no longer exists has no meaning, and leaving
  one behind would silently re-grant access if the id were ever reused.)*
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
      "network": "bridge",
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

**`network`** *(first-run-experience §S2, additive)* is one of the **closed** set
`"" | "none" | "bridge"`; any other value is `400 validation_failed`. `"host"` is refused the same
way, with the `400` naming `QUASAR_CONTAINER_NETWORK` and why: `--network host` removes the
container's network namespace rather than widening it, exposing the host's own loopback (control
plane, Postgres, the docker proxy, any admin-only port) to the app — and because a preset is
portable (it can be materialized from a catalog image manifest authored on another machine),
accepting `host` here would let a manifest dissolve the isolation boundary on every host that
installs it. Host networking stays reachable only through the agent's own host-local
`QUASAR_CONTAINER_NETWORK` operator knob, never through this API. On create, absent falls through
to the server default (`""`, inherit) like every other field. On patch, absent means unchanged and
an explicit `""` is a real, meaningful write — it clears an app-owned override back to inherit,
distinct from omitting the key.

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
| `network` | *(first-run-experience §S2, additive)* same rule as `image`: the app's own `runtime_spec.network` overrides when set; blank/absent inherits the preset's. **When neither states one, the key is omitted entirely** from the flattened `app` object sent to the agent — not sent as `""` — so the agent's own host fallback chain (`QUASAR_CONTAINER_NETWORK`, else `none`) applies exactly as it does for an app with no preset at all (`agent-api.md` `session_assign.app.network`). (none|bridge only — see POST above) |

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

### `GET` / `PATCH /v1/me/ui-preferences`

Per-user client presentation state. The server stores and validates it; it never
acts on it. Nothing in the session pipeline, scheduler, or encode path reads this
table — a corrupt value can only produce a differently-drawn overlay.

`PATCH` is a **partial merge, one level deep**: a body of
`{"session_overlay": {"strip_position": "top"}}` changes only that field and
leaves `strip_preset`, `strip_items` and `strip_auto_hide` alone. Top-level keys
the server does not recognise are **preserved verbatim**, so an older control
plane cannot silently delete a newer client's preferences.

Invalid enum values are a `400 validation_failed`, not a silent clamp: a client
sending `strip_position: "left"` has a bug, and clamping it to `bottom` would
hide that bug on every device the user owns.

`strip_items` mixes two kinds of thing. `signal`, `identity`, `codec`, `metrics`
and `hint` are **readouts** — turning one off removes information. `capture`,
`exit`, `mic` and `fullscreen` are **actions**: they put the controls that were
previously drawer-only onto the always-on strip, so none of them requires
summoning the drawer first. The server treats all nine identically (a validated
boolean it never reads), but a client must not: an action drawn while input is
captured is unreachable, because the pointer is locked to the game. Draw them
only while input is free.

An action item asks the client to **draw** a control; it never grants the
capability behind it. `mic` is the case where the distinction bites: whether a
session may capture microphone audio is `session.mic_granted`, a server
decision, and the two are independent. `mic=true` on a session with
`mic_granted=false` means "the user wants this control on their strip" and must
render as unavailable — not hidden (the user asked for it) and not enabled (the
server said no).

---

### `GET /v1/me/highlights` *(home-rail amendment, 2026-08-05)*

The featured rail on the logged-in landing page. Additive: a new path, no
existing shape changed.

**The server owns the ranking.** That was the explicit decision at sign-off, and
it is the reason this endpoint exists rather than a pair of fields on
`AppListItem`. Order and the per-item `reason` are produced here; a client
renders what it is handed. A client must **not** re-derive the rail by sorting
`/v1/apps` against `/v1/sessions` — two rankers drifting apart is the failure
this design forecloses.

**Why not fields on `AppListItem`.** `AppListItem` is `allOf`-inherited by both
`App` and `AdminApp`, so a field there lands on three read shapes at once,
including admin ones with no use for it. `GET /v1/apps` is also paginated, and
"most played" cannot be *ordered by* a value computed after `LIMIT/OFFSET` has
already been applied — ranking the catalogue by play time would push a
per-row aggregate over `sessions` into the sort key of the hottest endpoint in
the product, to render five cards.

**Entitlement filtering is a correctness requirement, not politeness.** A
session row outlives the entitlement that authorised it: revoking a user's
entitlement does not delete their history. This endpoint therefore applies the
same entitled + enabled predicate as `GET /v1/apps`. Omit it and a revoked title
reappears on the user's home page with its play time attached. The launch itself
is still refused at `POST /v1/sessions` — this endpoint is UX, not the
authorization boundary — but surfacing a revoked app is an information leak on
its own terms.

**Best-effort by contract.** `items` may be shorter than five and is empty for a
user who has never launched anything. An empty rail is a normal state a client
must render, never an error.

`play_seconds` is clamped per session, so an unreconciled `NULL ended_at` is
bounded rather than open-ended, and `state='failed'` sessions are excluded — a
launch that never ran is not play time.

Deliberately **absent** from `Highlight`: the app's name, artwork, kind and
favourite flag (the client already holds the full `AppListItem` and joins on
`app_id` — a second copy would drift the moment artwork is re-resolved), and any
host name (a user-facing `Session` carries `host_id` only, and `GET /v1/hosts`
is admin-gated).

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
  - **`403 forbidden`** *(Steam library discovery Phase 2 — a **new terminal status** on this
    endpoint, which is why that amendment is not additive)* — the caller holds neither an `all`
    nor a personal entitlement for `app_id`. **This is the authorization boundary of the whole
    entitlements feature**; the filtered `GET /v1/apps` is UX, and a client that ignores it and
    posts an app id directly is refused here.
    - **`403`, not `404`.** `GET /v1/apps/{id}` answers `404` for a non-entitled app precisely
      because a *read* can afford to say nothing. A launch cannot: the caller named this app
      deliberately, and "no such app" would send them to report a broken catalogue instead of
      asking an admin for access.
    - **`403`, not `409`.** The three `409`s around it (`profile_ineligible`,
      `profile_not_launchable_for_app`, quota) all mean "valid request, wrong right now". This one
      is a statement about the **caller**, which is what `403` means in this contract.
    - **Checked inside the scheduling transaction, before the quota check and before placement.**
      Authorization precedes resource accounting — a caller who is both unentitled and at their
      session limit must be told "you may not launch this", not "you have too many sessions" —
      and nothing is reserved for a launch that is going to be refused.
    - **For every role, admin included; there is no bypass.** An admin who wants to launch an app
      they have restricted grants themselves the entitlement (`POST
      /v1/admin/apps/{id}/entitlements`), which is one call and leaves an audit row.
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

#### `mic` — microphone capture request (2026-08-02)
> *Additive amendment — one optional top-level boolean request field on `POST /v1/sessions`, one
> boolean on every session `stream` body, one instance setting. Changes no existing shape or
> status code (absent ⇒ today's behavior exactly).*

`POST /v1/sessions` accepts an optional `"mic": true | false` field (absent ⇒ `false`). When
`true` **and** the instance setting `mic_capture_enabled` is on, the control plane grants
microphone capture for the session: `session_assign.stream.mic = true` is dispatched to the
agent (`agent-api.md`), and the host's `pc:"audio"` offer carries a client→host microphone
m-line (`signaling.md` §Microphone m-line). When the instance setting is off (or the field is
absent/`false`), the launch **succeeds normally** with no microphone — a mic request against a
mic-disabled instance is not an error, it is silently not granted, and the response reflects
what was granted.

Every session body's `stream` object gains **`mic`: bool** — the granted state, not the
requested state. Enforcement is server-side at launch; there is no per-app or per-user
granularity in v1.

`mic_capture_enabled` (boolean, default `false`) joins `instance_settings` and appears on
`GET /v1/admin/settings` / `PATCH /v1/admin/settings` (optional-pointer decode rule, same as
every other settings field). Admin-gated server-side via the standard `RequireAuth →
RequireAdmin` chain. Flipping it affects only **subsequent** launches; live sessions are
untouched. Microphone *content* is never recorded, relayed to any endpoint other than the
session's own app container, or logged; admin surfaces see only the granted/negotiated state
(`session.effective_media`).

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
(device decode), 4 (failure history), 5 (hardware encoder) and 6 (encoder throughput). This is
exactly the forced re-test path by which a
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
  `decode_history` (4) · `hardware_encoder` (5) · `encoder_throughput` (6) · `unknown_codec` (a rung
  whose catalog codec does not map — hand-edited data; `codec` then carries the raw catalog value).
  Treat the set as **open**: a client must render an unrecognised reason rather than assume the list
  is closed.
- `encoder_throughput` *(NEW, #506)* means the placed host advertises the codec but reported a
  sustained encode throughput below what the launch needs — the **launch-effective**
  `width × height × fps` (the rung's own values, with an explicit `stream.*` size override applied)
  against `capacity.codec_throughput[codec].max_pixel_rate_mpix_s` (agent-api.md). The size is the
  effective one and not the rung's nominal one because an encoder's budget is spent on the pixels
  actually encoded: sizing a 1440p120 chain down to 720p60 asks for 55 Mpix/s, not 442, and must not
  cost the codec. Note this is the one clamp a `stream.*` size override *retargets* rather than
  bypasses — a `stream.codec` override still skips it outright. It exists because
  throughput is codec-asymmetric on the same silicon (a measured 5090 sustains ~395 Mpix/s on
  `vulkanh265enc` and ~1400 on `vulkanh264enc`), and an encoder that cannot keep up back-pressures
  the compositor rather than dropping — so the session runs below its tier with zero drops and zero
  freezes, invisibly. A host that reports no hint for the codec never rejects this way.
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
| **operator override** (clamp 0) | `override` names the forced codec and the selected rung has `clamps_bypassed: true` with `rejected_by: null`. It **skipped** clamps 2/3, 4, 5 and 6 rather than surviving them — clamp 1 is the only one an override honours, so a `rejected_by: "host_encoder"` here is the `409` path and no session persists |
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

#### `app_launch_state` — in-container application launch state (2026-08-02)
> *Additive amendment — one optional, read-only string on the session resource (user and admin
> session GET/list alike), plus one new conventional `state_detail` value. Changes no existing
> shape, field or status code; a client that ignores it behaves exactly as before.*

Every session read shape gains an optional **`app_launch_state`**:
`"starting" | "client_only" | "game_running" | "game_exited"` — the coarse in-container
application launch state last reported by the agent (`agent-api.md`
`session_metrics.app_launch_state`, which is the normative definition of the enum and its
semantics). Read-only: there is no request field and no endpoint that sets it.

**Absence semantics, by reference to `agent-api.md`.** The field is **omitted entirely** when
the app image does not report launch state, and absence means *unknown*. A client MUST treat an
absent field and an unrecognized value identically — as unknown — and fall back to
transport-level readiness. It is a **hint and a metric, never a session-state authority and
never access control**: `state` remains the only progress signal, and every session must
function identically if the field never appears. An unrecognized value is not an error.

**`state_detail` gains `"game exited"`.** A session that ends because the target title exited
ends `stopped` with `state_detail = "game exited"` — a normal end, not a failure. Like every
other `state_detail` value it is free-text, human-readable and advisory (`schema.md`); clients
must not branch on it for correctness.

#### `failure_code` / `app_log_tail` — app-exit-before-frames detail (first-run-experience §S5)
> *Additive amendment — two optional, read-only fields on the session resource (user and admin
> session GET/list alike). Changes no existing shape, field or status code; a client that ignores
> them behaves exactly as before.*

Every session read shape gains two fields, **always serialized, both `null` unless a terminal
failure warrants them**:
- **`failure_code`** — a machine-readable classification of the failure. Today's only defined
  value is `"app_exited_early"` (the app container exited before it ever presented a frame —
  #463). It sits **beside** `error_message`, not in place of it: `error_message` remains free-text
  operator prose that may be rewritten at any time, while `failure_code` is the stable key a
  client branches on.
- **`app_log_tail`** — the app container's own captured log tail (newline-joined, oldest first,
  ~100 lines bound). App containers run `--rm`, so these lines are otherwise unrecoverable the
  moment the daemon reaps the container; this is the only surviving copy of the decisive output.

Sourced from the agent's `session_state.reason_code` / `session_state.app_log_tail`
(`agent-api.md`) and stored as `sessions.failure_code` / `sessions.app_log_tail` (`schema.md` —
note the wire/column name difference on the first field). Read-only: there is no request field
and no endpoint that sets either. **The UI renders `error_message` as prose and `app_log_tail`
preformatted** — the two fields have different rendering needs and must not be conflated in a
client. Neither field is a session-state authority: `state` remains the only progress signal.

#### `stream.external_width` / `external_height` / `external_resize_supported` / `external_owner` / `rungs` — live external (encoded) resolution (session-display-stream, approved 2026-08-16)
> *Additive amendment, approved 2026-08-16 (PR #15). Extends the `stream` block already
> returned by session GET/list and by `PATCH /v1/sessions/{id}/display`'s `202` body; no existing
> `stream` field changes meaning or presence rule. See the vocabulary note at the top of this
> document and `openapi.yaml` `Stream`.*

The session `stream` block gains:
- **`external_width` / `external_height`** *(optional int)* — the session's **current**
  encoded/streamed size. **Present whenever the control plane KNOWS the current external size** —
  i.e. it has seen either a `202` from `PATCH /v1/sessions/{id}/display` or a `session_metrics`
  sample reporting it (`agent-api.md`) — **including when that size equals the launch size.**
  **Absent means *unknown*, not "at launch":** a control plane that has not yet received either
  signal for this session (freshly restarted, or no `stream_width`/`stream_height` update and no
  metrics sample have landed yet) omits the pair entirely, and a client MUST NOT infer "still at
  launch size" from absence. (`stream.width`/`stream.height` remain "the truth of the profile"
  and never move — `session-display-update` never touched them and this amendment doesn't
  either.) Moved only by `PATCH /v1/sessions/{id}/display`'s `stream_width`/`stream_height`. Like
  `session-display-update`'s render size, this pair is **ephemeral** — not written to the
  `sessions` table — but the control plane keeps an **in-memory cache of the last-known value**
  (populated from the `202` ack path and from the agent's `session_metrics`, `agent-api.md`) so a
  `GET` between updates still reflects the current external size without a client having to
  remember it; **a control-plane restart drops the cache**, and the pair reads as absent again
  until the next `202` or `session_metrics` sample repopulates it.
- **`external_resize_supported`** *(optional bool)* — whether the assigned host's encoder can
  live-resize the stream at all; readback of `agent-api.md` `session_metrics.
  external_resize_supported`. **Absent until the agent reports** (older agent, or no sample yet)
  — absence means *unknown*, never `false`. A client that wants a hard answer either waits for a
  sample or attempts the `PATCH` and handles `409 external_resize_unsupported`.
- **`external_owner`** *(optional string, `"auto"` | `"pinned"`, abr-resolution-fps-ladder
  amendment, approved 2026-08-16 (PR #15))* — who currently owns the current external
  size: the host's ABR resolution ladder (`"auto"`) or a manual `PATCH` (`"pinned"`, see §Pin /
  release semantics below). Readback of `agent-api.md` `session_metrics.external_owner`, cached
  the same way and on the same lifecycle as `external_width`/`external_height` (in-memory only,
  lost on a control-plane restart). **Present only when known AND `external_width`/
  `external_height` differ from the launch `width`/`height`** — the agent itself only reports
  `external_owner` in that same window (there is no meaningful owner of the launch size), so this
  is a narrower presence rule than `external_width`'s "present whenever known, including at
  launch". A client renders an "Auto · `external_width`×`external_height`" chip when this reads
  `"auto"`, a plain size otherwise, and nothing extra when the key is absent.
- **`rungs`** *(array of `[width, height]` pairs, always present for a `running` session)* — the
  discrete external-resolution steps this session may be set to via
  `PATCH /v1/sessions/{id}/display`, filtered to the launch profile's aspect-ratio family and to
  sizes `≤` the launch size (the launch size is always one of the listed pairs). The fixed rung
  table, by family:
  - **16:9** — 3840x2160, 2560x1440, 1920x1080, 1600x900, 1280x720
  - **16:10** — 2560x1600, 1920x1200, 1680x1050, 1440x900, 1280x800
  - **21:9** — 3440x1440, 2560x1080
  - **4:3** — 1600x1200, 1280x960, 1024x768
  **21:9 family membership is by a set of reduced ratios, not one single ratio** — `3440x1440`
  reduces to `43:18` and `2560x1080` reduces to `64:27` (a third, `7:3`, is reserved for a future
  entry); this control-plane table is the reference for family membership, not a computed
  tolerance.
  `stream_width`/`stream_height` on `PATCH /v1/sessions/{id}/display` MUST be one of these pairs
  (after filtering to `≤` launch) or the request is `400 validation_failed` — see
  §`PATCH /v1/sessions/{id}/display`. **Not to be confused with the admin-configured stream-profile
  "rungs"** (§Stream profiles, AS10-01) — this is a fixed, aspect-ratio-derived table for live
  external-resize, unrelated to the admin encode-rung catalog and not itself admin-editable.

### `GET /v1/sessions/{id}/events` — session lifecycle push (SSE, 2026-08-02)
> *Additive amendment — one new read-only endpoint, no existing shape changes. Signed off
> alongside the game-exit lifecycle work; exists to replace client lifecycle POLLING with
> push. The endpoint is for API clients (the browser SPA, a native client); app containers
> never reach the control plane and are unaffected.*

`text/event-stream` (SSE). Owner-or-admin bearer, exactly the authorization of
`GET /v1/sessions/{id}` (`403` otherwise). Browsers consume it with fetch-streaming rather
than `EventSource` so the `Authorization` header carries as normal.

**One event type: `session`.** Its `data` is the same `{ "session": { ... } }` envelope as
`GET /v1/sessions/{id}` — no new vocabulary, no delta encoding. Sent once immediately on
subscribe (snapshot), then on every change to `state`, `state_detail`, `app_launch_state`,
or `health_state` (coalesced, best-effort). The event carrying a **terminal** state is
final: the server sends it and closes the stream. Comment lines (`:`) act as keep-alives
(~25 s cadence).

**A latency optimization, never an authority.** `GET /v1/sessions/{id}` remains canonical.
A client MUST keep polling as its fallback: an older control plane answers `404` (feature
absent — poll exactly as before this amendment), and a dropped stream is re-subscribed or
polled — a client MUST NOT treat stream loss as a session-state signal.

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
  - **`403 forbidden` — the session's OWNER is not entitled to the target app** *(Steam library
    discovery Phase 2 — a **new terminal condition** on this endpoint; the status code itself
    already existed here for the ownership check)*. Same code, same reasoning and same no-role-bypass
    rule as `POST /v1/sessions`.
    - **Why it exists at all.** A swap *is* a launch of a different app into a live session, so
      without this check the launch gate is defeatable in two requests: launch an app you are
      entitled to, then swap into one you are not, and the launch check never sees the second app.
      This is the only enforcement site the spec's §6.3 list does not name; it is here because
      leaving it out makes the rest of the feature decorative.
    - **Keyed on the session OWNER, not the caller** — and this is the load-bearing half. Swap is
      owner-**or-admin**, so keying on the caller would let an admin swapping someone else's
      session launder their own entitlements into that user's session: the session would end up
      running an app its owner may not launch, and its owner keeps the stream. The question this
      check asks is "may this session run this app", and a session's access is its owner's.

### `PATCH /v1/sessions/{id}/display` — change render resolution / interface scale (live)
> *Additive amendment (session-display-update), pre-approved 2026-08-15. New endpoint; no change
> to any existing shape. Modelled on `POST /v1/sessions/{id}/swap` — a best-effort relay to the
> assigned host's agent that never transitions session state and whose rejection is always a
> no-op.*

> **Amendment — session-display-stream (live external/encoded resolution), additive, approved 2026-08-16 (PR #15)
> 2026-08-16, approved 2026-08-16 (PR #15).** Adds `stream_width`/`stream_height` to the request body (below)
> and `409 external_resize_unsupported` to this endpoint's error set. See the vocabulary note at
> the top of this document. **APPROVED (Michael, 2026-08-16); PR #15 merged.**

Lowers (or restores) the **INTERNAL** (app-facing compositor `wl_output` logical mode) and/or
**EXTERNAL** (encoded/streamed) resolution of a live session, and/or pushes a new **UI scale**
(`wp_fractional_scale_v1 preferred_scale`) to its toplevels. INTERNAL and EXTERNAL are
independent request-field pairs on the same endpoint — see the vocabulary note in the amendment
banner at the top of this document. Owner-or-admin (same rule as `DELETE`/`swap`).
```json
// request — at least one field/pair; render_width/render_height and stream_width/stream_height
// are each both-or-neither, independently of one another
{ "render_width": 1280, "render_height": 720, "ui_scale": 1.5, "stream_width": 1920, "stream_height": 1080 }
// 202 — accepted; agent is applying it. Body is the current session (stream gains optional
// external_width/external_height/external_resize_supported/rungs — see §GET /v1/sessions/{id}).
{ "session": {
    "id": "<uuid>", "app_id": "<uuid>",
    "state": "running", "state_detail": "running", "error_message": null,
    "stream": { "width": 1920, "height": 1080, "fps": 60, "bitrate_kbps": 15000, "h264_profile": "main",
                "external_resize_supported": true, "rungs": [[1920,1080],[1600,900],[1280,720]] },
    "created_at": "...", "started_at": "...", "ended_at": null
  } }
```
- **Async, best-effort relay — like swap, but with no state-machine involvement at all.** The
  control plane validates the request, sends `session_display_update` (`agent-api.md`) to the
  assigned host, and returns `202`. **No `state` or `state_detail` transition accompanies this
  call** — unlike swap there is no `swapping`-style detail to poll, because neither render size,
  UI scale, nor external/stream size are part of the session state machine (`schema.md`).
- **Render resolution and UI scale are EPHEMERAL** (unchanged from session-display-update).
  Neither value is written to the `sessions` table, and neither appears on the `Session` resource
  returned here or by `GET /v1/sessions/{id}`. A client reads the live values back from
  `session_metrics` (`agent-api.md` — the *only* authoritative readback), or keeps its own
  last-acked value.
- **External/stream resolution is also EPHEMERAL, with one difference:** it changes what
  is actually **encoded**, so the control plane keeps an **in-memory cache of the last-known
  external size** — populated from this endpoint's own `202` ack path as well as from
  `session_metrics` — and surfaces it on the `Session` resource as `stream.external_width` /
  `stream.external_height` (present whenever the control plane knows the current external size,
  **including when it equals the launch size**; absent means *unknown*, not "at launch" — see
  §GET /v1/sessions/{id}) purely for UI convenience; it is still not written to the `sessions`
  table, and the cache is lost on a control-plane restart (fields read as absent again) until the
  next `202` or `session_metrics` sample repopulates it. `session_metrics` remains the sole
  *authoritative* readback, same as render size.
- **Semantics of `stream_width`/`stream_height`:** changes the **coded size** — what is
  encoded and streamed — at the **next IDR**; there is **no WebRTC renegotiation**
  (`signaling.md` unchanged), the client `<video>` element simply follows the new coded size.
  `stream.width`/`stream.height` (the launch size, "the truth of the profile") are **never**
  affected by this call. **The app-facing (INTERNAL/render) size is never affected by this call
  either (2026-08-16 amendment):** render and external/stream size are independent axes, each
  bounded only by the session's pinned LAUNCH size, never by each other. When the external size
  lands below the current render size, the encoder's scale stage simply downsamples the
  (unchanged) render framebuffer into the smaller encoded frame — the app never sees a mode
  change from a stream-only update, and stepping the external size back up later is a
  passthrough. This replaces the earlier "agent clamps the render size down to match" rule.
- **The encoded stream resolution does not change unless `stream_width`/`stream_height` is
  present.** Absent, `stream.width`/`stream.height` (and the rest of the `stream` block) stay
  exactly as the session was launched (and any subsequent swap) left them.
- **Validation (`400 validation_failed`):**
  - `render_width`/`render_height` supplied together or not at all; both **even**; `16 ≤
    render_width ≤` the session's **pinned LAUNCH** width and `16 ≤ render_height ≤` the
    session's pinned LAUNCH height (2026-08-16 amendment: render is bounded only by the launch
    size — independent of the external/stream size entirely, never by "current external"); `ui_scale`,
    if present, within `[1.0, 3.0]`.
  - `stream_width`/`stream_height` supplied together or not at all; the pair MUST be
    one of the session's `stream.rungs` (§GET /v1/sessions/{id}) — the fixed, aspect-ratio-filtered
    table, always `≤` the launch size. A pair not on that list (wrong aspect family, above launch,
    or simply not a listed rung) is `400 validation_failed`.
  - At least one of `render_width`+`render_height`, `ui_scale`, or `stream_width`+
    `stream_height` must be present.
- **Errors:**
  - `404 not_found` — no such session. **Checked before the ownership check**, so a non-owner
    cannot use this endpoint to probe whether a session id exists.
  - `403 forbidden` — caller is neither owner nor admin.
  - `409 session_not_running` — the session's top-level `state` is not `running`.
  - `409 display_update_rejected` — the agent rejected the update (out-of-range values it
    re-validated, or no `session_state`-adjacent hold on the session), or no `ack` arrived within
    the control plane's normal command timeout. **The session is left untouched in every
    rejection** — same no-op contract as a rejected swap; this call never fails or changes the
    session's `state`/`state_detail`.
  - **`409 external_resize_unsupported`** — the assigned host's encoder cannot live-resize
    the stream at all (`stream.external_resize_supported` reads `false`, or the agent's ack said
    so). Only returned for a request that includes `stream_width`/`stream_height`; a
    render-size-only or ui-scale-only request never sees this code. Like every other rejection on
    this endpoint, the session is left untouched.
- **Semantics recap:** this is a live, best-effort **presentation** (and **encode-size**)
  knob relayed to the host agent — not a renegotiation, not a session-state transition, and (aside
  from the external-size cache, which is a convenience readback, not the source of truth)
  not persisted anywhere in the control plane.
- **Pin / release semantics.** A `stream_width`/`stream_height` PATCH is a
  **statement of ownership**, not just a resize:
  - **PATCH to any NON-launch size ⇒ the session is PINNED.** The host's ABR resolution
    ladder (`abr_ladder_resolution`, per-host Host Settings) stops moving the external size
    for the rest of the session, or until released. A human chose; the ladder must not
    fight them.
  - **PATCH to the session's LAUNCH size ⇒ RELEASED back to auto.** "Back to launch" is the
    release; there is no separate field, and none will be added — the launch size is
    already the one value every client can name and every session accepts.
  - The ownership is **agent-held and ephemeral**, like the size itself. It is reported
    back on `session_metrics.external_owner` (`"auto" | "pinned"`, agent-api.md) and is not
    stored in the `sessions` table. **The control plane also surfaces it on the `Session`
    resource as `stream.external_owner`** (abr-resolution-fps-ladder amendment, approved 2026-08-16
    2026-08-17 — see §GET /v1/sessions/{id} above), the same way `external_width`/
    `external_height` mirror the size — so a client never has to open its own path to
    `session_metrics` just to render the "Auto ·" chip. A client renders "Auto · 1920×1080"
    vs a plain size from that key.
  - A PATCH that the agent **rejects** changes neither the size nor the ownership.
  - On a host where the ladder is off (the default), every session is effectively `"auto"`
    until a PATCH pins it — the flag is still reported, and still released the same way.

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

> **Amendment — first-run-experience §S1 (host readiness), additive.** The host body
> (`GET /v1/hosts`, `GET /v1/hosts/{id}`, and the host list) gains two fields, **both always
> serialized**: `readiness` (array of `{ id, status, summary, remediation }`, or `null` — `null`
> means no amendment-aware agent has reported yet, distinct from `[]` meaning "reported, nothing
> to say") and `readiness_reported_at` (RFC3339 timestamp, or `null` — when the stored
> `readiness` value last changed, so the admin UI can show a report as stale rather than silently
> presenting old evidence as current). Sourced from the agent's `capacity.readiness`
> (`agent-api.md`), stored opaquely (`schema.md` `hosts.readiness` / `readiness_reported_at`).
> **ADVISORY WORDING ONLY** — the admin Hosts detail page and the setup wizard render this as
> guidance; it never gates registration, admission, or scheduling, and no endpoint anywhere
> refuses a request because a check failed. Canonical schema: `openapi.yaml` `Host` /
> `ReadinessCheck`.

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
                   // present cadence — the distribution, not just its mean
                   // (additive, approved 2026-08-23; read present_fps_median):
                   "present_fps_median": 60.0, "present_interval_median_ms": 16.67,
                   "present_interval_max_ms": 33.4, "present_beat_fraction": 0.02,
                   "present_long_frames": 0, "present_n": 59,
                   // RVFC captureTime capability (not abs-capture-time wire proof):
                   "rvfc_capture_time_available": 1, "abs_capture_time_negotiated": 0,
                   // RVFC capture-to-display estimate. Both keys carry the SAME
                   // value this release: glass_to_glass_ms is deprecated (it
                   // overclaims — this is capture-time-to-present, not glass to
                   // glass) and rvfc_capture_to_display_ms is its replacement:
                   "glass_to_glass_ms": 71, "rvfc_capture_to_display_ms": 71,
                   "network_pacing_ms": 7.5,
                   "decode_display_ms": 30.9,
                   // measured client-side and posted, but DROPPED at ingest (they
                   // are not in the browser allow-list) — see the manifest:
                   "input_msg_per_sec": 118, "input_backpressure": 0 },
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
- **The field dictionary of record is `docs/session-trace/metrics.json` in the quasar repo — the
  *metric manifest*.** For every key on either telemetry wire it states the unit, the clock the
  value sits on, the window it summarises, the estimator that produced it, and the key carrying
  its sample count. The taxonomy, the browser ingest allow-list, the diagnostics-panel labels and
  the `trace-format.md` §2 table are all derived from it mechanically. `schema.md`'s field
  dictionary remains the storage-shape reference; the manifest is where a key's MEANING lives, and
  the two must not be duplicated into each other. The manifest is also where a key that is posted
  and then dropped (`input_*`), or declared and never produced (`encode_ms`), is named as such.
- *(Additive, approved 2026-08-23)* **`rvfc_capture_to_display_ms`** replaces `glass_to_glass_ms`
  under a name that does not overclaim: the number is RTP capture-time to browser present, a
  **median over a never-drained ring of up to 600 RVFC samples** — minutes of history sitting
  beside 1 s numbers — and it excludes app render and client scan-out, so it was never
  glass-to-glass. The client posts **both** keys with the same value this release; the taxonomy
  carries `client.rvfc_capture_to_display_ms` alongside `client.glass_to_glass_ms`, and no
  falsifier, derived window or stored series changes.
- *(UI-P6, additive)* `codec_mime_type` is an optional per-sample **string** carrying the
  `getStats()` codec `mimeType` the receiver is actually decoding. It is a sibling of the numeric
  `metrics` object, not a key inside it. The server normalises it to the wire codec vocabulary and
  records it on the session as `negotiated_codec` so an operator can compare it against the
  server-resolved `stream.codec` — see §Codec decision for the full rule (newest usable value wins,
  written only on change, never for a terminal session, junk dropped).
- *(Present cadence, additive, approved 2026-08-23)* `present_fps` is fps from the **mean**
  presentation interval and keeps that meaning forever — the stored series must stay comparable.
  It is no longer the number to read: when the source frame rate equals the client's display
  refresh rate, a single missed vsync doubles one interval and drags the mean, which is how a
  healthy 2560x1440@120 session reported 88-108 fps on 2026-08-22. The six additive keys ship the
  distribution instead. `present_fps_median` and `present_interval_median_ms` are the estimator
  that survives a doubled frame; `present_interval_max_ms` is the window's real longest interval
  (and the source of `client.freeze_detected`'s `gap_ms`, which is now omitted rather than
  fabricated when unknown); `present_beat_fraction` is the share of intervals within +/-20% of
  exactly 2x the median — the inherent vsync beat when source fps == display Hz, not stutter;
  `present_long_frames` counts intervals above 2.5x the median, which the beat never produces, so
  a beat with zero long frames is benign; `present_n` is how many intervals the window held. Below
  five intervals every `present_*` key is omitted rather than computed from a fragment. All six are
  optional: a client that predates them simply omits them, and the server stores what arrives.
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

> **Amendment — #385 item 7 (admin session display names), additive, admin-gated.**
> `AdminSession` additionally carries `username`, `app_name` and `host_name` — the display names of
> the `user_id` / `app_id` / `host_id` the session references. **Additive and omit-when-absent:** no
> existing field changes shape, and a client that ignores them sees the response it saw before.
> ```json
> { "items": [ { "id": "743a921f-…", "user_id": "7622e7d4-…", "username": "deltest",
>                "app_id": "e808ef43-…", "app_name": "Redout: Enhanced Edition",
>                "host_id": "741dc00a-…", "host_name": "tower", "state": "running", "…": "…" } ],
>   "next_cursor": null }
> ```
> - Resolved by a **`LEFT JOIN`** on the admin read path only. Each field is **omitted** when the
>   join resolves nothing, and the client falls back to the truncated id — a session whose app or
>   host row has been deleted must still appear in an operator's oversight view, not vanish from it.
>   `host_name` is additionally omitted while the session is unassigned (`host_id` is `null`), which
>   is the routine case, not an edge.
> - **Admin-gated, and deliberately not on the user-facing `Session`.** `GET /v1/sessions` and
>   `GET /v1/sessions/{id}` are unchanged: the owner-facing session body does not begin disclosing
>   other rows' names. This is the documented additive/admin-gated exception in §Authorization.
> - Restores the signed-off `admin-sessions` mockup, which has always specified the User cell as
>   *id + username stacked* and the App cell as the app's name.

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

**`bench.window` is instrument-only.** It is emitted solely by the SPA's bench mode
(`?bench=1`), which decodes the `quasar-benchapp` per-frame marker in the page and posts one
aggregate per second; an ordinary user session never emits it. Its payload is free-form like
every other event payload — deliberately NOT a persisted stats key, because the browser stats
series is allow-listed per `schema.md` and a measurement instrument must not extend that
contract. Additive: no existing event's shape changes, and a control plane that does not know
the type simply drops it (the bench harness therefore also reads its windows straight out of
the page, and does not depend on this ingest path).

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
  "classifier": { "…": "the Verdict value — see below" }
}
```
- `classifier` is the **Verdict** (ST-09): the same `verdict` + `evidence` it always carried, plus
  `reason`, `window`, `clock`, `evidence_tier`, `falsifiers` and `thresholds_version`. It is
  identical to the body of `GET .../verdict` below.

### `GET /v1/admin/sessions/{id}/verdict` — the Verdict alone (admin)
### `GET /v1/sessions/{id}/verdict` — the same Verdict, owner-or-admin
*(ST-09, additive, approved by Michael 2026-08-23.)* Both return **the same body**: the value the
bundle carries as `classifier`, without the series and events. The admin form is admin-gated
(`403` before lookup, `404` unknown session); the owner form applies the ownership check
`DELETE /v1/sessions/{id}` uses (`404` unknown, `403` not yours) and is rate-limited per session.
Both accept the bundle's `?from=&to=` unix-ms window and apply the same default (5 min) and clamp
(`[2,10]` min). Neither has any session authority: reading a verdict never changes a session.
```json
// 200
{
  "verdict": "likely_network_congestion",
  "evidence": [ "network congestion: {\"packets_lost_delta\":14,\"rtt_ms_p95\":60}",
                "client tab not hidden during the window" ],
  "reason": "Packet loss and round-trip time both crossed their congestion thresholds over a 300 s window (612 host, 598 client samples).",
  "window":  { "from_ms": 1735689300000, "to_ms": 1735689600000, "n_host": 612, "n_client": 598 },
  "clock":   { "quality": "measured", "offset_ms": -3.2, "uncertainty_ms": 1.8 },
  "evidence_tier": "full",
  "falsifiers": [
    { "name": "transport.packets_lost", "estimator": "delta", "value": 14, "op": ">",  "threshold": 5,  "unit": "count", "n": 300, "holds": true },
    { "name": "transport.rtt_ms",       "estimator": "p95",   "value": 60, "op": ">",  "threshold": 50, "unit": "ms",    "n": 300, "holds": true },
    { "name": "encoder.encode_ms",      "estimator": "p95",   "value": 4.6,"op": "<",  "threshold": 16, "unit": "ms",    "n": 298, "holds": true },
    { "name": "client.is_hidden",       "estimator": "any",   "value": 0,  "op": "==", "threshold": 0,  "unit": "bool",  "n": 298, "holds": true }
  ],
  "thresholds_version": "2026-08-23.1"
}
```
- **`verdict`** is **observational only** — today one of `likely_encoder_saturation` /
  `likely_network_congestion` / `likely_client_presentation_limit` / `nominal` /
  `indeterminate_client_hidden` / `unknown`, with its `evidence` logged. **No automatic action.**
  `nominal` = healthy session (no negative signal AND the tab was not hidden);
  `indeterminate_client_hidden` fires when the *only* reason no verdict was reached is the
  `client.is_hidden` guard; `unknown` is reserved for genuine insufficient-data windows.
  **The control plane owns this vocabulary and grows it: a string a consumer does not recognise
  is DATA. Report it verbatim; never fail on it.**
- **`falsifiers`** are the few numbers that would overturn the verdict — each is a taxonomy series
  `name`, the `estimator` applied to it over the window (`p10|p95|max|delta|mean|any`), the
  computed `value`, and the `op`/`threshold`/`unit` of the condition the verdict relies on.
  `holds` is whether the data satisfies that condition. A series with no samples in the window
  reports `value: null`, `n: 0`, `holds: false` and a `note`. `evidence` (prose) and `falsifiers`
  (numbers) are deliberately separate: neither replaces the other.
- **`evidence_tier`** — `full` (both host and client contributed ≥3 samples **and** the clock was
  measured), `host_only` / `client_only` (one side missing, or the clock unmeasured so the two
  sides cannot be aligned), `insufficient` (neither side reached 3 samples).
- **`clock.quality`** is `measured` only when the session has a `session_trace_clock` row;
  `unmeasured` caps `evidence_tier` below `full` and is called out in `reason`.
- **`thresholds_version`** identifies the golden threshold set the falsifier thresholds came from
  (`docs/session-trace/thresholds.json` in the quasar repo).

## On-demand capture (session-capture, approved by Michael 2026-08-23)
> *Additive, admin-gated, observability-only amendment. Two new endpoints and one new key on
> the diagnostic bundle; no existing shape, status code, or endpoint changes. Reads and the
> arm are both `RequireAuth → RequireAdmin` (`403` before any resource lookup). Wire contract:
> `agent-api.md` §`session_capture` (downstream) and §`session_trace_event` `diag.*`
> (upstream). **No new table and no migration** — a capture is stored as a
> `session_trace_events` row, exempt from the rolling prune. No session authority: arming,
> polling, or reading a capture never changes a session.*

A **capture** is a bounded, admin-triggered observation of a live session: arm it, the agent
observes within a byte **and** time budget, and reports once. It exists because the three
things that cost the most debugging time in August 2026 — *what is the encode graph actually
wired as right now*, *what is the encoder's live property set*, and *what do encode times look
like at a finer grain than the heartbeat* — were each answerable only by an ssh hop, a rebuild,
or a guess. It is deliberately **not** a probe: nothing is inserted into the media path.

### `POST /v1/admin/sessions/{id}/capture` — arm one capture (admin)
```json
// request
{ "kind": "pipeline_dot", "params": { "windows": 20, "window_ms": 250 } }
// 202 — the agent acked; the result arrives later
{ "capture_id": "<uuid>", "kind": "pipeline_dot", "session_id": "<uuid>",
  "accepted_at": "2026-08-23T12:00:00Z" }
```
- `kind` is required: `pipeline_dot` | `encoder_props` | `burst_stats`. `params` is optional
  and read only by `burst_stats` (`windows` 1–40, `window_ms` 100–1000; both clamped, by the
  control plane and again by the agent).
- The `capture_id` is minted here, before dispatch, so the caller holds the join key the moment
  it gets its `202`.
- **`202`, never `200`:** the ack means *armed*, not *done*. The result is fetched from the
  read below.

| status | code | meaning |
|---|---|---|
| `202` | — | armed; poll the read below |
| `404` | `not_found` | no such session (to an admin) |
| `409` | `capture_busy` | a capture is already in flight for this session — **single-flight, never queued** |
| `409` | `session_not_running` | the session is not `running`, so there is nothing to observe |
| `422` | `capture_kind_unsupported` | the agent does not know this `kind`, or cannot do it on this session right now |
| `501` | `capture_unsupported` | this host's agent predates captures (it never acked). Rebuild the agent; retrying will not help |
| `503` | `agent_not_connected` | the session's host has no live agent connection |

### `GET /v1/admin/sessions/{id}/captures/{capture_id}` — read the result (admin)
```json
// 200 — the stored diag.* event, verbatim payload plus its timestamp
{ "capture_id": "<uuid>", "kind": "pipeline_dot", "ts_unix_ms": 1735689600000,
  "encoding": "gzip+base64", "content_type": "text/vnd.graphviz",
  "data": "<base64 of gzip>", "bytes": 48213, "compressed_bytes": 6104,
  "original_bytes": 48213, "truncated": false, "duration_ms": 42 }
```
- **`404` until the result arrives.** That is the poll signal, not an error: arm, then poll
  until `200` or the budget's `max_ms` plus a small grace elapses. A capture that failed after
  being accepted returns `200` with `error` set — so a poller always terminates on a body.
- `encoding` is `gzip+base64` (payload in `data`) or `json` (payload in `json`).
- The read is **not** window-bounded, unlike every other trace read: captures are sparse,
  explicitly requested, and exempt from the rolling prune, so scoping them to a recent window
  would hide exactly the artifact the caller asked for.

### The bundle gains `captures[]`
`GET /v1/admin/sessions/{id}/diagnostic-bundle` grows one key: every capture belonging to the
session, **regardless of the bundle's window** (same reasoning as above — they are sparse and
prune-exempt), each entry the shape of the read above.
```json
"captures": [
  { "capture_id": "<uuid>", "kind": "encoder_props", "ts_unix_ms": 1735689600000,
    "encoding": "json", "content_type": "application/json",
    "json": { "encoder_factory": "vulkanh264enc", "…": "…" },
    "bytes": 812, "compressed_bytes": 812, "original_bytes": 812,
    "truncated": false, "duration_ms": 3 }
]
```
The array is always present (empty when the session has no captures), and the payloads ride
inside it — bounded by construction, since no capture may exceed its `max_bytes`.

### What a capture may never contain
The agent enforces the envelope (`agent-api.md` §`session_capture`), and it is repeated here
because it is the reason this surface is safe to expose at all: no media (pixels, audio,
bitstream), no input or microphone, no credentials, no `node_secret`, no environment
wholesale, no path outside the session's scratch directory. `encoder_props` reads an
**allow-list** of property names and elides long string values. A capture is bounded in bytes
and in time, single-flight per session, and refused rather than queued.

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
  "username": "mike", "app_name": "Steam", "host_name": "tower",
  "provider": "volume", "ref": "quasar-home-…", "bytes_used": 123456789,
  "created_at": "…", "last_used_at": "…", "gc_after": null } ], "next_cursor": null }
```

**Resolved names (`username`, `app_name`, `host_name`) — additive, nullable.**
*Additive amendment: three new fields on the storage-home object. Nothing existing changes
shape; a client that ignores them behaves exactly as before.*

Each is the human-readable name of the entity named by the id beside it —
`users.username`, `apps.name`, `hosts.node_name`, i.e. the same column the admin Users,
Apps and Hosts surfaces already display, so the storage listing agrees with them. They exist
because an operator reading this list cannot tell whose home is holding 249 GB from a
truncated UUID.

**All three are nullable, and that is not an oversight.** `user_id`, `app_id` and `host_id`
are themselves nullable: a home routinely **outlives** the user, app or host it belonged to
(the owning columns are `ON DELETE SET NULL`), and such a row **must stay in the listing** —
its bytes are still on disk, and an orphaned home is exactly what an admin opens this page to
find. So the name is resolved with an outer join and comes back `null` whenever the
referenced row is gone or the id was never set. Dropping the row instead of nulling the name
would hide the very storage the endpoint exists to account for. A client must render a
missing name as an explicit placeholder (the id, or a dash), never treat it as an error.

**The ids remain authoritative.** Names are display-only and are not unique, not stable
across a rename, and not resolvable back to a row. `DELETE /v1/admin/storage/homes/{id}` and
every other action key off the ids, which are still present on every item.

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

## Derived tiles (Steam library discovery, Phase 3)

> *Additive amendment (migration 0044), requires Opus + explicit human sign-off — it also carries
> the `AppKind` widening. DDL companion: `schema.md` `apps.parent_app_id` / `origin` /
> `library_provider`, `apps_derived_shape_ck`, `apps_parent_external_uk`. It sits here, beside
> §Storage, because everything that is hard about it is a **storage** question.*

A **derived tile** is an `apps` row with `parent_app_id` set. It carries **identity and
presentation only** and borrows **everything executable** from its parent at launch.

| Lives on the tile | Resolved from the parent at launch |
|---|---|
| `name`, `description`, `kind`, `cover_url` / `hero_url` | `image` / `runtime_spec` / `runtime_preset_id` |
| `external_source`, `external_id`, `origin` | `managed_home`, `home_container_path`, and the user's home |
| `enabled`, `default_profile_id`, `profile_policy` | `default_vram_mb`, `default_encode_slots` |
| favourites, entitlements | `default_width` / `height` / `fps` / `bitrate_kbps`, and the mounts |

The tile contributes exactly one thing to execution: an environment override,
`STEAM_STARTUP_FLAGS = "-bigpicture -applaunch <external_id>"`, merged over the parent's
`runtime_spec.env` at dispatch. **Merge order is parent first, tile second, tile wins** — the same
rule UI-P3 established for runtime presets. The flag string is composed server-side from a fixed
template and the already-validated `external_id`; it is never concatenated from stored free text,
and the template is not operator-editable.

**The tile's own resource columns are never read.** Admission resolves `default_vram_mb` and
`default_encode_slots` from the **parent**. This is not cosmetic: `cb97bfb` is a live incident in
which omitted create fields zeroed encode slots and bypassed admission entirely, and a tile created
by a background job (Phase 4) is precisely the shape that incident took.

### The rule that must not be broken

**Every guard, lock and placement decision keyed on an app's identity for a STORAGE purpose is
keyed on the tile's home-owning app, not on the tile.** The tile does not own a home; it borrows
its parent's:

```
homeAppID(app) = app.parent_app_id ?? app.id
```

Missing one site is not a cosmetic bug. The single-writer guard exists because **two Steam clients
on one `steamapps` tree is a documented corruption class** — a mid-write Steam death marks the
install unclean and forces a full checksum re-verify on next boot. A guard keyed on the tile
returns "not in use" for a session that is actively writing.

The same substitution applies to the **tombstone** guard on `DELETE /v1/admin/storage/homes/{id}`
(§Storage): its `409 home_in_use` must see a live derived-tile session as using the parent's home,
or an admin can tombstone a home that a running session is writing to. That is a data-destruction
path, and it is the one where a wrong guard fails **silently**.

### Launch behaviour: three `409`s

All three apply to **`POST /v1/sessions` and `POST /v1/sessions/{id}/swap` alike** — a swap is a
launch of a different app into a live session, so a rule that only guarded the launch would be
defeatable in two requests.

**`409 home_in_use`** — *not a new code (P5-01), but it now keys on the home-owning app and its
body carries `session_id`.*

- With a session of the **parent** live, **every** derived tile is refused.
- With **any** derived tile live, the **parent** (the Launcher tile) is refused, as is every
  sibling tile.
- **It is one lock, held by the parent app's home**, and the Launcher tile is simply its most
  visible holder.

```json
// 409
{ "error": { "code": "home_in_use",
             "message": "Steam is already running",
             "session_id": "…" } }
```

`session_id` is **omitted, never emitted empty**, when the guard fired but could not name the
conflicting session — so a client branches on its presence and never renders a link to nowhere.
Nesting an extra field inside the error object is the existing idiom (`live_sessions` on
`restart_required`), not a new envelope shape.

> **The client requirement, and it is not optional polish.** Keeping the Launcher tile visible
> (rather than hiding the provider app after discovery) makes this lock **user-visible for the
> first time**. From the user's side they clicked a *game* and were told a *different app* is in
> the way, so a raw error toast reads as a bug.
>
> 1. Render it as **"Steam is already running — go to your session"**, with a link built from
>    `session_id`. Not a generic launch failure.
> 2. Name the **Launcher tile** — what the user sees in their library — **not** a parent app id.
> 3. While a session backed by the parent home is live, the client *should* mark the rest of that
>    family blocked rather than let the user discover it by clicking. It has what it needs: the
>    live session's `app_id` and each tile's `parent_app_id`. **This is a presentation nicety. The
>    enforcement is this `409`** (invariant #6: a client-side gate is never the access control).
>
> Switching games *inside* a live session is the desirable end state and is future work; it is not
> smuggled in here.

**`409 home_not_provisioned`** — *new.*

A derived tile **provisions nothing**. The launch path resolves the home read-only: it requires a
live `user_homes` row for `(caller, parent, host)` and **never creates one**. A create-on-miss here
would be the worst possible behaviour — an empty directory is provisioned, the game is not in it,
Steam boots into a library with nothing installed, and the session reaches `running` looking
healthy. The same upsert would also **un-tombstone** a home an admin had just marked for reaping.

```json
// 409
{ "error": { "code": "home_not_provisioned",
             "message": "launch Steam once on a host to create your library, then try again" } }
```

The message names the **parent app**. This is checked **before placement**, and **no `user_homes`
row is written** on this path.

> **On a swap the constraint is narrower, and this is the one place the two endpoints genuinely
> differ.** A swap is pinned to the **live session's host** and has **no placement step**, so
> there is nowhere to re-pin it to: the parent's home must exist **on that host**. A tile whose
> library lives on a *different* host is refused here even though launching it directly would
> succeed, because the launch path resolves the home host and pins there. The remedy is therefore
> "launch it directly", not "try again".
>
> This returned `500 internal` until a review caught it: the sentinel survived the wrap and only
> the mapping was missing, so an ordinary, user-correctable condition was reported as a server
> fault — and the contract documented the code on the launch path only, so **both sides were wrong
> about a reachable state**.

**`409 parent_app_disabled`** — *new, and a behaviour change ruled in at review.*

**A tile's own `enabled` and its parent's `enabled` are ANDed at launch.** Launching or swapping
into a derived tile whose **parent** is disabled is refused, and the message **names the parent**.

> **This deviates from the implementation spec's field table, deliberately, and the contract is
> the durable artifact.** That table lists `enabled` under *"lives on the tile"*, which reads the
> other way. It says which **row** a field is read **from**; it does not say the parent's
> `enabled` is irrelevant. **A derived tile has no independent existence** — image, runtime,
> mounts and home are all the parent's — so launching a tile **is** running the parent, and
> `enabled = false` on an app in this system means *"stop this from running"*. Stated here so
> nobody re-litigates it from the table.

Before the fix this demonstrably did not hold: a reviewer verified empirically that with the
parent `enabled = false` the tiles **still launched** — dispatching the disabled parent's image,
against the disabled parent's home, with no error. An operator taking Steam out of service over a
bad image tag, a mid-upgrade window or an incident would have had every game tile keep running
that exact image, with nothing in any UI warning them, because the tiles are separate rows and
they still look enabled. That is the inverse of a kill switch.

```json
// 409
{ "error": { "code": "parent_app_disabled",
             "message": "this game is launched through \"Steam\", and that app is currently disabled — ask an operator to re-enable it" } }
```

- **The tile is not modified and still appears in the caller's library.** Its own `enabled` is
  untouched and `GET /v1/apps` still lists it, so a client **cannot pre-empt this** from the app
  object. Disabling a parent is not a bulk edit of its children, and re-enabling it restores every
  tile at once — which is the point of a kill switch.
- **Its own code, not the generic `conflict`**, on the same reasoning as `home_not_provisioned`:
  the remedy is an action on a **different app**, by **somebody else**. A client cannot derive
  "ask an operator to re-enable Steam" from a message string it is not allowed to parse, and the
  refusal is emphatically not the caller's fault.
- **The parent's name rides in the `message`, not in a structured field**, and unlike
  `home_in_use`'s `session_id` that asymmetry is deliberate. `session_id` exists because a client
  turns it into a **link** and must branch on its presence; this has no client action attached —
  there is nothing to navigate to. A field nobody consumes is contract surface with no purchase.
  The message is human-readable and forwardable; **it is not a machine-readable channel and must
  not be parsed.**

### Placement: a hard pin, not an affinity

A derived tile is placed with a **hard host constraint** — the host holding the caller's home for
the parent app, resolved pre-schedule and passed through the **existing** pin mechanism, so this
adds no placement machinery. If no such host exists, the launch fails with `home_not_provisioned`
above, **before** placement.

**This is deliberately stricter than the existing locality preference**, which is only a sort key
and loses to a free host. A derived tile cannot fall back to a free host and will refuse where a
normal app would have run. That is the correct trade: **a session that refuses to start is
recoverable in one click; a session that silently creates a second empty Steam library is a
support ticket and a wasted disk.** It also means the pin behaves identically under either
placement policy, so an operator never has to change one to make this correct.

### Shape rules on the write path

`AppWrite` accepts `parent_app_id` (tri-state on patch: absent = unchanged, explicit `null` =
clear, uuid = set) and `library_provider` (absent = default/unchanged, explicit `""` = a deliberate
un-marking). It does **not** accept `origin`.

A row with `parent_app_id` set must satisfy, at the handler and again at the database
(`apps_derived_shape_ck`):

| rule | why |
|---|---|
| `runtime_spec = '{}'` | the merge happens at launch, so a parent edit reaches every tile with no re-sync. A validated Tower experiment hardcoded a host path into a tile's `runtime_spec.mounts`; the `CHECK` is why that cannot ship |
| `managed_home = false` | the tile borrows a home, it does not declare one. The single-writer guard must read the **parent's** `managed_home`, or it does not fire for derived tiles at all — the exact inverse of its purpose |
| `runtime_preset_id IS NULL` | a preset is container configuration, and the tile contributes none |
| `library_provider = ''` | a tile cannot itself be a provider |
| `external_source <> '' AND external_id <> ''` | the tile is nothing without the title it names, and `external_id` carries the injection grammar |

`parent_app_id` must name an app that is **not itself derived**: one level, never a chain.
`apps_parent_external_uk` makes one tile per `(parent_app_id, external_source, external_id)`
**fleet-wide** — a duplicate is `409 conflict`, and it is what keeps the catalogue bounded by the
union of installed titles rather than by users × titles.

**Deleting a parent that has derived tiles** is refused with `409` listing them, unless the request
explicitly opts in — mirroring the existing in-use `409` pattern. The FK cascade is the integrity
backstop underneath that confirmation, not the primary UX.

**Every one of those rules answers `4xx`, never `500`.** `apps_derived_shape_ck` maps to
`400 validation_failed` and `apps_parent_external_uk` to `409 conflict`, each naming what the
operator can fix. A `CHECK` violation reaching a client as `500 internal` is a lie: the request is
malformed, the server is not.

Two rules the database **cannot** express as a row `CHECK` are enforced at the handler and answer
`400`:

- **`parent_app_id` must name an existing app that is not itself derived.** Home resolution
  substitutes the parent **exactly once**, so a grandchild would resolve its home to a *tile*,
  which owns none.
- **`library_provider` may not be set on a derived tile** — evaluated against the **effective
  patched-or-stored** shape, not the request alone, so a two-request path cannot assemble a state
  that a single request would be refused for.

**`origin` is read-only and is deliberately not on `AppWrite`.** An admin create is `'manual'` by
construction; only a library-discovery sync writes `'discovered'` (**Phase 4's reconciler does, since
migration 0045**), so the write path buys no capability anything needs.

**What settles it is that BOTH relabel directions are footguns under `apps_parent_external_uk`**,
and it is worth stating because the first instinct is that only one of them is:

- Relabelling a discovered tile `'manual'` — the obvious *"stop the reconciler managing this"*
  move — does not stop it. The tile still occupies its `(parent, source, appid)` slot, so a
  reconciler reading it as *"not mine, therefore missing"* **cannot re-create it either**: a
  resurrection loop with no visible cause. The specified way to suppress a tile is the
  `library_appid_rules` Ignore rule, which is durable and idempotent across every sweep.
- Hand-creating a tile pre-labelled `'discovered'` so a sweep adopts it has the **same root cause**
  from the other side.

The field being unwritable removes both. It also matches this API's standing rule for provenance:
`entitlements.granted_by` is likewise never client-settable, and `favourite` is resolved rather
than assertable.

> **The enforcement is `DisallowUnknownFields()`, not a field-specific rejection branch, and the
> error message reflects that.** Because `origin` is simply absent from `AppWrite`, sending
> `{"origin": "discovered"}` to `POST /v1/apps` or `PATCH /v1/apps/{id}` is
> **`400 validation_failed` with the generic malformed-JSON-body message** — the same answer any
> unknown key on this shape gets — rather than a message naming the field. A client author
> debugging a `400` here should look for a stray `origin` in the body, not for a validator. On
> both verbs the request is rejected *whole*: no app row is created, and no stored value is
> touched.

### Deleting a provider app

`DELETE /v1/apps/{id}` gains a **second** refusal, mirroring the existing refuse-if-in-use pattern
and reusing the generic `conflict` code rather than adding one. Deleting an app that has derived
tiles is `409` **unless** the caller opts in with **`?delete_derived=true`**, because
`apps.parent_app_id` **cascades**: the delete would silently take every derived tile with it, and
each tile's `app_artwork` and `user_app_favourites` rows with those.

```json
// 409
{ "error": { "code": "conflict",
             "message": "this app has derived tiles that will be deleted with it — re-send with ?delete_derived=true to confirm",
             "derived_tiles": [ { "id": "…", "name": "Portal 2" }, … ] } }
```

**The body lists the tiles; it does not count them.** The point of a confirmation is that the admin
sees *what* they are about to destroy, and "12 tiles" is not that. The list is capped, and an empty
array means the tiles could not be listed — never that there are none, since otherwise this `409`
would not have been raised. Only the exact string `"true"` opts in, so a typo (`"1"`, `"yes"`,
`"TRUE"`) refuses rather than deletes.

---

## Library discovery (Steam library discovery, Phase 4)

> *Additive amendment (migration 0045), requires sign-off. Seven routes and one settings field;
> no existing shape, status code, endpoint or behaviour changes, and no new error code. DDL
> companion: `schema.md` `library_scans`, `library_observations`, `library_appid_rules`, and
> `instance_settings.library_discovery_enabled`. **`agent-api.md` is byte-identical.** It sits
> here, after §Derived tiles, because everything it produces is a derived tile.*
>
> *Extended by the **force-scan** follow-on (additive, admin-gated, requires sign-off, no
> migration): one route, `POST /v1/admin/library/scan`, plus the on-enable janitor nudge on
> `PATCH /v1/admin/settings` and the agent's 60s poll cadence. **`agent-api.md` is still
> byte-identical.** See the amendment block above.*

Discovery is the job that turns *"this user has Portal 2 installed under their Steam home"* into a
derived tile plus a `granted_by='provider'` entitlement, and back again when they uninstall it.
It runs on a schedule, it publishes automatically, and it is off until an operator turns it on.

### Where it runs, and why it is not a WebSocket message

The control plane never touches a host filesystem (architecture invariant #1) — that is the
reason `/v1/agent/storage/gc-*` exists at all — and on a real deployment the homes root is
bind-mounted only into the node-agent container. **So discovery runs agent-side or it does not
run**, and the only open question was which channel carries it.

It reuses the **#175 pull channel shape**, and therefore **`agent-api.md` is byte-identical**: a
new additive HTTP surface, not a WebSocket-message change. The alternative — a new `agent-api.md`
message — would have cost a frozen-interface sign-off and a `quasar-client` pin bump, and bought
nothing: discovery is not latency-sensitive, it is idempotent, an in-flight scan is a **database
row** rather than state on a socket, and `spawn_gc_reaper` is the existing precedent for exactly
this shape. The pull model survives an agent reconnect for free, because there is nothing to
correlate across the gap.

### The agent pull channel

**Authentication is the node-secret scheme, not a user bearer token** — identical to
§Agent storage GC, and deliberately not a second implementation of it. The agent presents
`Authorization: Bearer {node_secret}` plus `X-Quasar-Node: {node_name}`; the control plane
verifies against `hosts.node_secret_hash` with a constant-time compare, and the resolved
`host_id` scopes every query, so an agent can only ever see and report scans for its own host.
Any failure ⇒ `401 unauthorized`, never distinguishing which. `/v1/agent/` is already exempt
from the HTTPS redirect.

#### `GET /v1/agent/library/scan-pending`
Claims up to 50 pending scans for the calling host (`FOR UPDATE SKIP LOCKED`, so two agents
polling simultaneously get **disjoint** sets rather than blocking on each other) and returns the
job payloads.

```json
{ "scans": [ {
    "scan_id": "…",
    "root_path": "/mnt/user/appdata/quasar/homes/…",
    "relative_roots": [".local/share/Steam/steamapps", ".steam/steam/steamapps"],
    "max_entries": 512,
    "max_manifest_bytes": 1048576
  } ] }
```

`relative_roots` and the two bounds are **control-plane-supplied**, not agent constants, so a
bound can be tightened without an agent release. An empty list is the steady state and is also
what the endpoint returns when the feature is switched off — the switch is re-read **here** as
well as in the scheduler, because the scheduler is what stops rows being *created* and this is
what stops an already-queued row being *handed out* after an operator turned discovery off.

**Poll cadence is not scan cadence, and the contract should not let anyone conflate them again.**
The agent polls this endpoint every **60 seconds** (it shipped at 6h, which was the defect: two
independent six-hour timers stacked into a twelve-hour worst case before a single tile appeared).
A poll is **one indexed query** against `library_scans` scoped to the calling host, returning an
empty list almost always; the expensive half — the filesystem walk over every user's home — is paced
entirely by the control plane's `QUASAR_LIBRARY_SCAN_INTERVAL`, unchanged. Polling faster therefore
buys queue latency and costs nothing that matters, and it is **agent behaviour, not wire shape**:
the payloads, bounds, claim semantics and node-secret auth here are untouched, which is why
**`agent-api.md` remains byte-identical** and no client pin moves. The 60s figure is the current
agent's cadence rather than a wire guarantee — a client must not derive a deadline from it — but
`POST /v1/admin/library/scan` is only useful *because* of it: a "scan now" button in front of a
six-hour poll would have made an operator wait six hours for their own button.

`root_path` is control-plane-supplied and the agent **validates containment against its own
configured homes root before walking it**. A path outside that root is refused and reported as an
error, never walked: the agent does not trust the control plane with a filesystem path any more
than it would trust a client.

#### `POST /v1/agent/library/scan-report`
Body:

```json
{ "scan_id": "…", "ok": true,
  "entries": [ { "external_id": "620", "name": "Portal 2",
                 "install_dir": "Portal 2", "size_on_disk": 0, "state_flags": 4 } ] }
```

`{ "ok": false, "error": "…" }` reports a failure — a refused path, no configured home root, a
walk that could not complete. **A failed scan changes nothing but its own row.** Revocation is
driven by *absence*, and absence is exactly what a transient error looks like, so "reconcile
whatever entries did arrive" would make a partial walk indistinguishable from a user uninstalling
their library. `entries` and `error` are both optional on the wire (the agent omits whichever is
empty); an absent `entries` on an `ok: true` report is a legitimate "this user has nothing
installed" and is reconciled as such.

Response is `{ "accepted": true }`. `404 not_found` for a scan the calling host does not own —
**indistinguishable from one that does not exist, deliberately**, since a `403` would confirm the
existence of another host's scan id to anyone holding one valid node secret. `409 conflict` for a
scan that is not currently claimed, which is where a duplicate report after a successful
reconcile lands.

Accepting a report runs reconciliation **in one transaction** with marking the scan reported:
observations are upserted and the ones this scan did not list are pruned, the suppression ladder
is evaluated **once**, tiles are created for what publishes, tiles are disabled and their provider
entitlements revoked for what suppresses, and the scanning user's provider entitlements are
granted and revoked. Tile creation and the first entitlement commit **together**, because
auto-publish means they are no longer separated by an admin decision and a tile that exists
without its entitlement is a window in which a user sees nothing where the feature just claimed
to add something.

**This is the first thing that ever writes `entitlements.granted_by='provider'`** — a value
Phase 2 put in the `CHECK` and deliberately left unwritten, which is why there is no `ALTER` here
and why revoking one was already a working path the day the first appeared. Phase 2 also left
`source_ref` as free-form provenance "for a Phase 4 grant"; the value it now carries is
**`library:<external_source>:<external_id>`** — e.g. `library:steam:620`. It is provenance for a
human reading a row, not a key anything joins on. `granted_by='admin'` rows are **never** touched
by the sync in either direction: an operator's grant survives an uninstall, because an admin said
so and a filesystem did not.

### The agent never learns a user

**There is no user id, no username and no user-derived field anywhere in either direction.** The
job is a scan id, an opaque path and two bounds; the report is a scan id, a flag and a list of
installed titles. The `scan_id → (user, app, host)` mapping lives in `library_scans` and is
resolved on receipt.

This is `agent-api.md`'s **P2-01 verdict — that per-user concerns never reach the agent** —
honoured literally, and it is a **guarantee of this interface**, not an implementation detail that
happens to hold today. A change to these two payloads that put a user on the wire would be the
wrong change even though the JSON would still validate.

### PII: a key allow-list, and a report shape with nowhere to put a person

Every `appmanifest_*.acf` carries **`LastOwner`, a SteamID64** — a persistent, globally unique,
externally resolvable identifier for a real person's Steam account.

**Rule: `LastOwner` is never read, never logged, never transmitted and never persisted.** The
mechanism is two independent things, and both are part of this contract:

1. the agent parses with a **key allow-list** — `appid`, `name`, `installdir`, `SizeOnDisk`,
   `StateFlags`, everything else discarded at parse time. An allow-list and **not** a denylist,
   because Steam adds keys over time and a denylist leaks every future key by default;
2. the report entry has **exactly those five fields**, so there is nowhere for a sixth value to
   travel even if the parser were wrong.

**Two of the five have no consumer, and this is the place that says so** rather than leaving a
reader to assume they are populated for some other purpose. `install_dir` and `size_on_disk` are
**collected, not used**: `install_dir` existed to feed launch verification, which was dropped, and
`size_on_disk` was never consumed by anything. They stay because narrowing a PII containment
allow-list buys nothing, while widening it later means re-touching the one piece of code whose job
is to touch as little of the manifest as possible. `state_flags` is likewise unread — on the live
capture it was `4` for all five Valve tools **and** for three of the four real games, so nothing
branches on it.

**`name` is retained deliberately** and its importance went *up*, not down: it is the tile title
and it is **half of the denylist key** below, so it is load-bearing for correctness. It is a game
title, not a person.

**The second-order PII, said out loud.** `library_observations` and the provider-written
`entitlements` rows are together **a per-user record of which games a person has installed**. That
is inherent to the feature, admins can see it by design — the entitlement UI and the "Seen, not
published" read both expose it — and it is **new**: nothing in Quasar recorded a per-user game
inventory before this phase. It is only appids and titles, and it cascades away with the user, but
it is worth naming rather than discovering. Note also that observations record **every** appid
including suppressed ones (that is what makes a wrongly-suppressed game recoverable), so the
per-user inventory is complete rather than filtered.

### The suppression ladder

On the live box **5 of 9** manifests were runtimes rather than games (Proton Experimental, Proton
Hotfix, two Steam Linux Runtimes, Steamworks Common Redistributables), and **no structural field
distinguishes a tool from a game** — `StateFlags` was `4` for all of them and for most of the real
games too. So filtering is on the two values the parser must extract anyway, in two layers over
one decision, evaluated per observed appid, **first match wins**:

1. an `allow` rule exists → **publish**;
2. an `ignore` rule exists → **suppress**;
3. the built-in denylist matches the appid **or** a case-insensitive name prefix → **suppress**;
4. otherwise → **publish**. This is the default, and it is what "auto-publish" means;
5. *(only when `QUASAR_STEAM_APPDETAILS_LOOKUP` is on)* Valve's store says this appid's `type` is
   not `game` → **suppress**.

**Rung 1 above rung 3 is what makes a wrongly-denylisted game recoverable without a release**, and
rung 2 above rung 4 is what makes an operator's Ignore stick. **Rung 5 is appended, never
substituted**: it is consulted *only* for appids rungs 1–4 would publish and *never* for one an
operator's rule decided, so turning it on cannot override a rule an admin wrote, and a Valve
outage, a rate-limit or an unrecognised appid degrades to "the denylist alone decided" — which is
the default behaviour anyway. An appid the store does not recognise is **not consulted**, not
"not a game": treating it otherwise would let the opt-in lookup suppress real games, which is the
one thing it must not be able to do.

**Layer 1 is a code constant and updating it is a release.** That is the honest answer to "how
does an operator update the denylist", and it is exactly why layer 2 exists. The residual is
specific and worth stating: a **new Valve runtime the built-in list does not know about
auto-publishes to every user at once**, because every Steam user has Proton installed, so a
denylist miss on a *runtime* is not a per-user annoyance the way a miss on a game would be. The
**name-prefix** half is what mitigates it — a future `Proton 10.0` ships a new appid but keeps the
prefix — and the recovery for what gets through is one Ignore, fleet-wide, immediate.

### Ignore is a hide, never a delete

`PUT …/library/rules/{external_id}` with `rule: "ignore"` does **three** things in one
transaction, and none of them is a `DELETE`:

1. writes the `library_appid_rules` row;
2. sets `enabled = false` on the tile, if one exists;
3. revokes that tile's `granted_by='provider'` entitlements — **fleet-wide**, not just for one
   user, because an Ignore is a fleet decision and a half-revoke would leave the tile live for
   everyone who happens not to be scanned next.

**Why it is not a delete**, plainly: `apps` cascades to `user_app_favourites` and to `app_artwork`,
so deleting a junk tile **destroys every user's favourite of it and its artwork row,
irreversibly** — and the next scan re-creates a bare row anyway, because the appid is still on
disk, at which point the admin reasonably concludes the feature is broken. Delete is the obvious
reflex and the admin app editor already offers it; it is the wrong action here twice over.

Durability comes from **two independent facts, deliberately**, because either alone is a
resurrection bug: the rule row makes the reconciler skip the appid, **and** the reconciler never
modifies an existing tile, so `enabled = false` is never flipped back even if the rule row were
somehow lost.

**`allow` does *not* re-enable an existing disabled tile.** It writes the rule; the **next scan**
republishes the appid through the ladder. That asymmetry is intentional: a tile may be disabled
because an admin disabled it by hand for a reason unrelated to discovery, and an un-ignore must
not override them. `DELETE` on a rule likewise enables and disables nothing by itself — it returns
the appid to whatever layer 1 says about it, and the next scan applies the ladder afresh.

### Admin surfaces

**`{id}` must be a library provider on all four of the `/v1/admin/apps/{id}/library/*` routes,
and the refusal is `400 validation_failed` — deliberately not `404`, and emphatically not `200`.**

**Why not `200`.** Every one of these routes is keyed on
`(parent_app_id, external_source, external_id)`, and the reconciler only ever reads rules whose
parent carries `library_provider='steam'`. So a rule written against an ordinary app **stores
fine, is acknowledged, and is permanently inert** — and the operator most likely to write one is
precisely the operator already hunting for why a rule "isn't working". **A success response for a
request that can never have an effect is worse than a failure**, because it removes the one signal
that would have ended the search.

**Why not `404`.** The app is real. `404` and `400` here are **two different operator mistakes
with two different fixes**: `404` means the app id does not exist, and `400` means the app is fine
and only the marking is missing. Collapsing them would send someone hunting for a missing app that
is sitting right there. The server's message names the remedy — *set Library provider to Steam on
the app first* — so it is **actionable**, and the usual rule still applies: **clients must not
parse it**, only the `code` is contractual.

**The check reads `library_provider` and never `kind`.** §Derived tiles' promise that no server
path may branch on `apps.kind` is load-bearing here in both directions: a provider app an operator
left at `kind='game'` must pass, and an app they styled as a **Launcher** without setting a
provider must **not**. Styling is not configuration. There is a test whose negative case is
deliberately a `kind='launcher'` non-provider app, because this is exactly the check a future
reader could "simplify" in the wrong direction.

`DELETE …/rules/{external_id}` keeps collapsing "no such app" and "no such rule" into one
`404 rule not found` — distinguishing them would confirm the existence of an app id to a request
that named a rule — but a real non-provider app is the `400` above, not that `404`.

**`POST /v1/admin/library/scan` applies the same rule to its optional `app_id`**, even though the
app id rides in the body rather than the path: the mistake is identical, so the answer is identical
rather than a second vocabulary for it. It is not one of the four routes above only because it is
instance-scoped by default — an unscoped force scan names no app at all.

> **No automated gate caught this omission, and that is worth recording rather than filing as a
> near-miss.** `TestOpenAPIDrift` compares **path and method only**, so a route documented with the
> wrong status-code set is invisible to it; and the API-conformance harness validates only the
> responses it actually drives, and it drives none of these four. That is a real and known limit of
> both gates rather than a failure of either — they answer *"does this route exist in both places"*
> and *"is this response well-shaped"*, and neither answers *"is every refusal documented"*. The
> only thing that closes that gap is a reviewer reading the handler, which is what happened here.

#### `GET /v1/admin/library/status`
```json
{ "enabled": false, "storage_provider": "auto", "scan_interval_secs": 21600,
  "appdetails_lookup": false,
  "inert_reason": "library discovery is switched off",
  "scans": { "pending": 0, "claimed": 0, "reported": 0, "failed": 0 } }
```

**`inert_reason` is the reason this endpoint exists**, and it is part of the contract rather than a
nicety. Under auto-publish the only observable signal of a working scan is **tiles appearing**, so
"nothing appeared" and "nothing ran" look identical to an operator. `inert_reason` is `""` when
discovery is live, and otherwise names exactly one of: the switch is off; the interval is `0`
(which disables discovery regardless of the database flag); the instance storage provider is
`volume`; or **no app is marked as a library provider**, so the eligibility query joins against
nothing — the first-run state, reported **last** of the four because the three above it make the
provider question moot. `scan_interval_secs` is the **resolved** interval in seconds, so an
operator can see what their `6h` actually became.

**Clients must not parse `inert_reason`.** It is human-readable prose, rendered verbatim, and the
set above will grow; only its **presence** is contractual.

#### `POST /v1/admin/library/scan` — the operator's "scan now"

> *Additive amendment, admin-gated, requires sign-off. One new route; no migration, no new error
> code, and **`agent-api.md` still byte-identical**. See the force-scan amendment block above.*

Body — **entirely optional, and an empty body is the common case**:

```json
{ "app_id": "<uuid|absent>", "user_id": "<uuid|absent>" }
```

```json
// 200
{ "queued": 3, "skipped": 1, "eligible": 4, "inert_reason": "" }
```

**Why it exists: discovery's pacing is anchored to process boot, not to the operator's decision.**
The janitor's enqueue timer starts when the control plane starts, and the agent polls on its own
cadence on top of that — right for steady state and wrong for the moment an operator *changes*
something and then watches an empty library, unable to tell a working feature from a broken one.
This is a **pull** channel (that is what kept `agent-api.md` untouched), so the control plane cannot
push a scan to an agent; the only thing it can do is queue one immediately and let a **60-second**
poll pick it up.

**It bypasses the pacing and nothing else.** What it drops is the janitor's *"no successful scan for
this triple inside `QUASAR_LIBRARY_SCAN_INTERVAL`"* predicate — the rule that stops a six-hourly
sweep re-walking every home on every pass, and precisely the rule an operator who just installed a
game or just fixed a home mount is overriding on purpose. **Every gate still applies and is still
reported**: the `library_discovery_enabled` switch, `QUASAR_LIBRARY_SCAN_INTERVAL=0`, the `volume`
storage provider, and the per-home `provider='local'` filter — the last enforced inside the enqueue
itself, so a volume-backed home on an otherwise-local instance is never forced into a `pending` row
nothing could claim. *Pacing is the operator's to override; the gates are the operator's own
decisions and are not.*

**An inert instance is a `200` with `inert_reason`, deliberately not a `409` or a `400`.** It
mirrors `GET /v1/admin/library/status`, whose `inert_reason` was added for exactly this question, so
a client renders **one** code path for "here is what happened" and "here is why nothing did" — and
an admin UI that shows the reason inline needs no second branch. The server computes both surfaces'
reasons through **one shared helper**, so the wording cannot drift between the status panel and the
button; two copies of that switch would diverge the day a fourth gate appears, and the operator
staring at `queued: 0` would be reading a different story from the one the panel tells.

**The response fields, and why `eligible` is one of them:**

- **`queued`** — `pending` rows actually inserted by this call.
- **`skipped`** — triples that already had an **open** scan (`pending` or `claimed`). **Not an error
  count.** `library_scans_open_uk` is partial on those two states, so "already queued" is the
  *correct* answer to "scan now", not a failure; enqueue is `ON CONFLICT DO NOTHING`, which is what
  makes a double-press idempotent — no duplicates, no `409`, no `500`.
- **`eligible`** — `queued + skipped`: the number of `(user, library-provider app, host)` triples the
  request matched **at all**.
- **`inert_reason`** — `""` when the call could do work; otherwise the reason it could not — the
  same four instance-level reasons `GET /v1/admin/library/status` reports, in the same words (one
  shared server-side helper computes both), **plus** the route-specific *"your scope matched
  nothing"* below. The instance-level four are answered first; the `eligible: 0` reason is only
  reached once none of them applies.

**`eligible` is the non-obvious field, and it is here because two zeros mean opposite things.**
`queued: 0` with `eligible > 0` means *everything is already queued — wait for the agent to claim
it*. `queued: 0` with `eligible: 0` means *your scope matched nothing — fix the scope*. One says do
nothing, the other says do something, and **without `eligible` a client cannot tell them apart**.
That is the same ambiguity §Storage-driver limitation exists to close: under auto-publish, silence
is indistinguishable from success, so **no path through this endpoint returns a bare zero** — an
`eligible: 0` also carries an `inert_reason` naming the eligibility rule that matched nothing.

**Scoping.** `app_id` narrows to one **library-provider** app, `user_id` to one user; either, both,
or neither. A **non-provider `app_id` is `400 validation_failed`**, reusing the rule the four
`/v1/admin/apps/{id}/library/*` routes already apply and for the identical reason — a scope naming an
app the reconciler will never read is permanently inert, and the operator most likely to write one
is already hunting for why nothing happens. An `app_id` naming an app that **does not exist at all**
is `404 not_found` — the same two-mistakes-two-fixes split the sibling routes make, applied to a
body field. A malformed UUID in either field is `400`; malformed JSON is `400`; **an absent body is
not an error**, because "everything, now" is what the admin button sends.

**Order of checks, which is observable and therefore contractual: inertness is answered before the
scope is.** A request carrying a non-provider or non-existent `app_id` against an instance where
discovery is switched off (or `volume`-backed, or interval-`0`, or has no library-provider app at
all) is the **`200` with `inert_reason`**, not the `400`/`404`. That is the right order rather than an accident of the code: the instance-level
answer is the one that explains why *nothing at all* can happen, and reporting a scope defect first
would send an operator fixing an app id on a control plane that would have refused any scope. Body
shape — malformed JSON, malformed UUID — is still rejected first, because a request the server
cannot parse has no meaning to report an inert reason *for*.

**Audited as `library.scan.force`**, target type `library`, with identifiers and counts only
(`app_id`, `user_id`, `queued`, `skipped`, `eligible`) and no free text. This is an action that
makes the entire fleet walk every user's home directory on demand, which is exactly the class of
thing an operator later asks *"who did that, and when"* about. The request body is bounded at
**64 KiB**; anything larger fails the decode and is the same `400` as malformed JSON.

> **The same gate limit §Admin surfaces records above applies to this route, and is worth saying
> rather than assuming otherwise.** `TestOpenAPIDrift` compares **path and method only**, so it
> proves this route is documented and not that its status codes or its four response fields are; and
> the API-conformance harness validates only the responses it actually drives, and it does not drive
> this one. Both gates are green for this amendment and neither of them checked the part that
> matters here — which is the reviewer reading the handler against this text.

#### `GET /v1/admin/apps/{id}/library/unpublished` — "Seen, not published"
```json
{ "items": [ { "external_source": "steam", "external_id": "1493710",
               "name": "Proton Experimental", "suppressed_by": "builtin_appid",
               "users": 3, "last_seen_at": "…", "has_tile": false } ] }
```

`400 validation_failed` when `{id}` is a real app that is not a library provider (above);
`404 not_found` when it does not exist at all.

Every appid observed under this provider app that has **no enabled tile**. This read is why
observations record suppressed appids at all: a suppressed appid has no tile, so without it a game
the built-in denylist wrongly caught would be **invisible, with no surface on which an admin could
find it** — let alone `allow` it. It is a read of one table and a button, not a review queue:
nothing waits on it, and the feature is fully correct for an operator who never opens it.

`suppressed_by` is `rule_ignore`, `builtin_appid`, `builtin_prefix`, `appdetails` — or **`other`**,
and that last value is deliberate honesty rather than a gap. `other` means the ladder's first four
rungs **would** have published this appid and yet no enabled tile exists. Exactly two things reach
that state and **neither is knowable from the database**: the opt-in `appdetails` lookup suppressed
it during a scan, or an admin disabled the tile by hand. Guessing between them would be worse than
saying so, and an admin who sees `other` and writes `allow` gets the right outcome in the first
case and no change in the second — which is the correct behaviour for both.

`has_tile` distinguishes "a disabled tile exists" from "nothing was ever created" — the difference
between an Ignore that took effect and an appid nothing has published yet. Both are recoverable;
only the first has favourites and artwork to preserve. `users` is the number of distinct accounts
observed with it installed; `name` is one of the observed names (they can diverge across users on
a torn read or a localised client, and the row is a handle for an admin action, not a catalogue
entry).

#### `GET /v1/admin/apps/{id}/library/rules`
```json
{ "items": [ { "external_source": "steam", "external_id": "1493710", "rule": "ignore",
               "note": "", "created_by": "…", "created_at": "…" } ] }
```
Newest first. `created_by` is the acting admin and is `null` once that account is deleted —
deleting the operator who wrote a rule must not silently un-suppress every appid they ignored.
Same `{id}` rule as the read above: `400` for a real non-provider app, `404` for one that does not
exist.

#### `PUT /v1/admin/apps/{id}/library/rules/{external_id}`
Body `{ "rule": "ignore" | "allow", "note": "…", "external_source": "steam" }`. `note` is optional
and truncated at 512 characters; `external_source` is optional and defaults to `"steam"`, and any
other value is `400 validation_failed` — the vocabulary is provider-shaped so Heroic or RomM slot
in later, but only `steam` exists today.

```json
{ "rule": { "external_source": "steam", "external_id": "1493710", "rule": "ignore",
            "note": "", "created_by": "…", "created_at": "…" },
  "disabled": true, "revoked": 4 }
```

**Note the shape asymmetry, which is real and is recorded rather than smoothed over:** `rule` on
the *request* is the two-value string, and `rule` on the *response* is the whole stored rule
object (whose own `rule` field is that string). `disabled` says whether a tile was disabled by
this call and `revoked` counts the provider entitlements it removed — both `false`/`0` for an
`allow`, which by design changes nothing but the rule row.

**`external_id` is validated at the handler as a bare positive integer** (`^[1-9][0-9]{0,9}$`,
bounded below 2³²) **before** the database `CHECK` sees it. Both are deliberate: this table is
**admin-writable and takes an appid straight from an HTTP body**, and the value ends up in
`STEAM_STARTUP_FLAGS`, which the `quasar-steam` entrypoint word-splits with `read -r -a`. The
handler is what makes a bad value a `400` instead of a `500`; the `CHECK` is the durable guard and
the only one of the two that survives an admin editing the row directly later.

#### `DELETE /v1/admin/apps/{id}/library/rules/{external_id}`
`204` on success, `404 not_found` when no such rule exists. Takes the same optional
`?external_source=` (default `steam`); unlike the `PUT` it does not reject an unknown source with
a `400` — an unknown source simply matches no rule and is therefore the `404` it already is.

### The operator setting, and the two env knobs beside it

**`library_discovery_enabled`** on `GET`/`PATCH /v1/admin/settings` (default **`false`** —
ship-dark, the posture artwork already holds) is the **master switch and the only switch**. Under
operator decision 1 **auto-publish is the behaviour, not a mode**: there is no review queue, so a
second "publish" toggle would only ever select between publishing and a path that was never built,
and a boolean whose `false` branch is unimplemented is precisely the half-built second path that
decision exists to avoid.

It is read **per pass and per request, never cached at boot**. A setting an admin flips in the UI
must take effect without a restart; the artwork provider is the recorded precedent for what
happens otherwise.

**Enabling it now runs a pass promptly, and that is contractual behaviour of `PATCH
/v1/admin/settings` even though the endpoint's shape does not change.** A `PATCH` that moves
`library_discovery_enabled` **false→true** causes the janitor to run a pass without waiting for its
next six-hourly tick. **true→true and false→false do not** — an admin re-saving the settings form to
change the registration mode must not re-walk every home on the fleet, and there is nothing to run
for a disable. Reading the switch per pass was never enough on its own: the pass that read `false`
had already scheduled its successor hours out, so "no restart needed" and "takes effect now" were
two different claims and only the first was true.

**It is the transition that is observed, not the value.** The previous value is read `FOR UPDATE` in
the **same transaction** as the write, so two admins saving concurrently cannot both see false→true
and both trigger a pass. This is stated here because a client author reading only the endpoint's
shape would reasonably assume that enabling discovery is inert until the next tick, and then build
a UI that tells the operator to come back in six hours — which is exactly the expectation this
closes. The nudge changes **when** a pass runs and nothing about what it does: the same code path,
the same re-read of this switch, the same inert reasons.

Two environment knobs sit beside it and **neither is a second switch**:

- **`QUASAR_LIBRARY_SCAN_INTERVAL`** (default `6h`) — how often a scan is enqueued per
  (user, provider app, host). **`0` disables discovery entirely regardless of the database flag**,
  so an operator can guarantee a control plane makes no scan and no third-party call without
  needing database access. Negative or unparseable **fails startup**, rather than silently
  disabling: "I set the knob and nothing was ever scanned" is not a failure anyone finds on their
  own.
- **`QUASAR_STEAM_APPDETAILS_LOOKUP`** (default **off**) — rung 5 of the ladder. **It discloses to
  a third party exactly which Steam appids this instance has installed**, which is the same privacy
  class as artwork hotlinking, and that was rejected for the same reason. Off by default, an
  operator's decision, never a default; bounded per scan; and contained so that enabling it cannot
  override a rule an admin wrote.

With the switch off, **nothing happens**: no scan rows, no agent requests, no third-party calls.

### Storage-driver limitation, stated up front

A `volume`-provider home has **no host path an agent can walk**, so on an instance whose storage
provider is `volume` discovery **enqueues nothing**. The limitation is the same one `bytes_used`
already has, and it is not new information — what is new is that under auto-publish it is
**invisible without help**, which is why surfacing it is part of this contract and not a UI
nicety: `GET /v1/admin/library/status` reports it as `inert_reason`, and the same reason is logged
once for an operator reading logs rather than the admin UI. The scheduler also filters per home,
not only per instance, so a volume-backed home on an otherwise-local instance is never enqueued
into a pending row nothing could ever claim.

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
  "codecs": ["h264", "h265"],
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
- **`codecs`** *(NEW, wizard-v2 §S5, additive)* — the **wire** codec set this host last reported
  it can actually produce (`schema.md` `hosts.codecs`, written from the agent's
  `agent-api.md` `capacity.codecs`): an array subset of `["h264","h265","av1"]`, in the order
  the agent reported it (no ordering meaning — codec *preference* is a launch-profile property,
  not a host one). **Read-only and `GET`-only**: `PATCH` neither accepts nor returns it, because
  it is an agent observation, not an override. `null` when the host has never reported the
  field — a pre-multi-codec agent. `null` is **not** the same statement as `["h264"]` even
  though the launch resolver treats both as H.264-capable-only: "never reported" means the
  control plane does not know, and an operator surface must say so rather than assert an
  H.264-only host. This is why the field is not normalised server-side.
  It lives on this endpoint rather than on the `GET /v1/hosts` host body deliberately: this is
  already the admin-only host-detail surface, so the disclosure stays as narrow as the fact
  needs (the host body is read by non-admin/library surfaces that have no use for it).
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

**Adaptation knob group (2026-08-16).** `abr_mode` (enum `off|protective|smooth`,
default `smooth`) supersedes the deprecated `abr_enabled` bool, which remains in the
catalog for compatibility (`false` ⇒ `off`; `true` ⇒ defer to `abr_mode`). The
`abr_ladder*` family exposes the SPT-08 ladder: the encoder-speed-bias rung's hysteresis,
and the external-resolution rung's comfort-bitrate exponent, engage/recover fractions and
dwells, minimum step interval and floor height. All are live-class. **Cross-knob
validation:** `PATCH /v1/admin/hosts/{id}/settings` returns `400 validation_failed` when
the RESOLVED configuration would collapse the resolution rung's hysteresis band
(`abr_ladder_res_recover_frac` must exceed `abr_ladder_res_engage_frac` by ≥ 0.05); the
message names **both** keys, because a patch that sets only one can still break the pair.

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

`cover_url` is the **TILE** crop for the **2:3 portrait** library-tile frame; `hero_url` is the
**HERO** crop, a much wider banner for the detail/hero panels. They are **different source
assets**. A 2:3 tile stretched into a ~3:1 hero reads as a blown-up thumbnail, which is the
specific failure this split exists to avoid. Either may be null independently; a client falls
back `hero_url` → `cover_url` → the gradient tile.

**The tile was 16:10 until issue #385.** The provider's most common grid dimension is portrait
box art (600×900), which is far more recognisable per pixel, so the tile frame and the provider
query both moved to portrait. This is an **operator-directed deviation** from the signed-off
16:10 mockup, and `design_handoff_quasar/` was updated alongside it. **The hero crop is
unchanged and still wide** — the two-crop split survives the change intact.

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

`none` is a first-class outcome. **`apps.kind IN ('desktop','launcher')`** is checked **before**
any provider call, so neither is ever queried at all: a games database will not have Blender or
Firefox, and asking costs a request and leaks the app name for a guaranteed miss. **`'launcher'`
joined the set in Steam library discovery Phase 3, and it is a deliberate call rather than an
oversight** — the fuzzy matcher's 7-for-7 failure list on the live catalogue is **headed by
"Steam (Dev)" matching "Steam Dev Days"**, so the Steam client is precisely the app a games
database mis-matches *confidently*. Short-circuiting renders the gradient tile, which the mockups
treat as a deliberate look; an operator who wants a Steam logo uploads one through the existing
manual path. The condition is a **set membership, not a chain of `if`s**, so a fourth `kind` has
to answer the question here rather than reintroduce it elsewhere.

> **This short-circuit is the ONLY server-side read of `apps.kind`, and it stays this narrow.**
> `kind` is presentation-only by contract; this is a presentation decision *about* presentation
> (which tile art to render), not `kind` becoming an input to a decision. Nothing in scheduling,
> admission, placement, profile/codec resolution, discovery or the agent wire may branch on it.

An unmatched *game* records `none` too, so the next sweep does not re-ask. A provider **error**,
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
- `{"rematch": true, "force": true}` — the same, but past a `locked` record. `force` qualifies
  **only** `rematch`; the other two intents set `locked` themselves and were never blocked by it.

Any override sets `locked: true`, and **the automatic sweep never touches a locked record** — a
correction must not be silently re-broken.

**`rematch` now honours `locked` too *(#385 amendment)*.** It previously cleared the record
unconditionally, so an explicit rematch silently discarded an admin's correction — the one thing
the flag exists to prevent. A `rematch` against a locked record is now `409 conflict`; `force:
true` is the deliberate override.

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

### `POST /v1/admin/artwork/reresolve` *(admin)* — bulk re-resolve *(#385)*

Catalogue-wide "fetch it all again". Body `{"force": false}` (optional; absent means `false`).
Returns `{ "total", "resolved", "skipped_locked", "failed" }`.

It exists because a change to the **provider query** cannot reach apps that already have a
record — automatic resolution returns early for those, by design — so the portrait tile change
above would leave existing apps on their old landscape art forever. **Not automatic and not a
migration:** the bytes come from a third party, and spending a deployment's third-party budget
without anyone asking would be the wrong default. It is an explicit operator action.

- **`locked` records are skipped and counted**, never overwritten. `{"force": true}` overrides
  that, and is the only way to replace an operator's manual correction in bulk.
- Per-app failures are counted, logged and stepped over — one unmatchable app never stalls the
  queue. A failed app is left with **no** record, which is exactly the background sweep's own
  work-queue predicate, so it is picked up again on the next tick rather than lost.
- `409 conflict` when no provider is configured, resolved once up front rather than N times.
- Cached blobs the re-resolve orphans are reclaimed by the existing boot-time orphan sweep; this
  route adds no new reclamation path.
- Audited as `app.artwork.reresolve` with counts only — no app names or ids in the payload.

### Authorization

Every route except `GET /v1/artwork/{asset}` is `RequireAuth → RequireAdmin`, enforced by the
middleware at route registration. Hiding the Artwork panel from a non-admin UI is **not** the
access control (invariant #6): a valid non-admin token is `403` on all six admin routes, and a
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

## Background jobs (jobs framework, 2026-08-12)

> *Additive amendment. Seven new routes (five admin, two agent-internal) over two new tables,
> `jobs` and `job_runs` (`schema.md`, migration `0066`). **Nothing existing changes**: no shape,
> status code, or endpoint above is touched, and `agent-api.md` is **byte-identical** — the agent
> side is an HTTP pull channel, the third in this contract to be chosen over a new WebSocket
> message and for the same reason (§The agent pull channel).*

Before this, background work in Quasar was eighteen independent goroutines and threads on
hand-rolled tickers with **no record anywhere of a job having run** — "when did the artwork
grabber last run" was unanswerable without reading container logs. The framework's single
structural idea is that every property an operator wants of a background job (last run,
duration, result, next run, is it running, may it run at 3pm) is a property of **a record of a
run**, so a run is a durable row and a `pending` row **is** the next run. There is deliberately
no `next_run_at` column: a denormalized next-run timestamp is exactly the derived state that
drifts out of sync with the thing it describes.

**Ownership is split, and the split is load-bearing.** A job's *identity* (`name`,
`description`, `plane`, `scope`, `managed`) is **code-owned** and reconciled at every boot; its
*schedule* (`enabled`, kind, `interval_secs`, the window fields, `timezone`, `history_limit`) is
**admin-owned** and is never overwritten by that reconcile. A boot that clobbered an operator's
`02:00–06:00` window because a developer edited a literal would make the whole surface
untrustworthy.

### Schedule and window semantics

- **`interval` is measured from the END of the previous run**, not from its start and not as a
  wall-clock cadence — the timer-reset-after-pass idiom the artwork sweeper and library janitor
  already used, which makes overlap structurally impossible rather than merely unlikely.
  Minimum 60 s (a `CHECK` **and** a handler guard).
- **`event`** never fires on a clock; a run row is created by an explicit trigger. This
  represents event-driven work **without pretending it is periodic**, while still showing
  last-run and result. **`manual`** only ever runs from an admin trigger.
- **The window snap is FORWARD-ONLY.** When the interval next fires, if that instant falls
  outside the permitted window, the pending run's `scheduled_for` is snapped **forward** to the
  next window open. So `interval_secs=86400` + `02:00–06:00` means "once a day, in the small
  hours", and `interval_secs=3600` + `02:00–06:00` means "hourly, but only four of them land".
- **`window_days` constrains the day the window OPENS** (0 = Sunday … 6 = Saturday; empty =
  every day). A wrapping window on `{5}` therefore runs Friday 22:00 → Saturday 04:00.
- **A wrapping window is legal** (`22:00 → 04:00`). `window_start == window_end` is rejected on
  the write path (`422`): an empty window would never open. A hand-edited row where they are
  equal is interpreted as **24 hours** — of the two failure modes, running the job beats
  silently wedging it forever.
- **The window governs STARTING a run, never stopping one.** A run in flight when the window
  closes is not killed; killing a half-finished dedupe pass or a half-built template is worse
  than overrunning by minutes. A job that must yield is expected to be interruptible **by its
  own design**, through its own abort primitive.
- **Windows are evaluated in `timezone` (IANA); intervals are not.** An interval is a duration
  in absolute time and has no opinion about the clock on the wall, so a DST transition can shift
  *when* a windowed run lands but can never lengthen or shorten an interval. An unknown zone is
  `422`, never silently coerced to UTC.
- **Env overrides stay authoritative.** Where an environment variable already documents itself
  as an *override, not a default* (`QUASAR_ARTWORK_SWEEP_INTERVAL`,
  `QUASAR_LIBRARY_SCAN_INTERVAL`), that does not change: the resolved schedule reports
  `locked: true` + `locked_by: "<VAR>"`, and a `PATCH` of the interval is refused with
  `409 schedule_locked` rather than silently accepted and overruled. This is the same treatment
  `GET /v1/admin/library/status` already gives with `interval_overridden_by_env`, and it exists
  to prevent the "which source is in force" confusion.

**Single-flight is a database invariant, not a convention.** At most one open (`pending` or
`running`) run per `(job, target)`, enforced by a partial unique index (`schema.md`
`job_runs_open_per_target`). "Two dispatchers double-fired" and "Run now while it is already
queued" are therefore impossible at the storage layer.

### `GET /v1/admin/jobs` (admin)

`?cursor=&limit=` (default 50, 1–500; an unparseable value falls back to the default).
`{ "items": [...], "next_cursor": "<opaque|null>" }`, the same envelope as the admin session
list. Three item shapes, discriminated by `managed` and `scope`:

- **managed, `scope: "instance"`** — top-level `running`, `next_run_at`, `last_run`,
  `consecutive_failures`.
- **managed, `scope: "host"`** — `targets[]` **instead**, one entry per host with that host's own
  `running` / `next_run_at` / `last_run`. "Last run" is not a single fact about a host-scoped
  job; it is a fact about the job **on a host**.
- **unmanaged** — neither, plus `unmanaged_note`.

```json
{ "id": "artwork.sweep", "name": "Artwork grabber",
  "description": "Resolves cover and hero art for apps that have no artwork record.",
  "plane": "control", "scope": "instance", "managed": true, "enabled": true,
  "schedule": { "kind": "interval", "interval_secs": 900, "window_start": null,
                "window_end": null, "window_days": [], "timezone": "UTC",
                "locked": true, "locked_by": "QUASAR_ARTWORK_SWEEP_INTERVAL" },
  "running": false, "next_run_at": "2026-08-12T14:15:03Z",
  "last_run": { "id": "8b0d…", "host_id": null, "state": "succeeded", "trigger": "schedule",
                "started_at": "2026-08-12T14:00:03Z", "finished_at": "2026-08-12T14:00:04Z",
                "duration_ms": 1188,
                "summary": {"apps_considered": 412, "artwork_resolved": 3}, "error": null },
  "consecutive_failures": 0, "history_limit": 50 }
```

A **run-derived field is omitted rather than sent as null** when it does not apply or is not
known; a client must read absent as null. `summary` is the runner's own opaque blob (`{}`, never
null, when a run reported nothing) — the framework never interprets it. `consecutive_failures`
is **derived from history**, not stored, so it cannot drift from the rows it describes.

**Unmanaged jobs are listed on purpose.** An unmanaged row is background work that exists in
code on a hard-coded timer, with no schedule, no history and no Run-now, shown so an operator can
*see* it; `unmanaged_note` names the file that hard-codes it. Omitting them would reproduce the
exact problem the page exists to fix — six visible rows next to twelve invisible goroutines — and
the list of unmanaged rows doubles as the adoption backlog.

### `GET /v1/admin/jobs/{job_id}` (admin)

The same item shape, alone. `404 not_found` for an unknown id. Note that `{job_id}` is **not a
UUID**: it is the code-owned dotted identifier (`artwork.sweep`), which is also the `jobs`
primary key and the id in every log line.

### `PATCH /v1/admin/jobs/{job_id}` (admin)

All-optional body — `enabled`, `interval_secs`, `window_start`, `window_end`, `window_days`,
`timezone`, `history_limit` — so "absent" is distinguishable from "zero". `window_start` and
`window_end` move **together**: both strings to set a window, both `null` to clear it. Returns
the updated job. Every accepted `PATCH` writes an `admin_activity` row (`job.update`, details
`{"keys": [...]}`).

| condition | status | code |
|---|---|---|
| malformed JSON, an **unknown key**, or a value of the wrong JSON type | `400` | `validation_failed` |
| `interval_secs` < 60, `interval_secs` on a non-interval job, `window_start == window_end`, a `window_days` value outside 0–6, an unknown IANA zone, `history_limit` outside 1–500 | `422` | `validation_failed` |
| the job is `managed: false` | `409` | `job_unmanaged` |
| an env override is authoritative over the interval | `409` | `schedule_locked` |
| no such job | `404` | `not_found` |

**An unknown key is rejected rather than ignored**, deliberately: a typo'd field name accepted
with a `200` is precisely the "did my edit take effect" confusion this framework exists to end.

### `POST /v1/admin/jobs/{job_id}/run` (admin)

Body `{ "host_id": "…" }` — **required** for a host-scoped job, **not allowed** for an
instance-scoped one (either mistake is `400 validation_failed`); an empty body is valid for the
instance case. `202 Accepted`:

```json
{ "run_id": "…", "state": "pending", "scheduled_for": "…",
  "eta_note": "queued; the host claims due jobs about every 60 s" }
```

The run is **queued, not executed inline**. `eta_note` is a human string the UI renders
**verbatim** so Run now does not read as broken for the minute before an agent-plane host polls;
**clients must not parse it**. If a `pending` run already exists it is **pulled forward**, not
duplicated.

**A manual run bypasses the window and never bypasses the job's own safety gates** — the
framework requests, the job decides; a runner that refuses reports `deferred`, which is an
outcome, not an error. Errors: `409 job_already_running`; `409 job_disabled` (**a disabled job
never runs, not even manually** — disabling is the operator's kill switch and must mean it);
`409 job_unmanaged`; `404 not_found`. Audited as `job.run`.

### `GET /v1/admin/jobs/{job_id}/runs` (admin)

`?host_id=&cursor=&limit=` — bounded history, newest first, `{items, next_cursor}`, each item the
`last_run` shape above. `404` for an unknown job.

### The agent pull channel

Two node-secret routes (`Authorization: Bearer <node_secret>` + `X-Quasar-Node`, verified
constant-time against `hosts.node_secret_hash`), identical in scheme to `/v1/agent/storage/*` and
`/v1/agent/library/*`. **This is the third time this contract has weighed a new `AgentMsg` and
chosen a pull channel instead**, and the property that decides it is the same one: a claim is a
**database row**, so an agent reconnect has nothing to correlate and `agent-api.md` stays
byte-identical. An implementer who concludes a new WebSocket message is needed here is
escalating, not patching.

**The agent asks; it is never told.** Nothing pushes a job at a host: the host polls when it is
ready, claims what is due **for itself**, and reports what its own runner decided.

**`GET /v1/agent/jobs/pending`** — claims this host's due agent-plane runs with
`FOR UPDATE … SKIP LOCKED` (so two polls take **disjoint** sets), stamps `claimed_at` +
`state='running'`, and returns:

```json
{ "runs": [ { "run_id": "1f4a…", "job_id": "template.warmup",
              "params": {"image_id": "steam"}, "deadline_secs": 3600 } ] }
```

Capped at **5 runs per poll**: a host returning from an outage with a dozen jobs due must not
start all of them at once, and the rest are still `pending` because the work is a durable row.
`params` is the opaque per-job blob the control plane stored when it materialized the run
(`{}`, never null); the framework never interprets it. `deadline_secs` is sent so the agent can
bound its own execution rather than discover the abort by racing it — a claim nobody reports is
reaped to `aborted` and re-materialized. An empty list is the steady state, **and is also the
answer when the jobs master switch is off**: the switch is re-read here as well as in the
dispatcher, because the dispatcher is what stops rows being *created* and this is what stops an
already-materialized row being *handed out*.

**`POST /v1/agent/jobs/report`** — `{ "run_id", "state", "summary", "error" }`, `200 {"ok": true}`.

- `state ∈ {succeeded, failed, deferred, skipped}`. **`aborted` is not reportable**: it is the
  reaper's verdict on a host that said nothing, and a host claiming it would be describing a
  decision it does not get to make. Sending it — or any other value — is `400 validation_failed`.
- **The 401-no-oracle rule.** A bad secret, an unknown `run_id`, and a run belonging to **another
  host** are one indistinguishable `401`. Ownership is checked **before anything is written**, and
  the failure is `401` rather than `404` so these routes never become an oracle for run ids. The
  real reason is logged, never returned.
- **The idempotent-report rule.** A report for a run that is **already terminal** is a **no-op
  `200`**, so an agent retrying after a network blip is safe. A `409` the agent cannot act on
  would turn a run that actually succeeded into a permanent error in an operator's face.
- A `deferred` report is a normal outcome (the runner's own gate refused) and the dispatcher
  materializes the retry on a **persisted** backoff ladder — persisted, so a back-off decision
  survives an agent reconnect and a control-plane restart. `409` covers a run that moved under
  the caller or a `summary` that blew the 4096-byte ceiling: **a summary that is too large fails
  the report, never the run.**

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
