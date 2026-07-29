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

> **Amendment — P3-01 (host-lifecycle + multi-host scheduling), additive, requires sign-off.**
> **No DDL.** Defines the **host status state machine** (the transitions over the existing
> `hosts.status` `{online,offline,draining}` column — the value `draining` was provisioned in `0001`
> for exactly this), and a new `sessions.state_detail` convention **`host_lost`** the offline reaper
> stamps when a host disconnect fails its sessions (a free-text value over the existing column,
> exactly like P2-02's `swapping` — the `state` CHECK set is untouched). No new column, type,
> constraint, or migration. See `docs/completed/phase3/P3-01-contract-host-lifecycle.md`.

> **Amendment — P9-01 (native-client prelude), additive, requires sign-off.** Widens the
> `session_metrics.source` `CHECK` to `('agent','browser','native')` so the first-party
> native client's per-session telemetry (`control-api.md` `POST /v1/sessions/{id}/stats`
> with `client: "native"`) persists as a distinct reporter — via **migration 0014**, a plain
> `CHECK` swap (the enum convention below; the prior values are unchanged). This is the
> **only** schema change for Phase 9's contract prelude: the version-handshake fields are
> login req/resp only (no storage), and the native capability report rides the existing
> `user_devices.capabilities` JSONB (no DDL). No new table, column, or type; no other
> constraint changes. The migration itself **lands in P9-07** (the native producer), not the
> contract ticket. See `docs/phase9/P9-01-contract-prelude.md`.

> **Amendment — ST-01 (Observability v2 — session trace), additive, requires sign-off.** Adds two
> new append/one-row-per-session tables via **migration 0016**: `session_trace_events` (discrete
> per-session markers — ABR retarget, source swap, encoder drop, webrtc state, playout change,
> freeze, visibility — `source ∈ {agent, browser}`) and `session_trace_clock` (one optional row per
> session carrying `client_offset_ms` + `uncertainty_ms`; **absence of the row means unmeasured,
> never offset 0**). It complements the periodic `session_metrics` samples (migration 0003): the
> diagnostic bundle reads `session_metrics` JSONB **joined with** these events — **no new samples
> table, no second sample write path**. Plain Postgres only (no `CREATE EXTENSION`, no
> `create_hypertable`); bounded by the same retention model as `session_metrics` (rolling per-session
> window + terminal prune; FK `ON DELETE CASCADE` reaps on session delete). **Purely additive** — no
> existing table, column, type, constraint, or the session state machine changes. Telemetry is
> observability, never access control and never a session-state authority. See
> `docs/session-trace/contract-amendment.md` §A and `docs/session-trace/trace-format.md`.

> **Amendment — SPT-05 (Stream Perf Tuning Phase C — encoder certification), additive, requires
> sign-off.** Adds one new table via **migration 0018**: `host_encoder_certification` — the measured
> sustainable encode envelope per `(host, GPU, encoder, profile, bench-bitrate)` (`encode_ms`
> percentiles, `output_fps`, `drop_rate`, `live_write_stable`, and a `verdict ∈ {ok, capped, unsafe}`)
> so the scheduler can avoid default-starting a profile a host cannot hold in real time (e.g. Renoir
> `1080p60` → `unsafe`). It is **scheduling input, not telemetry**: **upsert-latest** (one current
> verdict per configuration, `UNIQUE (host_id, gpu_index, encoder, profile_id, bitrate_kbps)`), **not**
> append-only, and **not** on the `session_metrics` retention prune — a durable capability fact that
> survives across sessions. **Purely additive** — no existing table, column, type, constraint, or the
> session state machine changes; never access control, never a session-state authority. Two related
> pieces need **no schema change**: SPT-07 consumes the existing `user_devices.capabilities` JSONB as a
> session envelope, and `abr_mode` (`QUASAR_ABR_MODE`) is node-agent config, not a schema column. Also
> records the pre-existing **migration 0017** (`stream_profile_policy.updated_by → ON DELETE SET NULL`,
> the #154 admin-delete follow-up). See `docs/stream-perf/contract-amendment.md` §A.

> **Amendment — LP-SEC-01 (W1 security wave — invites + device binding), additive, requires
> sign-off.** Adds, via **migration 0020**: (0) one new **singleton** table `instance_settings`
> holding the global `registration_mode` (`closed` **default** | `invite_only` | `open`) — the
> invitation system is **off by default** and is turned on by an admin at runtime (persisted,
> UI-settable via `control-api.md` `PATCH /v1/admin/settings`; the `REGISTRATION_MODE` env only
> **seeds** the row on first boot, thereafter the persisted value is authoritative); (1) one new
> table `invites` — admin-minted, single-or-multi-use redemption codes stored **hashed**
> (SHA-256 hex, like `auth_tokens`), never plaintext-at-rest, delivered as **magic one-time
> links**; (2) two additive columns on `user_devices` — `name TEXT NULL` (user-set display
> label) and `trusted BOOLEAN NOT NULL DEFAULT false` (advisory trust posture, **not** an
> authorization input in W1); (3) one additive column `auth_tokens.device_id UUID NULL →
> user_devices(id) ON DELETE SET NULL` — the **token↔device binding** that makes device
> revocation real (a token minted before the migration, or a login that declares no
> `device_key`, keeps `device_id = NULL`; backfill NULL, stamped at login going forward); (4)
> one optional additive column `sessions.device_id UUID NULL → user_devices(id) ON DELETE SET
> NULL` so a device's **live** session can be shown/ended (reuses P2 teardown, **no agent-wire
> change**). **Purely additive** — no existing table, column, type, constraint, or the session
> state machine changes; the frozen `0001`–`0019` migrations are untouched. Invites are
> **account provisioning, never session authority**; device rows are **owner-scoped**, never an
> access-control flag (enforcement is the token binding + the server role gate + the persisted
> `registration_mode`, never a UI field). See `docs/w1-security/LP-SEC-01-contract.md`.

> **Amendment — multi-codec (HEVC/AV1), additive, requires sign-off (signed off 2026-07-25).** Adds
> a codec dimension to four places, all ship-dark (zero behaviour change until an admin flips a
> profile's codec status to launchable). Via **migration 0031** (`0031_multi_codec`): (1) one new
> column **`sessions.codec TEXT NOT NULL DEFAULT 'h264' CHECK (codec IN ('h264','h265','av1'))`** (the
> wire codec resolved server-side at launch and sent to the agent in `session_assign.stream.codec`,
> `agent-api.md`); (2) one new column **`stream_profiles.codecs JSONB NULL`** (a per-profile ordered
> codec-preference list `[{codec, status}]` in the **catalog** vocabulary `h264|hevc|av1`; NULL means
> the in-code default of h264-launchable, hevc/av1-future, so existing rows need no backfill; the
> admin `/v1/admin/stream-profiles` write path materialises it when an operator enables a codec); (3)
> one new column **`hosts.codecs JSONB NULL`** (the last-reported **wire** codec set the host's active
> encoder path can produce, from the `capacity` report; NULL means the control plane assumes
> `['h264']`, so old agents keep working). Via **migration 0032** (`0032_codec_scoped_history`): (4)
> one new column **`user_device_profile_history.codec TEXT NOT NULL DEFAULT '' CHECK (codec IN
> ('','h264','h265','av1'))`** with its uniqueness key widened from `(user_id, device_key, profile_id)`
> to `(user_id, device_key, profile_id, codec)`, making a decode-failure verdict codec-scoped (`''` =
> a profile-level verdict, the meaning of every pre-0032 row). Both migrations are **purely additive**
> (new defaulted columns, one widened unique key over a new column); no existing column, type,
> constraint, or the session state machine changes; the frozen `0001` migration is untouched.
> `stream_profiles` and `user_device_profile_history` have no dedicated prose section in this
> contract (the profile catalog is in-code and the history table is AS10-08 detail), so only their
> migration-ledger lines below carry the amendment; the `sessions` and `hosts` column rows are added
> to their tables. The `h264_profile` machinery is unchanged and applies to the `h264` codec only.
> §3.4 semantics of the codec-scoped history are documented at the migration-ledger line for 0032.
> See `docs/design/plans/2026-07-22-multi-codec-hevc-av1-spec.md` §3/§3.4.

> **Amendment — UI-P1 (app classification + per-user favourites), additive, signed off
> 2026-07-27.** Via **migration 0034** (`0034_app_kind_favourites`): (1) one new column
> **`apps.kind TEXT NOT NULL DEFAULT 'game' CHECK (kind IN ('game','desktop'))`**
> *(the `CHECK` was widened to add `'launcher'` by Steam library discovery Phase 3, migration 0044
> — see that amendment and the `apps.kind` row; this line records what 0034 shipped)* — a
> **presentation-only** library classification (`control-api.md` `AppListItem.kind`). Nothing in
> scheduling, admission, profile/codec resolution, or the agent wire reads it; the default makes
> every existing row valid with **no backfill**. (2) one new table **`user_app_favourites`** — a
> per-(user, app) join carrying no payload beyond `created_at`, whose composite primary key
> `(user_id, app_id)` is both the uniqueness constraint and the idempotency key for
> `PUT /v1/me/favourites/{app_id}`. **Purely additive** — a new defaulted column and a new table;
> no existing table, column, type, constraint, or the session state machine changes. A favourite
> is **presentation state, never access control and never a session authority**: it does not
> affect what a user may launch, and `control-api.md`'s server-side role/ownership gates are
> untouched. The breaking half of UI-P1 (`GET /v1/apps` becoming `RequireAuth`) is an API-surface
> change with **no schema component** — see `control-api.md`. `agent-api.md`, `signaling.md`,
> `input.md`, `native-client.md` are unchanged. See
> `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 1".

> **Amendment — UI-P3 (runtime presets), additive, signed off 2026-07-27.** Via **migration
> 0035** (`0035_runtime_presets`): (1) one new table **`runtime_presets`** — a reusable container
> configuration (image, launch arguments, environment, mounts, and the managed-home storage
> defaults) that many apps inherit instead of repeating; (2) one new nullable column
> **`apps.runtime_preset_id UUID NULL REFERENCES runtime_presets(id) ON DELETE RESTRICT`**.
> **`NULL` means the app carries everything itself, which is exactly the pre-UI-P3 behaviour**, so
> every existing row is unchanged and **no backfill** is needed. **Purely additive** — a new table
> and a new nullable column; no existing table, column, type, constraint, default, or the session
> state machine changes.
> **The name is deliberate and not interchangeable.** A *runtime preset* configures **what
> container runs**. UI-P4's *launch profiles* (an ordered chain of *stream profiles*) configure
> **how the stream is encoded**. Three distinct nouns; blurring them in a column name, an API
> field or UI copy is the specific mistake this naming exists to avoid.
> **The preset is merged into the app SERVER-SIDE AT LAUNCH, never flattened on save** — that is
> what makes editing a preset reach every app already using it. The merge rules (env: app wins on
> a conflicting key; mounts: appended, preset first, **no dedupe**; args: appended, preset first;
> image: app overrides, blank inherits; managed_home/home_container_path: preset default, app may
> override) live in `control-api.md` §Runtime presets and are implemented in the control plane's
> launch-time app resolution. **`agent-api.md` is unchanged**: the agent still receives one
> opaque, already-flattened `app` object, and an app with no preset dispatches a `runtime_spec`
> **byte-identical** to before. `signaling.md`, `input.md`, `native-client.md` are unchanged. See
> `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 3" and
> `docs/design/plans/2026-07-27-admin-mockup-implementation-notes.md` §12.

> **Amendment — UI-P4 (stream profiles + launch profiles). NOT purely additive. Requires Opus +
> explicit human sign-off.** Via **migration 0036** (`0036_launch_profiles`, the *expand* half of an
> expand/contract pair). The model splits one object into two: a **stream profile** is now one
> encode **rung** (one codec at one resolution, frame rate and bitrate), and a **launch profile** is
> an **ordered list of rungs**. A user picks a launch profile; the launch walks its rungs and takes
> the first the placed host can encode and the client can decode.
> **(1)** Two new tables, **`launch_profiles`** and **`launch_profile_rungs`** (ordered, see their
> sections below).
> **(2)** One new column **`stream_profiles.codec TEXT NULL CHECK (codec IN ('h264','hevc','av1'))`**
> — the catalog vocabulary, matching `stream_profiles.codecs`, not the wire `h265`. NULL on every
> pre-0036 row, non-NULL on every rung row created by the fan-out.
> **(3)** One new column **`sessions.stream_profile_id TEXT NULL REFERENCES stream_profiles(id)`** —
> the **rung** the launch resolved to. `sessions.profile_id` is unchanged and continues to hold the
> **user's pick**, which is now a launch profile id.
> **(4) Three foreign keys are repointed** from `stream_profiles(id)` to `launch_profiles(id)`:
> `stream_profile_policy.global_default_profile_id`, `apps.default_profile_id`, and
> **`user_profile_preferences.default_profile_id`**. The third is easy to miss and fails in the
> worst possible way: it has 0 rows, so the migration passes and the tests stay green, and then the
> first user preference set against a genuinely new launch profile is a 500 from an FK violation.
> All three are repointed together.
> **(5) `apps.profile_policy` loses the value `custom`**; the `CHECK` narrows to
> `('inherit','prefer','force')`. **The up migration first asserts that no app is using it and
> FAILS, naming the offending apps, if any is.** `custom` cannot be converted behaviour-neutrally: a
> `custom` app resolves no profile and lands on the legacy tier path, where its effective settings
> are `min(tier from the user's probe, the app defaults)` and *not* simply the app defaults, so no
> automatic conversion reproduces it. A refused deploy is the correct outcome; the operator converts
> those apps deliberately.
> **(6) Ids are preserved as LAUNCH PROFILE ids.** `1080p60` stays `1080p60` and becomes a launch
> profile; rungs get new ids `<launch-profile-id>-<codec>`. This keeps `sessions.profile_id` (676
> rows on Tower), `user_device_profile_history.profile_id` (147) and
> `host_encoder_certification.profile_id` (3) — all **un-FK'd `TEXT`**, none of which would have
> failed loudly — pointing at an object that still exists and still means what it meant.
> **(7) `user_device_profile_history` is not migrated.** Its rows keep their launch-profile-shaped
> `profile_id`; a pre-0036 row with a non-empty `codec` bans every rung of that launch profile using
> that codec, which is exactly its old meaning. Rows written from UI-P4 onward key decode-side
> failures by **rung id**. No DDL change to that table.
> **What 0036 deliberately does NOT do**, because it is the *expand* half: it does **not** drop
> `stream_profiles.codecs`, does **not** delete the legacy `stream_profiles` rows, does **not** make
> `stream_profiles.codec` NOT NULL, and does **not** give
> `stream_profile_policy.global_default_profile_id` a value (it is NULL today, and setting it would
> change the effective resolution of every `inherit` app for every user in one invisible step). A
> separate **contract migration**, after a Tower soak, does the first three. The split is what lets
> a bad *code* deploy on top of a good migration be recovered in one step, and what lets the
> migration rehearsal diff before and after **in one database**.
> **The down migration is a snapshot restore, not a computed collapse.** The fan-out is lossy in two
> directions (a single h264 rung cannot be distinguished from "`codecs` was NULL" versus "the
> default list was stored explicitly", and `future`/`unsupported` entries leave no trace in the
> rungs), so `up` first copies the affected tables into `_backup_0036_*` tables and `down` restores
> them verbatim. Two honesty clauses live in the migration header: **admin writes made after `up`
> are lost by `down`**, and **launch profiles created after `up` are dropped with a `NOTICE`**
> rather than lossily collapsed. Proof obligation: `pg_dump --data-only` of the affected tables
> before `up` and after `down` must be byte-identical, run against a **Tower snapshot**, not an
> empty database. Behaviour neutrality is a **separate** test from that round trip.
> **`agent-api.md` is unchanged** — the agent receives a resolved `stream` block and a `codec` and
> has never known what a profile is. `signaling.md`, `input.md`, `native-client.md` are unchanged.
> See `docs/design/plans/2026-07-28-phase4-profile-restructure-respec.md` and `control-api.md`
> §Stream profiles and launch profiles.

> **Amendment — UI-P5 (per-app launchable launch profiles), additive, requires sign-off.** Via
> **migration 0037** (`0037_app_launch_profiles`): **one new join table `app_launch_profiles`**
> (`app_id`, `launch_profile_id`), which constrains **which launch profiles a user may pick** for
> an app. **No existing table, column, type, constraint, default, or the session state machine
> changes.** In particular it does **not** disturb UI-P4's expand/contract state:
> `stream_profiles.codecs` and the legacy (non-rung) `stream_profiles` rows stay exactly where
> migration 0036 left them, awaiting the separate contract migration.
> **A TABLE, NOT A COLUMN**, for the reason the whole schema prefers one: a `jsonb`/`text[]` column
> would carry no referential integrity, so an entry could name a launch profile that no longer
> exists and nothing would notice until a menu offered something that resolves to nothing.
> **Empty set = today's behaviour** (any launch profile the device is eligible for) — the state of
> every existing app, so no backfill and no behaviour change on upgrade. Non-empty **intersects**
> with eligibility; it can only narrow, never widen.
> **The app's default is implicitly always included and is NOT stored here.** It is
> `apps.default_profile_id` one table over, and a stored copy would need syncing on every default
> change.
> **Only meaningful for `profile_policy IN ('inherit','prefer')`.** `force` pins the app's profile
> outright, so no allow-list can apply. That is enforced in the **write path** (which refuses to
> store a list for a `force` app and clears any existing one), not by a `CHECK` — the rule spans two
> tables, the same reason the H.264 floor rule is a write-time rule.
> **Both foreign keys are `ON DELETE CASCADE`, and the launch-profile side is the one that had a
> real choice.** `RESTRICT` would have made retiring a launch profile harder the more carefully an
> operator had curated their apps; `DELETE /v1/admin/launch-profiles/{id}` already refuses (`409`)
> for the three references that would otherwise leave something pointing at nothing
> (`apps.default_profile_id`, the global default, a user preference), and an allow-list entry is not
> one of those. **The cost is stated rather than discovered:** a cascade that empties an app's list
> turns it back into "unrestricted", so a delete can *widen* a menu. That is bounded — the widened
> set is still only what the device is eligible for, which is the pre-UI-P5 behaviour, and this list
> is stream-quality curation and never an authorization boundary — and the affected apps are written
> into the admin activity log so it is recorded.
> **The enforcement is at launch, not in the UI.** `POST /v1/sessions` rejects a `profile_id`
> outside the list with `409 conflict` (`control-api.md` §Sessions); the filtered read
> `GET /v1/me/profiles?app_id=…` is a convenience. **`agent-api.md` is unchanged** — the agent
> receives one resolved `stream` block and has never known what a profile is. `signaling.md`,
> `input.md`, `native-client.md` are unchanged. See
> `docs/design/plans/2026-07-27-ui-implementation-spec.md` §"Phase 5" and
> `docs/design/plans/2026-07-27-admin-mockup-implementation-notes.md` §3.

> **Amendment — Steam library discovery, Phase 1 (`apps` external ref), additive, requires
> sign-off.** Via **migration 0042** (`0042_app_external_ref`): **two new columns** —
> **`apps.external_source TEXT NOT NULL DEFAULT '' CHECK (external_source IN ('', 'steam'))`** and
> **`apps.external_id TEXT NOT NULL DEFAULT '' CHECK (external_id = '' OR external_id ~
> '^[1-9][0-9]{0,9}$')`** — plus the **partial index `apps_external_ref_idx ON apps
> (external_source, external_id) WHERE external_id <> ''`**. Together they say *"this app **is**
> provider X's title Y"*, today only `('steam', <appid>)`, surfaced as `control-api.md`
> `AppListItem.external_source` / `.external_id`. **Purely additive** — two new defaulted columns
> and one new index; no existing table, column, type, constraint, default, or the session state
> machine changes. **`''` on both means "this app is not a provider title"**, which is every
> existing row, so **no backfill** and no behaviour change on upgrade.
> **Only one thing reads them, and it is artwork resolution** (`control-api.md` §Cover artwork): an
> app carrying an appid resolves its two crops **by id** and never enters the fuzzy title matcher —
> an exact id beats a fuzzy title match by construction. **Nothing in scheduling, admission,
> profile/codec resolution, or the agent wire reads either column.** The `app_artwork` shape is
> untouched; `provider_ref` simply now carries the appid for a tagged app.
> **`apps_external_id_ck` IS A SECURITY CONTROL, not a tidiness constraint** (spec §10): the value
> is eventually rendered into `STEAM_STARTUP_FLAGS`, which the `quasar-steam` entrypoint
> word-splits with `read -r -a`, so anything stored here is read by a shell-adjacent consumer as
> **arguments**. The appid is validated at four independent points and this is the one that also
> covers **an admin editing the column later**, which a handler guard cannot — the same reason
> `apps.kind` carries a `CHECK` behind its handler gate, applied to a value that has real teeth.
> **SCOPE — this is Phase 1 of a five-column plan and deliberately lands two.** Spec §4.1 also
> defines `parent_app_id`, `origin`, `library_provider`, the derived-tile shape `CHECK` and the
> `(parent_app_id, external_source, external_id)` unique index; **all five of those are Phase 3**
> and are not in 0042. Phase 1 ships only what it reads.
> **The down migration drops both columns**, so which apps were tagged with an appid is lost and a
> re-apply leaves every app back on the fuzzy title path until an admin re-tags them. The
> `app_artwork` rows themselves are keyed on `app_id` and survive, so the casualty is a *tagging*,
> not a cache. **`agent-api.md` is byte-identical** — the whole Steam library discovery spec
> deliberately avoids touching the agent contract (spec §14). `signaling.md`, `input.md`,
> `native-client.md` are unchanged. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §4.1, §10, §12 and §13 "Phase 1".

> **Amendment — Steam library discovery, Phase 2 (`entitlements`). NOT ADDITIVE at the API, and
> requires Opus + explicit human sign-off.** The *schema* half is one new table
> (**migration 0043**, `0043_entitlements`) and touches no existing table, column, type,
> constraint, default, or the session state machine. What is not additive is what reads it:
> `control-api.md`'s `GET /v1/apps` becomes entitlement-filtered for **every role**, and
> `POST /v1/sessions` / `POST /v1/sessions/{id}/swap` gain a terminal `403`. Read that amendment
> for the blast radius; this one is the storage contract behind it.
> **An entitlement is one fact: "this subject may see and launch this app."** Presence of the row
> is the fact — no `revoked` boolean, no soft delete. `subject_type`/`subject_id` is deliberately
> **wider than the Steam use case** (the roadmap's library-provider object, not a narrower
> parallel mechanism): an admin grant, "everyone", and a future `group` are all expressible
> without a retrofit. Phase 2 ships only `('user','all')`; `'group'` is additive when it comes —
> a new `CHECK` value and a third partial unique index, no shape change.
> **TWO PARTIAL UNIQUE INDEXES, NOT ONE PLAIN `UNIQUE`, and that is a correctness requirement.**
> Postgres does not treat `NULL`s as equal in a `UNIQUE` constraint, so a single
> `UNIQUE (subject_type, subject_id, app_id)` would consider every `('all', NULL, <app>)` row
> distinct from every other and **silently permit unlimited duplicate `all` rows for one app**.
> Not merely untidy: the read predicate is `EXISTS`, so duplicates would not corrupt the library
> list and nothing would fail loudly — but a revoke that deletes *"the"* row would leave the app
> still visible, and a grant's `ON CONFLICT` idempotency would have nothing to conflict on.
> Splitting by shape gives each shape a real uniqueness key. (Postgres 15+ `NULLS NOT DISTINCT`
> would also work; two partial indexes are used because they additionally serve as the lookup
> indexes for the two halves of the filter predicate and carry no version floor.)
> **THE BACKFILL IS IN THE SAME TRANSACTION AS THE `CREATE TABLE`, and it is the single most
> load-bearing statement in the migration.** Filtering goes live in the same deploy that creates
> the table, and against an empty table that filter returns nothing — **every user's library goes
> blank on every existing deployment, simultaneously.** And it would ship: the migration applies
> cleanly, the control plane boots, `go-test-db` passes (the tests create their own entitlements)
> and the web build passes. **There is no automated gate between "empty table" and "every library
> is empty" other than this `INSERT`.** Granting `('all', granted_by='migration')` for every app
> that exists makes Phase 2's day-one behaviour change **exactly zero** — every user sees
> precisely the apps they saw before, because "no entitlements" is what universal visibility
> meant before this table existed — and every subsequent narrowing is then a deliberate admin
> action measured against a visible baseline.
> **Disabled apps are backfilled too.** The filter is ANDed with `apps.enabled = true`, so a
> disabled app is invisible either way today — but skipping it would mean re-enabling an app
> silently failed to bring it back, months later, with nothing to point at. The backfill is over
> the **catalogue**, not over what is currently visible.
> **The down migration drops the table**, which drops the backfill *and every grant an admin has
> made since*. Re-applying 0043 re-runs the backfill, so the catalogue comes back **fully open**
> rather than at whatever narrower state the operator had configured: a **widening**, so it fails
> safe for availability and **unsafe for confidentiality**. Dump `entitlements` before rolling
> back any deployment where access has actually been restricted. Note this is for a deliberate,
> operator-run down migration — never for "deploy `main` over the phase branch", which the
> one-way migration rule in `CLAUDE.md` already forbids.
> **`agent-api.md` is byte-identical** — the whole Steam library discovery spec deliberately
> avoids touching the agent contract (spec §14). `signaling.md`, `input.md`, `native-client.md`
> are unchanged. See `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §6 and §13
> "Phase 2". *(Numbering note: the spec calls this migration 0042 throughout, written when
> Phases 1 and 2 shared one. Phase 1 shipped first and took 0042; §13's "0042 continued, or 0043
> if Phase 2 shipped separately" is the clause that applies.)*

> **Amendment — Steam library discovery, Phase 3 (derived tiles + the `launcher` kind). Additive,
> and requires Opus + explicit human sign-off on TWO counts** — it widens a **frozen enum**
> (`control-api.md` `AppKind`) and it adds an app shape whose rules are a database `CHECK` rather
> than a convention. **Migration 0044.** It touches one existing constraint and adds three columns,
> three constraints and one unique index to `apps`; it creates no table and changes no other
> table, no type, no default, and not the session state machine.
> **(1) `apps.kind`'s `CHECK` is widened to `('game','desktop','launcher')`.** Widening is
> **data-safe in itself** — no existing row can violate a strictly larger allow-list, so there is
> no backfill and no validation scan, and the recreate is deliberately **not** `NOT VALID`.
> **THE TRAP IS THE CONSTRAINT NAME, NOT THE DATA.** `apps.kind` was declared by migration 0034 as
> an **inline, unnamed** column `CHECK`, so it carries a **server-generated** name — conventionally
> `apps_kind_check`, but that convention is not a guarantee. Widening is a drop-and-recreate, and
> a `DROP CONSTRAINT` naming a constraint that does not exist **fails the migration on boot**,
> which is the control-plane crash-loop class `CLAUDE.md` warns about. **Verify the real name
> against the live database** (`\d apps`, or `pg_constraint` filtered on `conrelid`) before writing
> the migration, and rehearse on a production snapshot.
> **`kind` REMAINS PRESENTATION-ONLY AND THAT IS NOW A LOAD-BEARING PROMISE.** A third value
> invites `AND kind = 'launcher'` in a scan or launch query. **Nothing in scheduling, admission,
> placement, profile/codec resolution, discovery or the agent wire reads this column.** The
> discovery trigger is `library_provider`, below, and gating it on `kind` would let an operator
> stop a background job by changing a presentation dropdown. The single reader anywhere is artwork
> resolution, which short-circuits `'desktop'` **and** `'launcher'` — presentation deciding
> presentation.
> **(2) Three columns on `apps`: `parent_app_id`, `origin`, `library_provider`**, plus
> `apps_derived_shape_ck`, `apps_origin_ck`, `apps_library_provider_ck` and
> `apps_parent_external_uk`. Every default (`NULL`, `'manual'`, `''`) makes every pre-0044 row
> valid with no backfill, and a catalogue with no derived tiles behaves exactly as it does today.
> **`apps_derived_shape_ck` IS THE POINT OF THE PHASE, NOT A TIDINESS CONSTRAINT.** A derived tile
> stores **no runtime of its own** so that the merge happens at *launch* rather than being
> flattened at *save* — which is what makes an edit to the parent (an image bump, a new GPU flag,
> a new mount) reach **every** derived tile with no re-sync and no stale copies. It is a database
> `CHECK` and not a handler rule because **a validated Tower experiment hardcoded a host path into
> a tile's `runtime_spec.mounts`**, and because `runtime_spec` has no schema validation anywhere on
> the control-plane write path (raw JSONB in, `json.RawMessage` through, opaque on the wire) — so
> there is no existing validation layer for it to live in, and only a constraint survives an admin
> editing the row later.
> **This completes the five-column plan Phase 1 deliberately split** (see the Phase 1 amendment's
> SCOPE note above): 0042 landed `external_source` / `external_id` because artwork read them; 0044
> lands the other three.
> **The down migration drops the three columns and their constraints, and narrows `kind` back to
> `('game','desktop')` — which FAILS if any row is `'launcher'`.** That is correct and deliberate:
> a silent `UPDATE apps SET kind='game'` would rewrite operator classification during a rollback.
> Re-classify the launcher apps first, then roll back. Dropping `parent_app_id` also destroys the
> parentage of every derived tile, leaving orphaned rows with a `runtime_spec` of `'{}'` that
> **cannot launch**; delete or re-parent them before a deliberate down migration.
> **The new constraints answer `4xx`, never `500`.** `apps_derived_shape_ck` maps to
> `400 validation_failed` and `apps_parent_external_uk` to `409 conflict`, each naming what the
> operator can fix — a `CHECK` violation reaching a client as `500 internal` is a lie, because the
> request is malformed and the server is not. Two rules the database cannot express as a row
> `CHECK` are enforced at the handler: `parent_app_id` must name an app that is **not itself
> derived** (home resolution substitutes the parent exactly once, so a grandchild would resolve
> its home to a tile that owns none), and `library_provider` may not be set on a derived tile.
> **`agent-api.md` is byte-identical** — a derived tile is invisible from the agent side by
> construction, exactly as a runtime preset is: the agent receives one opaque, already-merged
> `app` object. `signaling.md`, `input.md`, `native-client.md` are unchanged. See
> `docs/design/plans/2026-07-29-steam-library-discovery-spec.md` §1–§5, §4.5 and §13 "Phase 3".

The persistence model for the control plane. This **replaces Wolf's TOML-based state**:
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
*(P4-01 adds two observability tables: `session_metrics` and `user_devices`.)*

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
| `max_concurrent_sessions` | `INT` NOT NULL DEFAULT `3` | *(P2-01, migration 0002)* per-user cap on simultaneously-active sessions, admin-settable. `CHECK (max_concurrent_sessions >= 0)`; `0` blocks all launches for the user (without disabling the account). Enforced at launch (`control-api.md` `session_quota_exceeded`); "active" = `state ∈ {pending, assigned, starting, running, stopping}`. |
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
> places them) and **includes `stopping`**: teardown still owns the app home and GPU resources,
> so replacement launch must wait for the terminal callback. Terminal states are excluded.
> This set differs from the **reservation**-holding set
> `{assigned, starting, running, stopping}` used for GPU availability — quota counts the in-flight
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
| `device_id` | `UUID` NULL → `user_devices(id)` ON DELETE SET NULL | *(LP-SEC-01, migration 0020)* the device this token was minted for — the **token↔device binding** that makes device revocation real. **NULL** for tokens minted before 0020 and for logins that declare no `device_key` (legacy/native): those are not device-revocable until re-login (backfill caveat). Stamped at login from the caller's declared `device_key` (`control-api.md` `POST /v1/auth/login`). `ON DELETE SET NULL` so deleting a device row never orphans/cascades a token — revocation is an explicit token expire/revoke, not a row delete. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

Index: `(user_id)` for listing/revoking a user's tokens; `(device_id) WHERE device_id IS NOT NULL` for revoke-by-device *(LP-SEC-01)*.
A token is valid iff `revoked_at IS NULL AND expires_at > now()`.

## `instance_settings` (LP-SEC-01)
> *Additive amendment (migration 0020). A new **singleton** table (one global row) holding
> instance-wide, admin-settable config — the home for `registration_mode` and future global
> settings. Follows the `stream_profile_policy` singleton precedent (migration 0015). It changes
> no existing table. Enforcement reads this row, never a client flag.*

| column | type | notes |
|---|---|---|
| `id` | `BOOLEAN` PK DEFAULT `true` | `CHECK (id)` — the singleton idiom (only one row can exist). |
| `registration_mode` | `TEXT` NOT NULL DEFAULT `'closed'` | `CHECK (registration_mode IN ('closed','invite_only','open'))`. **Default `closed`** — the invitation system is **off** on a fresh install; nobody self-registers until an admin turns it on (`control-api.md` `PATCH /v1/admin/settings`). |
| `storage_provider` | `TEXT` NOT NULL DEFAULT `'auto'` | *(storage-config amendment, migration 0030, additive)* `CHECK (storage_provider IN ('auto','local','volume'))`. Managed-home (P5) backing store: `local` = host directories under the effective home root; `volume` = named docker volumes; **`auto`** = `local` when the session host has an effective home root, `volume` otherwise. Admin-settable via `PATCH /v1/admin/settings`; affects **new** homes only (existing `user_homes` rows keep their recorded provider/ref — refs are sticky). |
| `updated_by` | `UUID` NULL → `users(id)` ON DELETE SET NULL | last admin who changed it. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

**Seeding.** On first boot the control plane inserts the singleton, taking `registration_mode`
from the `REGISTRATION_MODE` env var **if set**, else `'closed'` — a one-time seed (idempotent,
like bootstrap-admin). **After** the row exists the admin UI / `PATCH /v1/admin/settings` is
authoritative and the env var is ignored. This is what makes "enable invites in the UI" a
persisted runtime change, not a redeploy.

## `instance_secrets` (encrypted-secrets facility, 2026-07-28)
> *Additive amendment (migration 0040). A new table; it changes no existing table. This is the
> reusable home for operator credentials that must persist but must not be readable from a
> database dump. Cover artwork's provider key is consumer #1, not the design centre.*

One row per named secret. **Instance-wide, not scoped:** everything here is a single
operator-set credential for the whole control plane — the same custody model as
`instance_settings` — so a `(scope, name)` composite key would be structure with no consumer.
Grouping lives in the **name** by convention (`artwork.steamgriddb.api_key`), which gives the
same effect for free and leaves an additive `scope` column available if a future feature ever
genuinely needs one.

| column | type | notes |
|---|---|---|
| `name` | `TEXT` PK | Stable identifier the code asks for, e.g. `artwork.steamgriddb.api_key`. Only names the build **declares** are settable through the API (an undeclared name is `404`); the column itself is unconstrained so a new secret needs no migration. |
| `ciphertext` | `BYTEA` NOT NULL | AES-256-GCM ciphertext (includes the GCM tag). The row's `name` is bound in as **additional authenticated data**, so a ciphertext copied onto a different name fails to decrypt rather than silently becoming that other secret. |
| `nonce` | `BYTEA` NOT NULL | 12 fresh random bytes per encryption, never reused. Stored because GCM needs it to decrypt; a nonce is not secret. Two writes of the same plaintext therefore differ in **both** byte columns — a correctness requirement, not a coincidence. |
| `key_version` | `INT` NOT NULL | `CHECK (key_version >= 1)`. Which master key encrypted this row. Rotation is not implemented but is deliberately not designed out: older keys can be supplied decrypt-only, and a re-encrypt sweep needs no schema change. |
| `hint` | `TEXT` NOT NULL DEFAULT `''` | Last **4** characters of the plaintext, and **empty** for a value short enough that 4 characters would be a meaningful fraction of it. Stored so the admin UI can say "a key ending `3f9a` is configured" **without** the master key being present or correct — which is exactly the situation an operator needs help diagnosing. |
| `updated_by` | `UUID` NULL → `users(id)` ON DELETE SET NULL | Last admin who set it. `SET NULL` so deleting that admin does not delete the secret and take the feature down with them. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | Maintained by the shared `set_updated_at()` trigger. |

**The master key is not in this table and must never be.** It comes from `QUASAR_SECRET_KEY`
(base64, exactly 32 bytes, optionally `"<version>:"`-prefixed), with
`QUASAR_SECRET_KEY_PREVIOUS` supplying decrypt-only predecessors. Unset is a **supported
state and the default**: the control plane boots normally and secret-backed features report
themselves unavailable. A key is never generated and persisted on first boot — that would
diverge across a multi-node deployment and make a backup unrestorable. **Losing the master key
makes every row here unrecoverable**; they must be re-entered. See `docs/configuration.md`.

## `invites` (LP-SEC-01)
> *Additive amendment (migration 0020). A new table; it changes no existing table. Codes are
> stored **hashed** (never plaintext-at-rest), admin-minted, delivered as **magic one-time
> links**. Invites are account provisioning, never session authority.*

One row per minted invite code. Redemption is atomic single-use (or bounded multi-use).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `code_hash` | `TEXT` NOT NULL UNIQUE | SHA-256 (hex) of the opaque code (≥128-bit entropy). Lookup key. **Plaintext is shown to the admin exactly once at mint, never stored** — same custody model as `auth_tokens.token_hash`. |
| `created_by` | `UUID` NOT NULL → `users(id)` ON DELETE CASCADE | the admin who minted it. |
| `role` | `TEXT` NOT NULL DEFAULT `'user'` | `CHECK (role IN ('user','admin'))`. The role the redeemed account is created with. **Admin-minted only** — the role is *never* claimable from the register wire; it rides the (admin-created) invite. |
| `max_uses` | `INT` NOT NULL DEFAULT `1` | `CHECK (max_uses >= 1)`. `1` = single-use (the magic-link default). |
| `used_count` | `INT` NOT NULL DEFAULT `0` | `CHECK (used_count >= 0 AND used_count <= max_uses)`. Bumped atomically on redemption. |
| `expires_at` | `TIMESTAMPTZ` NULL | NULL = no expiry. Redemption refused when `now() >= expires_at`. |
| `revoked_at` | `TIMESTAMPTZ` NULL | admin-revoked; non-null = unusable. |
| `note` | `TEXT` NULL | admin free-text ("for Bob"). |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

An invite is **redeemable** iff `revoked_at IS NULL AND used_count < max_uses AND (expires_at
IS NULL OR expires_at > now())`. **Atomic single-use consumption** (prevents two accounts on
one code under a race): `UPDATE invites SET used_count = used_count + 1 WHERE code_hash = $1 AND
revoked_at IS NULL AND used_count < max_uses AND (expires_at IS NULL OR expires_at > now())
RETURNING id, role` — zero rows ⇒ invalid/exhausted/expired/revoked, all indistinguishable to
the caller (`control-api.md` generic `400 invalid_invite`, no oracle). The `used_count` bump is
rolled back if the subsequent account create hits a `409` duplicate (so a `409` never burns a
single-use invite). Index: `(created_by)` for the admin list view.

## `apps`
The library. **Replaces Wolf's per-app TOML.** An app is a launchable container plus the
defaults the scheduler and pipeline need.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | |
| `name` | `TEXT` NOT NULL | display name |
| `description` | `TEXT` NOT NULL DEFAULT `''` | |
| `cover_url` | `TEXT` NULL | library art — the **TILE crop** (**2:3 portrait**, the frame the library grid renders; 16:10 before issue #385, whose move to portrait box art is an operator-directed deviation from the signed-off mockup — the hero crop stays wide, and no column or shape changed with it). *(UI-P7, migration 0039)* Present since 0001 but **never written until UI-P7**; the artwork service now owns it for any app carrying an `app_artwork` row. `NULL` = no artwork = the gradient tile, which is a deliberate design, not an error state. Stores a **local** path (`/v1/artwork/<sha256>.<ext>`), never a third-party URL: a self-hosted deployment must not depend on a third party at browse time, and a hotlinked `<img>` would report the whole library to that third party on every page view. |
| `hero_url` | `TEXT` NULL | *(UI-P7, migration 0039)* the **HERO crop** — a much wider banner for the detail/hero panels. A **different source asset**, not `cover_url` scaled: a 2:3 portrait tile stretched into a ~3:1 hero reads as a blown-up thumbnail — the two crops are further apart in shape since #385, so the split matters more, not less. Null independently of `cover_url` (a title may have one crop and not the other); the client falls back `hero_url` → `cover_url` → gradient. Same local-path rule. Read-only on the API write shape — the artwork service is the only writer, together with the `app_artwork` row, in one transaction, so the render URLs and the provenance can never disagree. |
| `kind` | `TEXT` NOT NULL DEFAULT `'game'` | *(UI-P1, migration 0034; **enum widened by Steam library discovery Phase 3, migration 0044**)* `CHECK (kind IN ('game','desktop','launcher'))`. **Presentation-only** library classification, surfaced as `control-api.md` `AppListItem.kind` so the client can split/filter the library. **Nothing in scheduling, admission, profile/codec resolution, or the agent wire reads it** — it is not an input to any decision, and Phase 3 makes that promise load-bearing rather than incidental: **discovery is triggered by `library_provider` below and never by this column**, so an operator changing a presentation dropdown can never stop a background job. The single reader anywhere is artwork resolution, which short-circuits `'desktop'` **and** `'launcher'` (`control-api.md` §Cover artwork) — presentation deciding presentation. `'launcher'` is the category the Steam **provider app** is filed under once it has derived children. The default makes every existing row valid with no backfill, and so does the widening (no row can violate a strictly larger allow-list). Optional on the write shape, where **absent means "default on create, unchanged on patch" — never a zero value** (`control-api.md`); the `CHECK` here is the backstop, not the primary gate. **The 0034 declaration was an inline, unnamed column `CHECK`, so it carries a server-generated name (conventionally `apps_kind_check`) — verify it against the live database before any further widening, because a `DROP CONSTRAINT` on a wrong name fails the migration on boot.** |
| `parent_app_id` | `UUID` NULL → `apps(id)` **ON DELETE CASCADE** | *(Steam library discovery Phase 3, migration 0044)* the app this row is **derived** from; `NULL` = a normal app, which is every pre-0044 row. A derived tile carries **identity and presentation only** and borrows everything executable — `image` / `runtime_spec` / `runtime_preset_id`, `managed_home` / `home_container_path`, the user's home, the resource defaults and the mounts — from its parent **at launch**. Surfaced on `control-api.md` `AppListItem` (the public read shape), unlike `origin` and `library_provider`, because the client needs it to mark the siblings of a live session blocked. **Self-referential and one level deep: a parent may not itself be derived.** `CASCADE` so the database can never hold an orphan tile, but it is the **integrity backstop under an application-layer confirmation**, not the primary UX — deleting a parent with derived tiles is `409` **listing them** (`error.derived_tiles`, a list and not a count — the point of a confirmation is that the admin sees what they will destroy) unless the request explicitly opts in with `?delete_derived=true`. **EVERY STORAGE-KEYED GUARD MUST SUBSTITUTE `parent_app_id ?? id`** — the single-writer launch guard, the swap guard, the home-tombstone in-use guard, home resolution, `last_used_at` touch, `bytes_used` reporting, and the placement locality lookup. A guard keyed on the tile silently reports "not in use" for a session that is actively writing, and two Steam clients on one `steamapps` tree is a documented corruption class. Indexed by `apps_parent_app_id_idx` (partial, `WHERE parent_app_id IS NOT NULL`) — Postgres does not auto-index the referencing side of an FK, and `apps_parent_external_uk` below leads on the same column but cannot serve the plain *"which tiles belong to this parent"* lookup that the delete guard and the launch-time family check both make. |
| `origin` | `TEXT` NOT NULL DEFAULT `'manual'` | *(Steam library discovery Phase 3, migration 0044)* `CHECK (origin IN ('manual','discovered'))`. **Provenance, not authority** — both launch identically. `'manual'` = an operator created it, which is every pre-0044 row and every app an admin creates. `'discovered'` = a library-discovery sync created it; **nothing writes it yet, that is Phase 4.** It is in the `CHECK` now so Phase 4 needs no `ALTER`. Surfaced on `control-api.md` `AdminApp` and **read-only there — deliberately not on `AppWrite`**: an admin create is `'manual'` by construction, and letting an operator declare a hand-made tile `'discovered'` would lie to the reconciler, which keys its create/suppress decisions off provenance. Both directions are footguns under `apps_parent_external_uk` below — a tile relabelled `'manual'` still occupies its `(parent, source, appid)` slot, so a reconciler reading it as *"not mine, therefore missing"* cannot re-create it either. *(Known discrepancy pending sign-off: the Phase 3 control-plane currently also accepts it on write; `control-api.md` §Derived tiles records it and the argument rather than papering over it. The column itself is identical either way.)* The `TEXT` + `CHECK` convention as elsewhere in this schema. |
| `library_provider` | `TEXT` NOT NULL DEFAULT `''` | *(Steam library discovery Phase 3, migration 0044)* `CHECK (library_provider IN ('', 'steam'))`. This app **IS the configured client for** external provider X, so installs under its managed home can be discovered. `''` = not a provider app, which is every pre-0044 row. **THIS COLUMN — AND ONLY THIS COLUMN — IS THE TRIGGER FOR DISCOVERY (Phase 4).** It is **operator-set and never inferred from the image name**, because image names change and a wrong inference here starts a filesystem scan of somebody's home. **Read it against `kind`, not with it:** `library_provider='steam'` is the functional fact and `kind='launcher'` is presentation; an admin editor may suggest one alongside the other, but nothing server-side branches on `kind`. **Do not confuse it with `external_source`**, which says the app IS a provider *title*; the two are mutually exclusive on one row, and a derived tile must carry `library_provider = ''` by the shape `CHECK` below. Optional on the write shape, where an explicit `''` **is** valid (a deliberate un-marking) but absent means **unchanged on patch** — the `cb97bfb` rule, and here an absent field overwriting the column would turn scanning off for a whole instance with nobody touching the setting. |
| *(constraint)* | — | *(Steam library discovery Phase 3, migration 0044)* **`apps_derived_shape_ck`** — `parent_app_id IS NULL OR (external_source <> '' AND external_id <> '' AND managed_home = false AND runtime_preset_id IS NULL AND runtime_spec = '{}'::jsonb AND library_provider = '')`. **THE POINT OF THE PHASE, NOT TIDINESS.** `runtime_spec = '{}'` is what makes *"the tile contributes no runtime"* **structural rather than conventional**: the parent's spec is merged in at launch, so an edit to the parent reaches every tile with no re-sync and no stale copies — the same reasoning UI-P3 applied to runtime presets. It is a database constraint because **a validated Tower experiment hardcoded a host path into a tile's `runtime_spec.mounts`**, and because `runtime_spec` has **no schema validation anywhere on the control-plane write path** (raw JSONB in, opaque on the wire), so there is no existing validation layer for this rule to live in and only a `CHECK` survives an admin editing the row later. `managed_home = false` is load-bearing in the other direction too: the single-writer guard is gated on `managed_home`, so it must read the **parent's** value or it does not fire for derived tiles at all — the exact inverse of its purpose. |
| *(index)* | — | *(Steam library discovery Phase 3, migration 0044)* **`apps_parent_external_uk ON apps (parent_app_id, external_source, external_id) WHERE parent_app_id IS NOT NULL`** — one tile per (provider app, appid) **fleet-wide**. This is what makes the catalogue bounded by the **union** of all users' installed titles rather than by users × titles, and it is what lets a second user's scan of the same game find the existing tile instead of creating a duplicate. **Partial** on `parent_app_id IS NOT NULL`, because Postgres does not treat `NULL`s as equal in a unique index — every normal app would be trivially distinct and the index would be almost entirely dead keys. |
| `external_source` | `TEXT` NOT NULL DEFAULT `''` | *(Steam library discovery Phase 1, migration 0042)* `CHECK (external_source IN ('', 'steam'))`. Read with `external_id` below as one fact: **"this app IS provider X's title Y"**, today only `('steam', <appid>)`. `''` = **not a provider title**, which is every pre-0042 row, so the default makes them all valid with no backfill. Surfaced as `control-api.md` `AppListItem.external_source`, so it is on the public read shape, not admin-only: it is **identity**, not operator configuration. Optional on the write shape, where **absent means "default on create, unchanged on patch" — never a zero value**, but where an explicit `''` **is** valid (unlike `kind`, whose `''` is a `400`) because `''` is the real domain value here; the `CHECK` is the backstop, not the primary gate. Widening the enum is a later migration (the `TEXT` + `CHECK` convention above). **Nothing in scheduling, admission, profile/codec resolution, or the agent wire reads it** — the sole Phase 1 reader is artwork resolution. |
| `external_id` | `TEXT` NOT NULL DEFAULT `''` | *(Steam library discovery Phase 1, migration 0042)* `CHECK (external_id = '' OR external_id ~ '^[1-9][0-9]{0,9}$')` — a bare positive integer, no leading zero, no sign, no whitespace, no separators, so `'0'`, `'007'`, `'1 2'`, `'1;rm -rf /'` and `'-applaunch 480 -foo'` are all rejected **at the storage layer**. **THAT `CHECK` IS ARGUMENT-INJECTION CONTAINMENT, NOT TIDINESS** (spec §10): the value is eventually rendered into `STEAM_STARTUP_FLAGS`, which the `quasar-steam` entrypoint **word-splits** with `read -r -a`, so anything stored here reaches a shell-adjacent consumer as **arguments**. The appid is validated at four independent points (agent parse, control-plane ingest, here, launch-time render) and this is the one that survives **an admin editing the column later**, which is precisely why it is a database constraint and not only a handler guard. `''` = no provider id. Same write-shape semantics as `external_source`; the handler carries the identical regex as the primary gate. |
| *(index)* | — | *(Steam library discovery Phase 1, migration 0042)* **`apps_external_ref_idx ON apps (external_source, external_id) WHERE external_id <> ''`** — the artwork resolver's *"which app is `steam:<appid>`"* lookup, and the index the later discovery reconciler reuses. **Partial**, because every app in a pre-discovery catalogue has `''` and a full index would be almost entirely one dead key. |
| `runtime_spec` | `JSONB` NOT NULL DEFAULT `'{}'` | container launch spec the **node agent** consumes: `{ "image", "args":[], "env":{}, "mounts":[], "gpu":true }`. JSONB (not frozen columns) because this is agent-internal launch detail that will grow; the scheduler does **not** read it. |
| `default_vram_mb` | `INT` NOT NULL DEFAULT `1024` | **DEPRECATED (#383)** — no longer read by the scheduler. Retained so existing rows and API clients are undisturbed; still accepted on write and returned on read, but placement ignores it. Admission now gates on `default_encode_slots` plus the live free-VRAM veto (`control-api.md` §Admission control). It was never enforceable — no VRAM cap is applied to a session — so it only ever encoded a guess. Slated for removal once a release has passed with no readers. |
| `default_encode_slots` | `INT` NOT NULL DEFAULT `1` | encode sessions this app needs (normally 1). |
| `default_width` | `INT` NOT NULL DEFAULT `1920` | launch defaults for the P1-5 pipeline; per-launch overrides live on `sessions`. |
| `default_height` | `INT` NOT NULL DEFAULT `1080` | |
| `default_fps` | `INT` NOT NULL DEFAULT `60` | |
| `default_bitrate_kbps` | `INT` NOT NULL DEFAULT `15000` | |
| `enabled` | `BOOLEAN` NOT NULL DEFAULT `true` | hidden from the public library when false. |
| `managed_home` | `BOOLEAN` NOT NULL DEFAULT `false` | *(P5-01, migration 0008)* when true the control plane injects the caller's per-(user, app) home into `runtime_spec.mounts` at dispatch (assign **and** swap) and enforces the single-writer + quota rules (`control-api.md` §Storage). False = today's stateless behaviour, byte-identical dispatch. |
| `home_container_path` | `TEXT` NOT NULL DEFAULT `'/home/quasar'` | *(P5-01, migration 0008)* container-side mount point for the managed home. |
| `default_profile_id` | `TEXT` NULL → **`launch_profiles(id)`** | *(migration 0015; **FK repointed by UI-P4 / migration 0036** from `stream_profiles(id)`)* the **launch profile** this app pins or prefers, per `profile_policy`. The stored value is unchanged by the repoint, because existing ids are preserved as launch profile ids. |
| `profile_policy` | `TEXT` NOT NULL DEFAULT `'inherit'` | *(migration 0015)* `CHECK (profile_policy IN ('inherit','prefer','force'))` **as of UI-P4 / migration 0036 — `'custom'` was removed** (`control-api.md` amendment B2). `inherit` = the user/global default decides; `prefer` = the app's `default_profile_id`, user may still override; `force` = the app's profile always. `custom` (the app opting out of profiles and carrying its own `default_width/height/fps/bitrate_kbps`) is gone: under the two-object model every app points at a launch profile, and `custom` was also the one mode that could not express a codec. **The `default_*` columns above stay** — they are the `display_stream` COALESCE fallback when no launch profile resolves, they are what `LaunchConsoleSession` uses unconditionally, and they are the ceiling on the `POST /v1/sessions` stream-override escape hatch, which is reachable for **any** app. |
| *(no column)* | — | *(UI-P5, migration 0037)* the app's **launchable launch-profile allow-list** lives in the join table **`app_launch_profiles`**, not on this row. Empty = unrestricted = the pre-UI-P5 behaviour of every app. `default_profile_id` above is implicitly always included in it and is deliberately not duplicated there. |
| `runtime_preset_id` | `UUID` NULL → `runtime_presets(id)` **ON DELETE RESTRICT** | *(UI-P3, migration 0035)* the shared runtime preset this app inherits its container configuration from. **`NULL` = the app carries everything itself = the pre-UI-P3 behaviour**, so no existing row changes. The preset is **never** flattened into `runtime_spec` on save — the merge happens server-side at launch (`control-api.md` §Runtime presets), which is what makes a preset edit reach every app already using it. `RESTRICT` is a backstop under the application-layer `409` on delete-in-use, not the gate; `SET NULL` would silently strip an app's image/env/mounts and let it launch with a smaller spec instead of failing. Indexed (`apps_runtime_preset_id_idx`) — Postgres does not auto-index the referencing side of a FK, and both the in-use check and the admin "Used by" list are per-preset lookups. |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | |

## `runtime_presets` (UI-P3)
> *Additive amendment (migration 0035). A new table; it changes no existing table semantics.*

A reusable container configuration many apps inherit instead of repeating. **This is not a
launch profile** — launch/stream profiles (UI-P4) are the quality/encode chain and have nothing
to do with this table. A runtime preset says *what container runs*; a launch profile says *how
the stream is encoded*.

The four container fields mirror the shape of `apps.runtime_spec`
(`{image, args, env, mounts}`) but as first-class columns rather than one opaque blob, because
the admin editor edits them individually and the launch-time merge reads them individually.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | |
| `name` | `TEXT` NOT NULL UNIQUE | display name; unique so an operator cannot end up with two presets they cannot tell apart. A collision is `409 conflict`. |
| `description` | `TEXT` NOT NULL DEFAULT `''` | what this preset is for, shown when picking one on an app. |
| `image` | `TEXT` NOT NULL DEFAULT `''` | the container image apps inherit. An app that sets its own `runtime_spec.image` overrides it; blank/absent on the app inherits this. |
| `args` | `JSONB` NOT NULL DEFAULT `'[]'` | launch arguments. **Prepended** to the app's own at launch (preset first). |
| `env` | `JSONB` NOT NULL DEFAULT `'{}'` | environment. Merged under the app's at launch — **a key set on the app wins**. |
| `mounts` | `JSONB` NOT NULL DEFAULT `'[]'` | mounts. **Prepended** to the app's own, with **no dedupe**: two mounts on one container path is a real misconfiguration and must surface rather than be silently resolved. |
| `managed_home` | `BOOLEAN` NOT NULL DEFAULT `false` | storage **default** for inheriting apps. `apps.managed_home` is `NOT NULL DEFAULT false` with no "unset", so an app can turn a managed home **on** when its preset has none but cannot turn a preset's **off** — a preset that provisions a per-user home is a storage guarantee for everything inheriting it. |
| `home_container_path` | `TEXT` NOT NULL DEFAULT `'/home/quasar'` | container-side mount point default. An app still at the schema default has expressed no preference and takes this; an app with its own non-default path keeps it. |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | `updated_at` is maintained by the shared `set_updated_at()` trigger. |

**Referenced by** `apps.runtime_preset_id` (`ON DELETE RESTRICT`). Deleting a preset that any app
references — **including a disabled app**, which still holds the reference — is refused with
`409 conflict` at the application layer (`control-api.md`). The FK is the backstop for every
other path.

## `stream_profiles` (migration 0015; reshaped by UI-P4)
The encode-rung catalog. Created by migration 0015 and, before UI-P4, it *was* the profile a user
picked. **As of UI-P4 a row is one rung: one codec at one resolution, frame rate and bitrate.** It
is no longer user-facing; users pick a `launch_profiles` row, which lists these in preference
order. This table has never had a prose section here (the pre-0015 catalog was in-code); only the
columns UI-P4 touches are documented, and the committed `0015` SQL remains the definition of the
rest.

| column | type | notes |
|---|---|---|
| `id` | `TEXT` PK | pre-UI-P4 rows use the quality-tier id (`'1080p60'`); rung rows created by 0036's fan-out use `<launch-profile-id>-<codec>` (`'1080p60-h264'`). |
| `codec` | `TEXT` NULL | *(UI-P4, migration 0036)* `CHECK (codec IN ('h264','hevc','av1'))` — the **catalog** vocabulary, matching `codecs` below; the wire spells HEVC `h265` and the rename is bridged in exactly one place server-side. **NULL on every pre-0036 row**, non-NULL on every rung. The **contract** migration makes it `NOT NULL` once the legacy rows are gone. |
| `codecs` | `JSONB` NULL | *(multi-codec, migration 0031)* the pre-UI-P4 ordered `[{codec,status}]` preference list. **Retained by 0036 and read by nothing on the launch path after it** — the rung's `codec` is authoritative. It exists solely so a code-level revert onto a migrated database still finds the data it expects; the **contract** migration drops it. Its `launchable \| future \| unsupported` status enum is gone from the API (`control-api.md` amendment B3). |
| `visibility` | `TEXT` NOT NULL | rung rows created by the fan-out are `'internal'`: a rung is never offered standalone, only as part of the launch profile that lists it. The launch profile carries the user-facing visibility. |
| *(all other columns)* | | unchanged from 0015. Every rung created by the fan-out copies **every** one of them from its parent row **verbatim** (resolution, fps, bitrate, ABR floor, playout0, `h264_profile`, the eligibility thresholds, `hardware_encoder_required`, `browser_client`). **No per-codec tuning happens in the migration** — that is the operator-facing payoff of the restructure and is a deliberate data change afterwards, not something smuggled into a behaviour-neutral migration. |

**Referenced by** `launch_profile_rungs.stream_profile_id` (`ON DELETE RESTRICT`) and
`sessions.stream_profile_id`. Deleting a rung listed by any launch profile is `409 conflict` at the
application layer; the `RESTRICT` is the backstop.

## `launch_profiles` (UI-P4)
> *New table, migration 0036. This is the object a user picks.*

An ordered chain of encode rungs, best first. The launch walks them and takes the first the placed
host can encode and the client can decode, so falling through a rung can change **resolution** as
well as codec. Fall-through happens **at launch only**, never mid-session.

| column | type | notes |
|---|---|---|
| `id` | `TEXT` PK | 0036 preserves the pre-UI-P4 `stream_profiles` ids here (`'1080p60'`, `'high'`, …). That is what keeps the three un-FK'd `TEXT` id columns elsewhere (`sessions.profile_id`, `user_device_profile_history.profile_id`, `host_encoder_certification.profile_id`) meaningful, and what keeps the in-code `conservativeDefaultID = "1080p60"` and the SPT-06 `lowerProfileRung` ladder correct without edit. |
| `display_name` | `TEXT` NOT NULL | picker label. |
| `description` | `TEXT` NOT NULL DEFAULT `''` | operator-facing note shown in the admin list and editor. |
| `visibility` | `TEXT` NOT NULL DEFAULT `'user'` | `CHECK (visibility IN ('user','debug','internal'))`. Same semantics as the pre-UI-P4 `stream_profiles.visibility`, migrated from it. Debug/internal launch profiles are never returned by `GET /v1/me/profiles`. |
| `sort_order` | `INT` NOT NULL DEFAULT `0` | catalog order, highest quality first. Migrated from the parent row's `sort_order`. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | maintained by the shared `set_updated_at()` trigger. |

**Referenced by** `apps.default_profile_id`, `stream_profile_policy.global_default_profile_id` and
`user_profile_preferences.default_profile_id` — the **three** FKs 0036 repoints. Deleting a launch
profile referenced by any of them is `409 conflict` at the application layer.

**The H.264 floor is a write-time rule, not a constraint.** A launch profile must contain at least
one rung whose codec is `h264`. It is enforced in the API (`400 validation_failed`), not by a
`CHECK`, because the rule spans two tables. "And it must be **last**" is a **warning**, not a
rejection: rejecting would make a migrated launch profile whose stored codec order puts h264 first
permanently uneditable, and it would add no safety, because the guarantee is the resolver's
unconditional floor — if no rung survives, the **last h264 rung dispatches bypassing every clamp**,
including its own `hardware_encoder_required`.

## `launch_profile_rungs` (UI-P4)
> *New table, migration 0036. The ordered join. A table, not a `jsonb` column, so the rung
> reference is a real FK with a defined refusal on delete.*

| column | type | notes |
|---|---|---|
| `launch_profile_id` | `TEXT` NOT NULL → `launch_profiles(id)` **ON DELETE CASCADE** | deleting a launch profile takes its rung list with it; the rungs themselves are shared objects and survive. |
| `stream_profile_id` | `TEXT` NOT NULL → `stream_profiles(id)` **ON DELETE RESTRICT** | `RESTRICT`, not `CASCADE`: silently removing a rung from every launch profile that lists it is exactly the kind of quiet quality change this contract exists to prevent. The application layer refuses the delete with `409` first. |
| `position` | `INT` NOT NULL | `CHECK (position > 0)`. **Order is preference**, assigned by the server from the ordered id array the write shape carries; a client never sends positions. |

`PRIMARY KEY (launch_profile_id, position)` and `UNIQUE (launch_profile_id, stream_profile_id)` (a
rung may appear at most once in one launch profile). Indexed on `(stream_profile_id)` for the
delete-in-use check and the admin "Used by" list — Postgres does not auto-index the referencing side
of a FK.

## `app_artwork`
*(UI-P7, migration 0039)* Where an app's artwork came from, which locally cached blobs back it,
and whether an admin has corrected a wrong match. `apps.cover_url` / `apps.hero_url` are the
denormalised render URLs the library reads (so the library query is untouched); **this** table is
what the fetcher and the admin override reason about.

| column | type | notes |
|---|---|---|
| `app_id` | `UUID` PK → `apps(id)` **ON DELETE CASCADE** | artwork is a property *of* the app and has no meaning without it. The cached blobs are content-addressed and shared, so they are **not** deleted by the cascade — an orphan sweep reclaims any blob no row references. |
| `source` | `TEXT` NOT NULL | `CHECK (source IN ('provider','manual','none'))`. `provider` = matched and fetched automatically. `manual` = an admin picked, supplied or uploaded it. **`none` = we looked and there is nothing — a NEGATIVE CACHE and a first-class outcome, not an error.** An app whose `apps.kind` is `'desktop'` **or** `'launcher'` *(Steam library discovery Phase 3 added the second)* is not in a games database and never will be, so recording "no art" is what stops every sweep re-querying a third party for a row that can never match. The Steam client is the sharper case of the two: a games database mis-matches its *name* confidently rather than missing it, which is why `'launcher'` short-circuits rather than being left to the matcher. All three states render correctly; `none` is the gradient tile. |
| `provider` | `TEXT` NOT NULL DEFAULT `''` | which provider produced the match (`'steamgriddb'`). Empty for `manual`/unmatched. |
| `provider_ref` | `TEXT` NOT NULL DEFAULT `''` | the provider's opaque id for the matched title. |
| `matched_name` | `TEXT` NOT NULL DEFAULT `''` | the provider-side title matched. Surfaced in the admin UI so an operator can **see** that "Portal" matched "Portal Knights" rather than reverse-engineering it from the picture. |
| `tile_asset` | `TEXT` NOT NULL DEFAULT `''` | cached blob name, **content-addressed**: `<sha256>.<jpg\|png\|webp>`. Never a remote URL and never a remote-chosen filename — the name is the SHA-256 of the bytes actually stored plus an extension mapped from the **sniffed** content type, so a hostile response cannot pick its own path on disk and path traversal is impossible by construction. Empty = that crop is unavailable. |
| `hero_asset` | `TEXT` NOT NULL DEFAULT `''` | same, for the hero crop. |
| `attribution` | `TEXT` NOT NULL DEFAULT `''` | credit line rendered beside the art when the source asks for one. |
| `locked` | `BOOLEAN` NOT NULL DEFAULT `false` | set by any admin override. The automatic fetcher **never** touches a locked row: fuzzy matching is wrong sometimes, and an operator who fixed a match must not have that fix silently re-broken by the next sweep. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

Partial indexes on `tile_asset` / `hero_asset` (`WHERE <> ''`) serve the orphan sweep's
"enumerate every blob still referenced" direction; the fetcher's own work query is the
anti-join "apps with no row here", already served by the primary key.

**The feature ships dark.** With no provider API key configured the control plane makes no
third-party request, writes no row for a game, and every app keeps `cover_url`/`hero_url` `NULL`
— byte-for-byte the pre-UI-P7 behaviour. The local half (upload, override, clear, serving) works
regardless, because it contacts nothing.

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
| `cpu_model` | `TEXT` NULL | *(host-observability-2, additive)* last-reported CPU marketing name. |
| `storage` | `JSONB` NULL | *(host-observability, additive)* last-reported storage volumes (`agent-api.md` `capacity.host.storage`): array of `{label, path, total_mb, available_mb}`. |
| `effective_settings` | `JSONB` NULL | *(host-observability, additive)* last-reported resolved runtime settings (`agent-api.md` `capacity.effective_settings`): string map, restart-class knobs latched. |
| `codecs` | `JSONB` NULL | *(multi-codec, migration 0031, additive)* last-reported **wire** codec set the host's active encoder path can produce (`agent-api.md` `capacity.codecs`): a JSON array subset of `["h264","h265","av1"]`. NULL ⇒ the control plane assumes `['h264']` (an old agent that never reports the field is H.264-only). Keep-if-absent on report (a later `capacity` that omits the key does not clobber the stored value). |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |

### Host status state machine (P3-01)
`hosts.status ∈ {online, offline, draining}`. **Liveness** is connection-derived (the agent's
WebSocket + heartbeats, `agent-api.md`); **`draining`** is an administrative cordon
(`control-api.md` `POST /v1/hosts/{id}/drain`). The scheduler (Phase 3 placement) considers
**only `online`** hosts — `draining` and `offline` are both ineligible for new sessions.

```
                 agent (re)connect + heartbeat
   offline ──────────────────────────────────────▶ online
      ▲                                            ▲   │
      │ agent disconnect /                         │   │ admin drain
      │ heartbeat-miss (from online or draining)   │   ▼
      └──────────────────────────────────────────── draining
                       admin uncordon ─────────────┘
```

| transition | driver | trigger |
|---|---|---|
| `offline → online` | system | agent (re)connects and heartbeats — **unless** the host was `draining` before the drop (see the limitation note) |
| `online → draining` | **admin** | `POST /v1/hosts/{id}/drain` |
| `draining → online` | **admin** | `POST /v1/hosts/{id}/uncordon` (requires the agent connected) |
| `{online, draining} → offline` | system | agent disconnect or heartbeat-miss past threshold (`agent-api.md`) |

- **`draining` is stable, not transient.** A drained host stays `draining` (reachable, still
  heartbeating) until an admin uncordons it or its agent disconnects. It does **not** auto-flip to
  `offline` when its last session ends — auto-flipping would be indistinguishable from a crash and
  would be undone by the next heartbeat.
- **Offline reaping** (the existing frozen failure model — see the session state machine below and
  `agent-api.md` §Reconnection): when a host goes `offline`, its non-terminal sessions are driven to
  `failed` with `state_detail = 'host_lost'` (P3-01) and their reservations released.

> **Known limitation (Phase 3) — drain intent is not preserved across a full agent disconnect.**
> Because `status` is a single column conflating liveness with cordon-intent, a `draining` host
> whose agent disconnects becomes `offline`, and on reconnect returns to `online` (not `draining`) —
> the operator must re-drain. This is benign for short maintenance drains. The **additive** migration
> path (no shape change, exactly the pattern of the signaling-token note below) is a k8s-style
> orthogonal `hosts.cordoned_at TIMESTAMPTZ` column (schedulable ⇔ `status='online' AND cordoned_at
> IS NULL`), deferred until rolling-ops at scale need intent to survive a reconnect.

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
| `render_node` | `TEXT` NULL | *(host-observability-2, additive)* stable by-path render-node device path (`agent-api.md` `gpus[].render_node`). |
| `vram_mb_used` | `INT` NULL | *(#383, migration 0033)* last live sample from the agent heartbeat. **NULL means unknown** — never sampled, never reported, or the sample failed plausibility validation. NULL is not zero. |
| `vram_mb_free` | `INT` NULL | *(#383)* as above. This is the figure the admission veto reads. |
| `vram_sampled_at` | `TIMESTAMPTZ` NULL | *(#383)* when the control plane **ingested** the sample — its own clock, never the agent's. The debit below compares it against `sessions.started_at`, which is also DB time; mixing clocks would silently disable the debit. A sample older than the server's freshness window is treated as unknown. |
| `vram_sample_agent_ms` | `BIGINT` NULL | *(#383)* the agent's own `ts_unix_ms` for the sample. Used **only** as a monotonicity guard so a displaced (zombie) websocket cannot overwrite a fresh sample with pre-restart data. Not a time source. |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | |

**Availability (derived, not stored):**
```sql
-- encode slots free on a GPU -- the reservation dimension:
encode_slots_total - COALESCE(SUM(s.reserved_encode_slots) FILTER (WHERE s.gpu_id = g.id AND s.state IN ('assigned','starting','running','stopping')), 0)
```
The scheduler reserves and checks this transactionally (`SELECT ... FOR UPDATE` on the `gpus`
row, see P1-8) so two launches cannot oversubscribe. At N=1 the check always passes; the code
path is real.

**Live VRAM (sampled, not derived)** *(#383)*. `vram_mb_free` is a measurement, not an
accounting identity, and it is **not** a reservation — it cannot be, since nothing caps a
session's VRAM. Admission applies it as an advisory veto with an in-flight debit for sessions
whose allocation the sample cannot yet show:

```sql
-- a GPU is vetoed only when the sample is fresh, the pool is meaningful, and:
vram_mb_free - (in-flight sessions on this GPU) * <inflight estimate> < <min free floor>
```

In-flight means `state IN ('assigned','starting','running','stopping')` with `started_at` null
or newer than the sample minus one freshness window. Note this state set intentionally differs
from the reservation set above: a `stopping` pipeline still holds Vulkan image refs, so it holds
memory even though it no longer holds a reservation. Residency and reservation are different
questions.

The declared-VRAM availability formula that used to live here (`vram_mb_total − Σ
reserved_vram_mb`) is **removed** — see the deprecation notes on `apps.default_vram_mb` and
`sessions.reserved_vram_mb`.

## `sessions`
The central unit: a scheduled, resource-reserved, lifecycle-tracked stream. Created by
P1-B `/sessions` (launch), placed by the scheduler, run by the agent (P1-A), connected via
signaling (P1-D).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | session id, used across all four contracts. |
| `user_id` | `UUID` NOT NULL → `users(id)` | owner. |
| `device_id` | `UUID` NULL → `user_devices(id)` ON DELETE SET NULL | *(LP-SEC-01, migration 0020, optional)* the device that launched the session, so the account UI can show/end a device's live session. NULL = unknown (legacy / no `device_key` declared). **Read-only linkage; does not change scheduling, the session state machine, or the agent wire.** Ending it is the existing `DELETE /v1/sessions/{id}` (owner-or-admin). |
| `app_id` | `UUID` NOT NULL → `apps(id)` **ON DELETE CASCADE** | what's launched. *(admin-delete erratum, migration 0014: was no-cascade.* Deleting an app — refused while any non-terminal session references it — cascades that app's **terminal** session history away.) |
| `host_id` | `UUID` NULL → `hosts(id)` **ON DELETE CASCADE** | set on assign; NULL while `pending`. *(admin-delete erratum, migration 0014: was no-cascade.* Forgetting a host — refused while online or holding a non-terminal session — cascades that host's terminal session history away; NULL rows unaffected.)* |
| `gpu_id` | `UUID` NULL → `gpus(id)` | set on assign; the reserved GPU. |
| `state` | `TEXT` NOT NULL DEFAULT `'pending'` | `CHECK (state IN ('pending','assigned','starting','running','stopping','stopped','failed'))`. See state machine below. |
| `state_detail` | `TEXT` NULL | human-readable substate / progress. |
| `error_message` | `TEXT` NULL | populated on `failed`. |
| **launch params** | | drive the P1-5 pipeline; default from `apps` then overridable per launch. |
| `width` | `INT` NOT NULL | |
| `height` | `INT` NOT NULL | |
| `fps` | `INT` NOT NULL | |
| `bitrate_kbps` | `INT` NOT NULL | |
| `h264_profile` | `TEXT` NOT NULL DEFAULT `'constrained-baseline'` | `CHECK (h264_profile IN ('constrained-baseline','main','high'))`. P1-11 negotiates this up; the Phase-0 floor is the default. Applies to the `h264` codec only (see `codec`). |
| `codec` | `TEXT` NOT NULL DEFAULT `'h264'` | *(multi-codec, migration 0031)* `CHECK (codec IN ('h264','h265','av1'))`. The single video codec resolved server-side at launch (profile preference, clamped by host encoder + device decode + decode-failure history; guaranteed h264 floor) and sent to the agent in `session_assign.stream.codec` (`agent-api.md`). WIRE vocabulary: `h265` is HEVC (the in-code profile catalog's `hevc` maps to it in one place). Default `'h264'` so every pre-multi-codec / legacy / tier / override launch is unchanged. `h264_profile` above is meaningful only when this is `'h264'`. |
| `profile_id` | `TEXT` NULL | AS10-03: the profile id this session was launched from (e.g. `'1080p60'`); NULL for a legacy/tier/override launch. The `width`/`height`/`fps`/`bitrate_kbps`/`h264_profile` columns carry the resolved concrete values. **Not FK-constrained**, deliberately: terminal session history must survive a profile being deleted. *(UI-P4: this is now a **launch profile** id — the **user's pick**. The rung it resolved to is `stream_profile_id` below. The pre-UI-P4 note claiming "the profile catalog is an in-code table, not a DB table" was already stale: migration 0015 created `stream_profiles`.)* |
| `stream_profile_id` | `TEXT` NULL → `stream_profiles(id)` | *(UI-P4, migration 0036)* the **rung** this session resolved to (e.g. `'1080p60-h264'`); NULL for every pre-0036 session and for any legacy/tier/override/console launch. `profile_id` answers *what did the user pick*, this answers *what did they get* — and because a rung carries its own resolution, the two can legitimately disagree about width, height, fps and bitrate. The `width`/`height`/`fps`/`bitrate_kbps`/`h264_profile`/`codec` columns remain the truth for the running session; this records which catalog row produced them. It is also the column the ABR floor and the AS10-06 stream-health evaluation must read, **not** `profile_id` — reading the floor off the launch profile works only while every rung inherits its parent's floor unchanged, and silently reads the wrong number the first time an admin tunes a per-rung bitrate, which is the entire point of the restructure. |
| `codec_decision` | `JSONB` NULL | *(UI-P6, migration 0038)* **how** the rung/codec above was resolved: every rung walked in position order, the clamp that rejected each (`host_encoder`/`client_decode`/`decode_height`/`decode_history`/`hardware_encoder`/`unknown_codec`), and whether the dispatched rung won on merit, was forced by an operator override, or is the unconditional h264 floor. Shape and the three-outcomes rule: `control-api.md` §Codec decision. NULL for every pre-0038 session and for any launch that walked no rung chain. **Observability only** — nothing in scheduling, admission, rung resolution or the agent wire reads it back. Written exactly once, in the single post-placement stream update that already persists the resolved values, so the two can never disagree. **A `JSONB` column rather than a child table** deliberately: the document is written once, never updated, never queried *by* its contents, and only ever read back whole with its session row, so a child table would buy queryability nobody needs and cost a join on the session read path. The rung ids inside are frozen text, not foreign keys — the record must survive the rung being retired, because it states what was true at launch. No `CHECK` on the shape: Postgres could validate structure but not meaning, and a malformed diagnostic must never fail the write carrying the resolved stream values. |
| `negotiated_codec` | `TEXT` NULL | *(UI-P6, migration 0038)* the wire codec the **client** reports it is actually decoding (`getStats()` mimeType, normalised at ingest via `POST /v1/sessions/{id}/stats` `codec_mime_type`). Sits beside `codec` — what the **server** resolved — so an operator can spot a silent fallback or a mis-negotiated m-line. A per-**session** fact, not a time series, which is why it is here and not in `session_metrics`: that dictionary is numeric by contract, and a string that changes at most once per session does not belong in a per-sample series pruned on a rolling window. **Deliberately NOT constrained to the `codec` CHECK set**: it records what the receiver said, including a codec the server never resolves (`vp9`), and that disagreement is the loudest signal this column exists to surface. Length and character set are bounded in Go at ingest — untrusted client input. Written only when the value changes and only while the session is non-terminal, so a late flush from a torn-down client cannot rewrite history. |
| **reservation** | | what was reserved on assign. |
| `reserved_vram_mb` | `INT` NOT NULL DEFAULT `0` | **DEPRECATED (#383)** — written as `0` for sessions created after the live-VRAM change; historical rows keep their values. Encode slots are now the only reservation dimension. |
| `reserved_encode_slots` | `INT` NOT NULL DEFAULT `0` | |
| **timestamps** | | |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |
| `assigned_at` | `TIMESTAMPTZ` NULL | |
| `started_at` | `TIMESTAMPTZ` NULL | entered `running`. |
| `ended_at` | `TIMESTAMPTZ` NULL | entered `stopped`/`failed`. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | bumped on every state change. |

Indexes: `(user_id, created_at DESC)` for a user's session list; `(host_id)` and `(gpu_id)`
for the availability sums; partial index `(gpu_id) WHERE state IN ('assigned','starting','running','stopping')`
to make the reservation sum cheap.

## `session_tokens`
Repeatable issuance with single-use consumption for launch and mid-session reconnection.

```sql
CREATE TABLE session_tokens (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id   UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    token_hash   TEXT NOT NULL UNIQUE,
    expires_at   TIMESTAMPTZ NOT NULL,
    consumed_at  TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX session_tokens_session_created_idx
    ON session_tokens (session_id, created_at DESC);
```

Every token remains hashed, short-lived, and atomically single-use. Expired rows may be deleted by
normal maintenance. The legacy three token columns on `sessions` are migrated into this table and
then removed.

## `admin_activity`

Append-only administrative audit history. `actor_user_id` is nullable so deletion of a user does
not erase history; the UUID value is retained without a foreign key. `details` is bounded sanitized
JSONB and must never contain credentials or signaling payloads.

```sql
CREATE TABLE admin_activity (
    id            BIGSERIAL PRIMARY KEY,
    actor_user_id UUID,
    action        TEXT NOT NULL,
    target_type   TEXT NOT NULL,
    target_id     TEXT,
    details       JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX admin_activity_created_idx ON admin_activity (created_at DESC, id DESC);
```

---

## `session_metrics` (P4-01)
> *Additive, observability amendment (migration 0003). A new append-only table; it changes
> no existing table, column, type, or the session state machine. Telemetry is a cache of
> reporter truth for troubleshooting — never the access control, never a session-state
> authority.*

Per-session performance telemetry, append-only time-series, **one row per sample per
source**. Two independent reporters write here: the **agent** (`agent-api.md`
`session_metrics`, host-observable encode numbers) and the **browser** (`control-api.md`
`POST /v1/sessions/{id}/stats`, its own `getStats()` — RTT, jitter-buffer, decode time,
packet loss, frame drops). The `source` column keeps them distinct so the admin surface can
reconcile a host→browser timeline (`P4-05`/`P4-06`).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `session_id` | `UUID` NOT NULL → `sessions(id)` ON DELETE CASCADE | the session this sample belongs to; cascade so a deleted session takes its metrics with it. |
| `source` | `TEXT` NOT NULL | `CHECK (source IN ('agent','browser','native'))` *(P9-01, migration 0014 widens this from `('agent','browser')`)*. Which reporter produced the sample. |
| `ts_unix_ms` | `BIGINT` NOT NULL | reporter wall-clock of the sample (ms since epoch), same convention as `agent-api.md` `heartbeat.ts_unix_ms`. |
| `metrics` | `JSONB` NOT NULL DEFAULT `'{}'` | the sample payload (the field dictionary below). JSONB (not frozen columns) so a new metric is not a migration; no consumer reserves a fixed shape — but the **staged glass-to-glass budget keys are enumerated** so two implementers cannot invent divergent names. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | **server-only** ingestion time (distinct from `ts_unix_ms`, the reporter's clock); not surfaced by the admin read. |

Index: `(session_id, ts_unix_ms DESC)` — serves "latest N samples for a session" (the admin
read) and the per-source ordering cheaply.

**`metrics` field dictionary.** All keys optional (a reporter sends what it has); units are in
the key suffix (`_ms`, `_kbps`). The `source` column scopes which set applies.
- **`source='agent'` (host-observable):** `fps`, `bitrate_kbps`, `encode_ms`,
  `encode_ms_p50`, `encode_ms_p95`, `frames_encoded`, `frames_dropped` (encoder-side),
  `source_fps`, `compositor_fps`, `compositor_pts_delta_p50_ms`, `compositor_pts_delta_p95_ms`,
  `interpipe_queue_level_max`, `interpipe_queue_dwell_p50_ms`,
  `interpipe_queue_dwell_p95_ms`, `interpipe_queue_drops`, `rtp_fps`, and
  `rtp_bitrate_kbps`. The admin UI joins these bounded per-window stage signals as the
  host-side portion of the glass-to-glass timeline.
- **`source='browser'` (`getStats()`-derived):** `fps`, `bitrate_kbps`, `rtt_ms`,
  `jitter_buffer_ms`, `decode_ms`, `packets_lost`, `frames_dropped` (receiver-side),
  `freeze_count`, `display_refresh_hz`,
  the **presentation-pacing** keys (`#108`, always-on, `requestVideoFrameCallback`-derived):
  `present_fps` (distinct frames presented to the display per second),
  `present_interval_sd_ms` (σ of frame-to-frame *presentation* intervals — the headline
  smoothness/judder metric: a clean 60 fps stream onto a 60 Hz display reads ~2 ms when smooth,
  rising sharply when the playout buffer is too tight to keep a frame ready for every vsync),
  `present_interval_p95_ms`, and `playout_target_ms` (the receiver jitter-buffer/playout target
  the sample was measured under, so a stored σ correlates with its setting); and the
  **RVFC capture-to-display estimate** when a valid `captureTime` exists. The capability marker
  `rvfc_capture_time_available` is `1` only after such a valid sample; strict
  `abs_capture_time_negotiated` remains `0` until SDP/RTP-extension wire proof lands. The legacy
  staged keys (`glass_to_glass_ms`, **`network_pacing_ms`**, **`jitter_buffer_ms`**, and
  **`decode_display_ms`**) must not be interpreted as strict abs-capture-time G2G. They form the
  qualified browser-only bar when `rvfc_capture_time_available=1`; valid-captureTime staleness
  clears that marker and stops the staged keys even if null/invalid RVFC callbacks continue.
  `decode_display_ms` is an **unattributed residual**, so the
  bar closes as `network_pacing_ms + jitter_buffer_ms + decode_display_ms ≤ glass_to_glass_ms`.
  Agent `encode_ms` is independently sampled and must not be added without correlated trace proof.
- **`source='native'` (P9-01, the native client via `client: "native"`):** the same
  receiver-side key set as `source='browser'` (`fps`, `rtt_ms`, `jitter_buffer_ms`,
  `decode_ms`, `packets_lost`, `frames_dropped` (receiver-side), the presentation-pacing
  keys, and the always-on staged glass-to-glass budget). Values the browser cannot
  expose — **true hardware-decode** and real jitter-buffer depth — ride the **capability
  report** (`native-client.md`, `user_devices.capabilities`), not per-sample. The producer
  + migration 0014 land in **P9-07**.
- **The dictionary is NUMERIC.** A reporter fact that is a string has no home inside `metrics`
  and travels as a **sibling field on the sample** instead: `client_health` /
  `client_health_reason` / `device_key` (AS10-11) and `codec_mime_type` (UI-P6). `is_hidden` is
  the shape of the rule rather than an exception — tab visibility rides *in* the dictionary
  because it can be expressed as `0`/`1`. Sibling fields are not stored in `metrics`; each has
  its own destination (`codec_mime_type` normalises into `sessions.negotiated_codec`).
- **Cross-source note:** `frames_dropped` exists under **both** sources with **different**
  meaning (encoder-side vs receiver-side). It is disambiguated by `source`; a merged
  `latest_metrics` (`control-api.md`) **must** keep them source-scoped (namespace or pick by
  source), never blind-overlay one key over the other.

> **Retention (bounded — load-bearing).** This table must not grow without bound. The policy
> (implemented in `P4-05`, tunable there): (1) the control plane keeps only a **rolling
> recent window per session** while it runs (prune samples older than a window — e.g. the
> last hour — so a long-lived session has bounded rows); and (2) on a session reaching a
> **terminal** state, its metrics are retained for a short TTL for post-hoc troubleshooting,
> then deleted (or dropped immediately via the `ON DELETE CASCADE` if the session row is
> reaped). The admin "recent history" view (`P4-06`) is always a bounded window, never the
> full history. No metric write is on the session hot path — ingestion is decoupled from the
> lifecycle transaction.

## `user_devices` (P4-01)
> *Additive amendment (migration 0004). A new table, owner-scoped; it changes no existing
> table. Built here because login is the natural capture point; **Phase 7 surfaces and
> manages it** (device list, trust posture) — Phase 4 only writes it. Privacy: `device_key`
> is an app-scoped opaque id the client generates, **not** a hardware fingerprint or
> cross-site identifier (see below).*

One row per `(user, device)` — the per-device connection + decode-capability probe captured
at login (`control-api.md` `POST /v1/me/devices`). The data source for later per-user/-device
settings (FPS tiering, codec selection); **Phase 4 produces it, no optimizer consumes it.**

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `user_id` | `UUID` NOT NULL → `users(id)` ON DELETE CASCADE | owner. |
| `device_key` | `TEXT` NOT NULL | a **stable, privacy-preserving** per-device id the client generates once (random UUID) and persists in `localStorage`; app-scoped, **not** derived from hardware and **not** a cross-site supercookie. Clearing browser storage resets it (a new device row results) — acceptable; this is best-effort device grouping, not identity. |
| `user_agent` | `TEXT` NULL | last-seen UA string (mirrors `auth_tokens.user_agent`). |
| `capabilities` | `JSONB` NOT NULL DEFAULT `'{}'` | the probe result: `{ "codecs": {"h264":true,"hevc":false,"av1":false,"vp9":true}, "max_decode_height": 2160, "bandwidth_kbps": 48000, "rtt_ms": 12, "measured_at": "<rfc3339>" }`. JSONB so the probe can grow without a migration. `measured_at` is **server-stamped** at upsert (not client-supplied, so a client cannot backdate). **Note:** browsers do not directly expose *hardware*-decode support; `codecs` is a best-effort heuristic (codec acceptance via `RTCRtpReceiver.getCapabilities` + a resolution probe) and is documented as such in `P4-08`. |
| `first_seen_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | first login from this device. |
| `last_seen_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | updated on each capability upsert. |
| `name` | `TEXT` NULL | *(LP-SEC-01, migration 0020)* user-set display label ("Living-room PC"). NULL = unnamed; the UI falls back to UA/first-seen. |
| `trusted` | `BOOLEAN` NOT NULL DEFAULT `false` | *(LP-SEC-01, migration 0020)* user trust posture. **Advisory metadata for the account UI — not an authorization input in W1.** Any future step-up/skip behaviour keyed on it is a separate, sign-off-gated change. |

Constraints: `UNIQUE (user_id, device_key)` — the upsert key (insert on first sight, update
`capabilities`/`last_seen_at`/`user_agent` thereafter). Index `(user_id)` for listing a
user's devices (the Phase-7 surface).

---

## `session_trace_events` (Observability v2, ST-01)
> *Additive, observability amendment (migration 0016). A new append-only table; it changes no
> existing table, column, type, or the session state machine. Discrete per-session markers
> (events) that complement the periodic `session_metrics` samples; the diagnostic bundle
> (`control-api.md`) reads existing `session_metrics` JSONB **joined with** this events table —
> there is no new samples table and no second sample write path. Telemetry is a cache of
> reporter truth for troubleshooting — never the access control, never a session-state
> authority.*

Per-session discrete events, append-only, **one row per event per source**. Two reporters write
here: the **agent** (`agent-api.md` `session_trace_event`, host-side markers — ABR retarget,
source swap, encoder drop, webrtc state) and the **browser** (`control-api.md` trace-events
ingest — playout change, freeze, visibility, client webrtc state). The `source` column keeps
them distinct so the admin surface can reconcile a host→browser timeline (the same pattern as
`session_metrics.source`).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `session_id` | `UUID` NOT NULL → `sessions(id)` ON DELETE CASCADE | the session this event belongs to; cascade so a deleted session takes its events with it (identical to `session_metrics`). |
| `source` | `TEXT` NOT NULL | `CHECK (source IN ('agent','browser'))`. Which reporter produced the event. (No `'native'` value in v1 — the native client rides the browser ingest as `source='browser'`; widening, if ever wanted, is a later migration, mirroring how `session_metrics.source` widened in 0014.) |
| `ts_unix_ms` | `BIGINT` NOT NULL | reporter wall-clock at the event (ms since epoch), same convention as `session_metrics.ts_unix_ms` / `agent-api.md` `heartbeat.ts_unix_ms`. |
| `type` | `TEXT` NOT NULL | the event type from the `trace-format.md` §3 v1 allow-list (`abr.retarget`, `pipeline.source_swapped`, `encoder.drop_detected`, `webrtc.state_changed`, `playout.changed`, `client.freeze_detected`, `client.visibility_changed`; plus the reserved `operator.annotation`). Browser-source unknown types are dropped at ingest (never stored). |
| `payload` | `JSONB` NOT NULL DEFAULT `'{}'` | the per-type payload (`trace-format.md` §3). JSONB (not frozen columns) so a new event field is not a migration. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | **server-only** ingestion time (distinct from `ts_unix_ms`, the reporter's clock); used by the retention prune. |

Indexes:
- `(session_id, ts_unix_ms DESC)` — serves "recent events for a session" (the trace/bundle read)
  and per-session ordering, mirroring `session_metrics_session_ts_idx`.
- `(session_id, type, ts_unix_ms DESC)` — serves the typed read (`.../trace/events?types=`) and
  the bundle's per-type derived windows.

**Plain Postgres only — no `CREATE EXTENSION`, no `create_hypertable`.** A plain relational table
on the existing retention/prune model (`trace-format.md` §6): rolling per-session window
(`metricsRetentionWindow`) + terminal prune + FK cascade.

> **Retention.** Same posture as `session_metrics`: a rolling recent window per session while it
> runs, a short post-terminal TTL, then delete (or immediate cascade on session-row reap). No
> event write is on the session hot path.

## `session_trace_clock` (Observability v2, ST-01)
> *Additive, observability amendment (migration 0016, same migration as `session_trace_events`).
> One optional row per session carrying the client↔host clock-offset estimate **and its
> uncertainty**, so the browser-clock series can be aligned against the host-clock series
> honestly. Absence of a row means the clock is **unmeasured** — never interpreted as offset 0.*

| column | type | notes |
|---|---|---|
| `session_id` | `UUID` PRIMARY KEY → `sessions(id)` ON DELETE CASCADE | the session this clock estimate belongs to. PK so there is exactly one row per session. Cascade-reaps with the session. |
| `client_offset_ms` | `DOUBLE PRECISION` NOT NULL | estimated client-clock − host-clock offset (ms); add to a browser `ts_unix_ms` to express it on the host clock. From the deep-trace ping/pong sync (ST-05). |
| `uncertainty_ms` | `DOUBLE PRECISION` NOT NULL | the spread of the offset estimate (e.g. min-RTT-derived) — the honest error bar carried on every cross-clock alignment. |
| `measured_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | when the offset was measured. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | last write (a re-measure refinement updates this). |

**Unmeasured is the absence of the row.** A session for which clock sync never succeeded has
**no** `session_trace_clock` row; the bundle reports `clock: { "unmeasured": true }`. There is no
sentinel `client_offset_ms = 0` — offset 0 is a measured value, not a default (`trace-format.md`
§4, no false precision). v1 stores **one** offset per session.

Migration `0016_session_trace`: creates both tables + the two `session_trace_events` indexes in
one `BEGIN; … COMMIT;` block. The down migration drops them in reverse order.

---

## `host_settings` (native-client-spike-186 / host-runtime-settings-admin)
> *Additive amendment (migration 0010). A new table; it changes no existing table, column,
> type, or constraint. Stores per-host sparse overrides for the server-side knob catalog;
> the JSONB column means future knobs are catalog-only additions (no migration per new knob).*

One row per host that has **at least one knob override**. Hosts with no overrides simply have
no row — the control plane resolves every knob to its catalog default in that case.

| column | type | notes |
|---|---|---|
| `host_id` | `UUID` PRIMARY KEY → `hosts(id)` ON DELETE CASCADE | the host these overrides belong to. PK so there is exactly one overrides row per host. Cascade-deletes automatically when the host is removed. |
| `overrides` | `JSONB` NOT NULL DEFAULT `'{}'` | **sparse** map of catalog knob keys → values. Absent keys resolve to the catalog default at read time. The JSONB is **validated server-side against the hostcfg catalog on every PATCH** and is never trusted blindly — unknown keys, out-of-range values, and wrong types are rejected before the row is written. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | last modification time; maintained by the application. |
| `updated_by` | `UUID` NULL → `users(id)` | the admin user who last changed the overrides. NULL until the first admin write (e.g. pre-populated by a migration). No cascade — the admin's identity is kept for the audit trail even after the admin user is deleted. |

Migration: `0010_host_settings.up.sql` — `CREATE TABLE host_settings (...)`. The down migration drops the table.

> **Knob catalog (authoritative in the control plane, not in the schema).** The catalog is a
> server-side typed manifest — each entry carries: key, type (`bool`/`int`/`float`/`enum`/`string`),
> default (equals today's env-var default), optional range/enum constraints, the corresponding env
> var name, and a class (`live` or `restart`). The control plane exposes the catalog to the UI via
> `GET /v1/admin/config/catalog` (`control-api.md`). **Live-class knobs** take effect on the next
> session launch (no restart needed). **Restart-class knobs** (`encoder`, `render_node`,
> `cuda_device`) are read once at the agent's first session build (`gst::init` is a process-wide
> `Once`); changing them requires an agent restart via the `restart` downstream message (`agent-api.md`).
> Precedence: catalog default → per-host override; env vars are the agent's bootstrap fallback only.
>
> *(storage-config amendment, catalog-only addition — no schema change, per the JSONB design
> above.)* The catalog gains **`home_root`** (`string`, live-class, env `QUASAR_HOME_ROOT`,
> default empty): the absolute host directory holding managed homes (P5). The **control plane**
> resolves a session host's *effective* home root as stored-override → the host's last-reported
> `effective_settings.home_root` (the agent's env baseline) → the control plane's own
> `QUASAR_HOME_ROOT` env (legacy fallback) when synthesizing local-driver home paths, so
> different hosts may use different roots. The **agent** uses the same overlaid value for its
> post-session home-usage measurement (`metrics.bytes_used`); an empty effective root disables
> measurement (volume-driver semantics).

---

## `console_config` (CM-01 / console-mode)
> *Additive amendment (migration 0022). A new table; it changes no existing table, column,
> type, or constraint. Stores per-host console-mode configuration (local display + local
> audio + local input for "use the host like a console"). Mirrors the `host_settings`
> pattern — per-host JSONB, validated server-side — but is a distinct structured surface
> (lists, nested selectors) rather than the flat scalar knob catalog. Requires sign-off.*

One row per host that has console mode configured. A host with no row = console disabled
(the control plane resolves the all-default object). Delivered to the node-agent additively
via `config_update.console_config` (`agent-api.md`); the agent reads it **instead of** the
spike's `QUASAR_LOCAL_DISPLAY` env hardcode.

| column | type | notes |
|---|---|---|
| `host_id` | `UUID` PRIMARY KEY → `hosts(id)` ON DELETE CASCADE | one row per host; cascades on host removal |
| `config` | `JSONB` NOT NULL DEFAULT `'{}'` | the console-config object (below). Sparse — absent keys resolve to defaults at read time. **Validated server-side on every PATCH** (unknown keys / bad types / bad enum / non-reported device rejected), never trusted blindly — same discipline as `host_settings.overrides`. |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | app-maintained |
| `updated_by` | `UUID` NULL → `users(id)` | last admin to write; no cascade (audit trail) |

Migration: `0022_console_config.up.sql` — `CREATE TABLE console_config (...)`. Down drops it.

Console capability is host-scoped, but output topology is session-scoped through
`session_assign.video_topology` (`stream_only` | `local_only` | `dual_output`). This prevents an
enabled console host from mirroring unrelated browser sessions. A local-only console launch has
zero reserved encode slots and no signaling-token row; dual-output retains both reservations.

Wave 3.2 adds `console_capabilities.outputs`, a typed per-card DRM connector/mode inventory. It
preserves card/render association and exact millihertz timing identity rather than flattening
connector names. The existing `connectors` array remains an additive compatibility projection.

**The `config` object (resolved shape + defaults).** Console-mode is **local-only by
default** (`stream:false`) and **off by default** (`enabled:false`):
```json
{
  "enabled": false,                 // master switch; false = no local leg (today's behavior)
  "connector": "auto",              // "auto" | "DP-4" | "HDMI-A-1" | ...  (display output; validated vs reported connectors)
  "output_id": null,                 // card-scoped id from console_capabilities.outputs
  "mode": null,                      // {width,height,refresh_millihz}; exact reported mode for output_id
  "compositor": "weston",           // "weston" | "cage"  (CM-04 may flip the default to cage)
  "audio_output": null,             // LOCAL host sink; no default — null ⇒ quiet until an admin picks one. "auto"(GPU HDA of the active connector) | "<alsa hw id>" | "hdmi" | "motherboard" | "usb:<...>". Independent of `stream`.
  "stream": false,                  // LOCAL-ONLY DEFAULT: also stream this session over WebRTC when true (dual-output)
  "stream_audio": false,            // SEPARATE from audio_output: also run the WebRTC Opus leg (only meaningful when stream=true)
  "input_devices": "auto",          // "auto"(enumerate connected) | ["/dev/input/eventN", ...] (validated vs reported input devices)
  "grab": true,                     // EVIOCGRAB exclusive-grab the physical devices to the session
  "auto_start_on_display": false,   // CM-06: auto-launch default_app when a display connects on `connector`
  "auto_connect_controller": false, // CM-07: auto-attach+grab a hotplugged controller
  "default_app": null,              // UUID of the app to auto-launch on the console, or null (FK-checked vs apps.id)
  "default_user": null,             // CM-06: UUID → users.id — OWNER of auto-started console sessions (admin-set; FK-checked). Required when auto_start_on_display=true (else auto-start is skipped + logged). Future: a console user-selection screen supersedes this.
  "fullscreen": true                // CM-04: fullscreen the console client
}
```
Validation: `compositor` ∈ `{weston, cage}`; `connector`/`audio_output`/`input_devices`
checked against the host's reported `console_capabilities` (`agent-api.md` `capacity`)
unless `auto`/`null`; `default_app` FK-checked against `apps(id)`; `default_user`
FK-checked against `users(id)`. `enabled:true` with
`audio_output:null` is **valid** — the console runs with no local audio until a sink is
picked (fail-safe/quiet). No volume/mute (out of scope — app/host mixer's concern).

---

## `user_homes` (P5-01)
> *Additive amendment (migration 0008). A new bookkeeping table; it changes no existing
> table semantics. The table tracks managed homes — the **backing store** (a docker volume
> or a host directory) is the authoritative data; rows here are the control plane's index
> of it (where it lives, how big it is, whether it awaits GC).*

One row per provisioned per-(user, app) home, **pinned to the host that holds it** (v1
homes are host-local; a shared-storage driver later relaxes `host_id`).

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()` |
| `user_id` | `UUID` **NULL** → `users(id)` **ON DELETE SET NULL** | *P5-05 erratum to P5-01: changed from NOT NULL/no-cascade.* NULL means the user was deleted and the row is orphaned pending GC. User deletion tombstones the row (sets `gc_after`) **before** the user row is deleted; the FK then sets this to NULL. After the grace period the row (and its backing store) is removed per the GC lifecycle note below — by the agent confirm if still host-pinned, else by the janitor. |
| `app_id` | `UUID` **NULL** → `apps(id)` **ON DELETE SET NULL** | *P5-05 erratum to P5-01: changed from NOT NULL/no-cascade.* Same orphan-pending-GC rule on app deletion. |
| `host_id` | `UUID` NULL → `hosts(id)` **ON DELETE SET NULL** | the host holding the backing store. NULLable for future shared-storage drivers (a shared home belongs to no single host). *(admin-delete erratum, migration 0014: was no-cascade.* Forgetting a host tombstones its homes (`gc_after`) **before** the host row is deleted; the FK then NULLs `host_id` and the janitor reaps the orphan via the `host_id`-NULL GC path.)* |
| `provider` | `TEXT` NOT NULL | `CHECK (provider IN ('volume','local'))` — extended by future driver amendments, never repurposed. |
| `ref` | `TEXT` NOT NULL | provider-scoped locator: the docker volume name (`volume`) or the host path under the agent's `QUASAR_HOME_ROOT` (`local`). |
| `bytes_used` | `BIGINT` NOT NULL DEFAULT `0` | last reported usage (the agent measures post-session; `volume`-driver usage is measured the same way via the mounted path). Advisory freshness — quota is enforced against the last report. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | |
| `last_used_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | touched on session end. |
| `gc_after` | `TIMESTAMPTZ` NULL | non-null = tombstoned; after a 24h grace period the row is removed — see the GC lifecycle note below for *who* removes it (control-plane janitor vs. agent confirm). |

Unique: `(user_id, app_id, host_id)` — one home per (user, app) per host.
Index: `(user_id)` (per-user usage summation); `(gc_after)` partial `WHERE gc_after IS NOT NULL` (janitor sweep).

> **GC lifecycle (#175 note).** A tombstoned row (`gc_after` set) lives out a 24h grace period,
> then its **backing store and row** are removed by one of two paths depending on `host_id`:
> - **`host_id` set (host-pinned).** The home's backing store sits on a specific host, and only
>   that host's node-agent can remove it (the control plane has no host/docker access —
>   invariant #1). The agent **pulls** these past-grace rows (`GET /v1/agent/storage/gc-pending`,
>   `control-api.md §Agent storage GC`), reaps the docker volume / directory host-side, and
>   **confirms** (`POST …/gc-confirm`) — the confirm is what hard-deletes the row, *after* the
>   backing store is gone. The control-plane janitor deliberately **leaves these rows alone**.
> - **`host_id IS NULL` (unpinned / host removed).** No agent owns the backing store, so there
>   is nothing to reap and nobody to confirm. The control-plane **janitor** hard-deletes these
>   rows directly after grace (a row-only delete — the pre-#175 status quo, when the janitor
>   deleted every past-grace row regardless).

---

## `host_encoder_certification` (Stream Perf Tuning Phase C, SPT-05)
> *Additive, scheduling-support amendment (migration 0018). A new table; it changes no existing
> table, column, type, or constraint, and it does **not** touch the session state machine. It
> records the measured, sustainable performance envelope of a concrete encode configuration on a
> concrete GPU — the per-(host, GPU, encoder, profile) answer to "can this box hold this rung in
> real time?". It is **scheduling input, not telemetry**: unlike `session_metrics` (a rolling
> per-session cache the retention prune reaps), a certification row is a durable capability fact
> about the host that the scheduler reads at launch and that survives across sessions. It is
> never access control and never a session-state authority.*

One row per **certified configuration**: a (host, GPU, encoder, profile/resolution+fps) tuple
plus the bitrate the bench was run at. Produced by the SPT-06 certification routine, which runs a
short bench encode at the candidate rung and records how the encoder behaved against the frame
budget. The scheduler reads the **latest** row per tuple to decide whether a profile is safe to
default-start on that host.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PK | `gen_random_uuid()`. |
| `host_id` | `UUID` NOT NULL → `hosts(id)` ON DELETE CASCADE | the host the bench ran on; cascade so forgetting a host reaps its certifications (identical posture to `gpus`/`session_metrics`). |
| `gpu_index` | `INT` NOT NULL | the GPU index on that host (matches `gpus.index`; **not** an FK to `gpus.id` — certification is keyed by the stable host+index the agent reports, so a GPU row re-creation does not orphan a still-valid measurement). |
| `encoder` | `TEXT` NOT NULL | the encode element family the verdict is for — `CHECK (encoder IN ('va','nvenc','openh264'))`. Matches the `QUASAR_ENCODER` selector. Widening (e.g. an AV1 element) is a later migration, mirroring how `session_metrics.source` widened in 0014. |
| `profile_id` | `TEXT` NOT NULL | the LAUNCH PROFILE the bench ran under (e.g. `'1080p60'`). Same id space as `sessions.profile_id`; **not** FK-constrained. **CONTEXT, NOT KEY since 0041** — a launch profile is a chain of rungs and has no single encode cost, so it identifies the run, not the measurement. |
| `stream_profile_id` | `TEXT` NOT NULL → `stream_profiles(id)` ON DELETE CASCADE | **(migration 0041)** the RUNG that was streamed and measured, i.e. one concrete (resolution, fps, codec). This is the key dimension the verdict is actually about, and what `sessions.stream_profile_id` records for a live session. `CASCADE` (not the `NO ACTION` `sessions.stream_profile_id` uses) because a session row is history that must survive its rung while a certification row is a measurement *of* the rung and is meaningless without it. |
| `width` | `INT` NOT NULL | resolved bench resolution (self-describing even if the profile catalog later re-tunes a rung). |
| `height` | `INT` NOT NULL | |
| `fps` | `INT` NOT NULL | the bench target fps — the frame budget is `1000.0 / fps` ms. |
| `bitrate_kbps` | `INT` NOT NULL | the CBR/ceiling bitrate the bench ran at (one row per (rung, bitrate) point). |
| `verdict` | `TEXT` NOT NULL | `CHECK (verdict IN ('ok','capped','unsafe'))`. `ok` = sustainable at this rung; `capped` = sustainable only with a live downshift / not at full fps; `unsafe` = cannot hold the frame budget (the Renoir-1080p60 motivating case). |
| `encode_ms_p50` | `DOUBLE PRECISION` NOT NULL | measured encode-time p50 over the bench window (agent `encode_ms` series). |
| `encode_ms_p95` | `DOUBLE PRECISION` NOT NULL | encode-time p95 — **the headline number the scheduler rule keys on**. |
| `encode_ms_max` | `DOUBLE PRECISION` NOT NULL | worst observed encode time in the window. |
| `output_fps` | `DOUBLE PRECISION` NOT NULL | the actual sustained encoder output fps during the bench (distinguishes "asked for 60, delivered 45"). |
| `drop_rate` | `DOUBLE PRECISION` NOT NULL | encoder-side dropped-frame fraction over the window, `[0.0, 1.0]`. |
| `live_write_stable` | `BOOLEAN` NOT NULL | whether live bitrate/CPB property writes applied cleanly on this encoder during the bench (the same writability probe ABR uses to arm). `false` ⇒ ABR cannot live-adapt this encoder, so the scheduler/ABR must treat the rung as static CBR. |
| `sample_window_ms` | `INT` NOT NULL | the bench measurement window length (ms) the percentiles were computed over. |
| `sample_count` | `INT` NOT NULL | number of `encode_ms` samples in the window (no false precision off a short run). |
| `agent_version` | `TEXT` NULL | `hosts.agent_version` at measurement time. |
| `measured_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | when the bench ran. The scheduler may treat a row older than a staleness horizon as "uncertified". |
| `updated_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | last write (an upsert refresh bumps this). |

**Upsert-latest, not append-only.** Unlike `session_metrics`, this table holds the **current
capability verdict** per configuration. A re-certification of the same (host, gpu_index, encoder,
stream_profile_id, bitrate_kbps) tuple **replaces** the prior row (upsert on the unique key below,
bumping `updated_at`/`measured_at`). `profile_id` rides along and is refreshed on conflict but is
not part of the key: one rung may be listed by more than one chain, and re-certifying it under a
different chain must update the single measurement rather than fork it. Uniqueness:

```
UNIQUE (host_id, gpu_index, encoder, stream_profile_id, bitrate_kbps)   -- migration 0041
```

Index: `(host_id, gpu_index, encoder, stream_profile_id)` — serves the scheduler's per-launch
lookup across all bench bitrates, and the admin per-host read.

**Why the key is the rung and not `(profile_id, codec)` (migration 0041).** The 0018 key had no
codec dimension at all: `encoder` is the backend *family* (`va`/`nvenc`/`openh264`/`vulkan`), not
the codec. Encode cost is codec- **and** resolution-dependent, and since the Phase-4 restructure a
launch profile chains rungs for up to all three codecs — `launch_profile_rungs` is unique on
`(launch_profile_id, stream_profile_id)`, so a chain may legally hold two rungs of the same codec
at different resolutions. `(profile_id, codec)` is therefore only unambiguous while every chain is
single-resolution, which the model does not promise. This is the same conclusion Phase 4 reached
for the other wrong-grained per-profile table: migration 0032 re-keyed
`user_device_profile_history` on `(profile, codec)`, and Phase 4 then moved decode-failure history
to rung grain for exactly this reason.

**The 0041 data migration.** Every pre-0041 row is an H.264 measurement — the SPT-06 bench sets no
codec on its bench session and the `sessions` INSERT coalesces an empty codec to `h264` — so rows
are re-pointed at the **first h264 rung by position** of the launch profile they name. 0036's
fan-out cloned every column of the legacy stream profile into that rung, so a migrated row's
`width`/`height`/`fps` still match its new key exactly. A row naming no launch profile with an
h264 rung cannot be interpreted at the new grain and is **deleted** (a missing cert is the
optimistic "uncertified" case, so dropping one can only remove a cap, never add one); the whole
pre-migration table is snapshotted to `_backup_0041_host_encoder_certification` first, and the down
path restores from it. The down direction is lossy where two rungs of one chain were certified at
the same bitrate — the 0018 key cannot represent both — and resolves it by keeping the newest
measurement, which is that key's own upsert-latest rule.

**Plain Postgres only — no `CREATE EXTENSION`, no `create_hypertable`.** There is no time-series
growth concern because it is upsert-latest (bounded by hosts × GPUs × encoders × rungs × bench
bitrates).

**The `verdict` enum (sustainability classification).** Derived from the bench window against
`budget_ms = 1000.0 / fps`: **`ok`** — `encode_ms_p95 ≤ 0.70 × budget_ms` **and** `output_fps ≥
0.97 × fps` **and** negligible `drop_rate`; **`capped`** — `encode_ms_p95 ∈ (0.70, 1.0] ×
budget_ms`, or `output_fps` short of target while drops stay low; **`unsafe`** —
`encode_ms_p95 > budget_ms`, or `output_fps` materially below target, or a non-trivial
`drop_rate`. The `0.70 × budget` threshold is the same constant the scheduler's session-start cap
uses (`control-api.md` §Host encoder certification) — the enum values and the `0.70`-budget
boundary are the contract; the exact epsilon/`output_fps` cutoffs are SPT-06 implementation
tuning.

Migration `0018_host_encoder_certification`: creates the table + the lookup index in one
`BEGIN; … COMMIT;` block. The down migration drops the table.
Migration `0019_encoder_cert_vulkan`: widens the `encoder` CHECK to admit `'vulkan'`.
Migration `0041_cert_rung_key`: adds `stream_profile_id`, migrates the existing rows onto it,
swaps the unique key and the lookup index onto the rung, and snapshots the pre-migration table for
its down path.

## `user_app_favourites` (UI-P1)
> *Additive amendment (migration 0034). A new join table; it changes no existing table
> semantics. It is **presentation state** — which apps a user pinned in their library — and is
> never access control, never a launch input, and never a session authority.*

One row per (user, app) favourite. **The presence of the row is the fact**: there is no
`favourited` boolean to keep in sync and no soft delete, so unfavouriting is a row delete and
the table can never hold a "false" row that a reader must interpret.

| column | type | notes |
|---|---|---|
| `user_id` | `UUID` NOT NULL → `users(id)` **ON DELETE CASCADE** | the owner. Always the bearer identity at the API (`control-api.md` §Favourites) — no endpoint accepts a `user_id`, so cross-user isolation is structural. |
| `app_id` | `UUID` NOT NULL → `apps(id)` **ON DELETE CASCADE** | the favourited app. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | when it was favourited. **Not exposed on the wire** — kept so a "recently favourited" ordering is a query change, not a future migration. A repeat `PUT` is `ON CONFLICT DO NOTHING` and therefore does **not** re-stamp it. |

```
PRIMARY KEY (user_id, app_id)
```

The composite primary key is **both** the uniqueness constraint and the **idempotency key**:
`PUT /v1/me/favourites/{app_id}` is `INSERT … ON CONFLICT (user_id, app_id) DO NOTHING`, so
favouriting twice is one row and one `204`.

Index: `(app_id)`. Postgres indexes the primary key (hence the `(user_id, …)` leading edge for
the per-user library join) but does **not** auto-index the *referencing* side of a foreign key —
so without this index the `apps` cascade delete has to sequential-scan this table per deleted
app. `DELETE /v1/apps/{id}` is a real operator path, so the index is part of the contract.

**Both FKs are `ON DELETE CASCADE`, deliberately not `SET NULL`** — note this differs from the
neighbouring `user_homes.user_id` / `user_homes.app_id`, which *are* `SET NULL`. The difference
is that `user_homes` guards a **backing store**: a home row must survive its user's or app's
deletion long enough for the GC path to reap the volume/directory it points at, so it is
orphaned-then-tombstoned rather than deleted outright. A favourite guards nothing — there is no
backing store to reap and no meaning to preserve in "somebody once favourited an app that no
longer exists". It is derived presentation state whose only consumer is a per-user library view,
so the correct behaviour on either parent's deletion is for the row to disappear.

## `app_launch_profiles` (UI-P5)
> *Additive amendment (migration 0037). A new join table; it changes no existing table
> semantics. It is **stream-quality curation** — which launch profiles an app offers from the menu
> beside Play — and is **never an authorization boundary**: it can only narrow what eligibility
> already permits.*

One row per (app, offered launch profile). **The presence of rows is the fact**: there is no
"restricted" boolean to keep in sync, so an app cannot be in a state where a flag says restricted
and no row says which.

| column | type | notes |
|---|---|---|
| `app_id` | `UUID` NOT NULL → `apps(id)` **ON DELETE CASCADE** | the app. The list is a property *of* the app and has no meaning once the app is gone — same posture as `user_app_favourites`. |
| `launch_profile_id` | `TEXT` NOT NULL → `launch_profiles(id)` **ON DELETE CASCADE** | the offered launch profile. **The side that had a real choice.** `RESTRICT` would make retiring a launch profile harder the more carefully an operator had curated their apps; `DELETE /v1/admin/launch-profiles/{id}` already refuses (`409`) for the three references that would leave something pointing at nothing, and an allow-list entry is not one of those — it is a *restriction* naming a catalogue object, and removing it leaves the app fully functional. **Cost, recorded not discovered:** a cascade that empties an app's list turns it back into "unrestricted", so a delete can widen a menu. Bounded (the widened set is still only what the device is eligible for = the pre-UI-P5 behaviour) and logged (the affected apps go into the admin activity log). |

```
PRIMARY KEY (app_id, launch_profile_id)
```

The composite primary key is both the uniqueness constraint and what makes a duplicate id in a
write shape a non-event (the API dedupes; this refuses).

Index: `(launch_profile_id)`. Postgres indexes the primary key (giving the `(app_id, …)` leading
edge the per-app launch read needs) but does **not** auto-index the *referencing* side of a foreign
key — so without it **both** cascade deletes above sequential-scan this table, as does the
"which apps allow-list this profile" lookup that runs before every launch-profile delete.

**What is deliberately NOT here.** The app's own default (`apps.default_profile_id` under
`profile_policy = 'prefer'`) is **implicitly always included** and is not stored as a row: it is one
column away, and a copy would need syncing on every default change. The `'force'` exclusion is
likewise not a `CHECK` — the rule spans two tables, so it lives in the write path, which refuses to
store a list for a `force` app and clears any existing one when an app is switched to `force`.

**Empty set = today's behaviour**, which is why no backfill exists and why upgrading changes
nothing for any app.

## `entitlements` (Steam library discovery Phase 2)
> *Migration 0043. The table is new and changes no existing table; **what reads it is not
> additive** — `control-api.md`'s `GET /v1/apps` becomes entitlement-filtered for every role and
> the launch/swap paths gain a terminal `403`. Requires Opus + explicit human sign-off. Unlike
> `user_app_favourites` (presentation state) and `app_launch_profiles` (stream-quality curation,
> explicitly never an authorization boundary), **this table IS an authorization boundary.***

One row per grant. **Presence of the row is the fact**: there is no `revoked` boolean and no soft
delete, so the table can never hold a row a reader must interpret before trusting.

| column | type | notes |
|---|---|---|
| `id` | `UUID` PRIMARY KEY DEFAULT `gen_random_uuid()` | a **surrogate** key, unlike the composite-PK join tables above, because the revoke API addresses one grant by id (`DELETE /v1/admin/apps/{id}/entitlements/{entitlement_id}`) and a composite key would put a subject id in a URL. Uniqueness is carried by the two partial indexes below, not by this. |
| `subject_type` | `TEXT` NOT NULL | `CHECK (subject_type IN ('user','all'))`. `'all'` = everyone; `'user'` = one account. **Widening to `'group'` is additive**: a new `CHECK` value and a third partial unique index, no shape change — which is the reason this is a subject *type* + id and not a `user_id` column. The `TEXT` + `CHECK` convention, as elsewhere in this schema. |
| `subject_id` | `UUID` NULL → `users(id)` **ON DELETE CASCADE** | the subject, `NULL` for `'all'`. **`CASCADE`, deliberately not `SET NULL`** — a `NULL`ed `subject_id` on a `'user'` row would violate the shape `CHECK` below anyway, and a grant to a deleted account has no meaning. Same call as `user_app_favourites`. |
| `app_id` | `UUID` NOT NULL → `apps(id)` **ON DELETE CASCADE** | the app. An entitlement to an app that no longer exists has no meaning, and leaving one behind would silently re-grant access if an id were ever reused. |
| `granted_by` | `TEXT` NOT NULL | `CHECK (granted_by IN ('admin','provider','migration'))`. **Provenance, not authority** — all three grant identical access. `'admin'` = an operator granted it (audited in `admin_activity`). `'provider'` = a library-discovery sync wrote it, and a sync may revoke it — **nothing writes this yet; it is Phase 4.** It is in the `CHECK` now so Phase 4 needs no `ALTER`, and so revoking one is already a working path the day the first one is written. `'migration'` = the 0043 backfill and *only* the backfill, so an operator auditing "who made this app public" gets "it was public before entitlements existed" rather than a false attribution to whichever admin happened to act first. |
| `granted_by_user` | `UUID` NULL → `users(id)` **ON DELETE SET NULL** | who performed an `'admin'` grant. **`SET NULL` rather than `CASCADE`** — and the difference matters: deleting the operator who granted an entitlement must not silently **revoke** it. Same posture as `instance_secrets.updated_by`. |
| `source_ref` | `TEXT` NOT NULL DEFAULT `''` | free-form provenance for a `'provider'` grant (Phase 4: which scan, which appid). `''` for everything Phase 2 writes. |
| `created_at` | `TIMESTAMPTZ` NOT NULL DEFAULT `now()` | when it was granted. |

```
CONSTRAINT entitlements_subject_shape_ck
  CHECK ((subject_type = 'all') = (subject_id IS NULL))
```
The two legal shapes and nothing else: `('all', NULL)` or `('user', <uuid>)`. Written as an
**equivalence** rather than two `OR`ed clauses so it stays one readable line when `'group'` is
added — it becomes `subject_type = 'all'` on the left and nothing else changes.

```
CREATE UNIQUE INDEX entitlements_all_uk  ON entitlements (app_id)             WHERE subject_type = 'all';
CREATE UNIQUE INDEX entitlements_user_uk ON entitlements (subject_id, app_id) WHERE subject_type = 'user';
CREATE INDEX        entitlements_app_idx ON entitlements (app_id);
```

**Two partial unique indexes, not one plain `UNIQUE` — a correctness requirement, not a style
choice.** Postgres does not treat `NULL`s as equal in a `UNIQUE` constraint, so a single
`UNIQUE (subject_type, subject_id, app_id)` would consider every `('all', NULL, <app>)` row
distinct from every other and **silently permit unlimited duplicate `all` rows for the same app**.
That is not merely untidy, and the failure is quiet in a specific way worth spelling out: the read
predicate is `EXISTS`, so duplicates would *not* corrupt the library list and nothing would break
loudly — but a revoke that deletes *"the"* row would leave the app still visible, and a grant's
`ON CONFLICT` idempotency would have nothing to conflict on. Splitting by shape gives each shape a
real uniqueness key. (Postgres 15+ has `NULLS NOT DISTINCT`, which would also work; two partial
indexes are used because they additionally serve as the **lookup** indexes for the two halves of
the filter predicate and carry no version floor.)

**Both shapes may coexist for one (user, app), deliberately.** The unique indexes prevent
duplicates *within* a shape, never *across* them: "everyone may see it" and "you specifically were
granted it" are two independent facts, and revoking one must not revoke the other. The consequence
for readers is that the filter must match on **existence**, never by joining the entitlement rows
into the app query — a join emits such an app twice, and `GET /v1/apps` pages by offset with a
`limit + 1` overfetch, so a duplicated row consumes a slot, shifts the page boundary and silently
drops an app off the far end. Every single-user test passes that defect.

`entitlements_app_idx` is the app-direction lookup (`GET /v1/admin/apps/{id}/entitlements`) and the
index the `apps` cascade needs: Postgres does not auto-index the *referencing* side of a foreign
key, and `DELETE /v1/apps/{id}` is a real operator path.

**The 0043 backfill — `INSERT ... SELECT 'all', NULL, id, 'migration' FROM apps`, in the same
transaction.** This is the most load-bearing statement in the migration and the reason this table
cannot be created ahead of its readers or after them. It is described in full in the amendment
block at the top of this document; the one-line version is that **filtering an empty table blanks
every user's library on every deployment at once, and every automated gate passes while it
happens.** The backfill makes the day-one behaviour change exactly zero.

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
   `state ∈ {assigned, starting, running, stopping}` (exactly the GPU-availability-sum filter), so on
   `pending → failed` the release is a no-op (nothing was reserved) and on every other path it
   returns the held VRAM + encode slots. There is **no terminal path that leaves a reservation
   dangling.**
3. **The agent WebSocket dropping while a session is `running` is an explicit failure edge.**
   Per `agent-api.md` §Reconnection: a heartbeat-miss / lost agent connection past the threshold
   marks the host `offline` and the control-plane reaper drives **all** that host's non-terminal
   sessions (including `running` and `stopping`) to `failed`, releasing each reservation. This is
   the multi-host failure model exercised at N=1 — the reaper is the authority of last resort, so
   liveness never depends on a cooperative `session_state` callback that a dead agent can't send.
   *(P3-01)* The reaper stamps `state_detail = 'host_lost'` on these sessions so a client can
   distinguish a host-loss from an ordinary failure; `state` stays `failed` (a free-text convention
   over the existing column, like P2-02's `swapping` — no new state).
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
  0003_session_metrics.up.sql    -- (P4-01) CREATE session_metrics (append-only telemetry) + index
  0003_session_metrics.down.sql  -- DROP session_metrics
  0004_user_devices.up.sql       -- (P4-01) CREATE user_devices (per-device capability) + unique(user_id, device_key)
  0004_user_devices.down.sql     -- DROP user_devices
  0005_session_playout.up.sql    -- (AS-02) ADD sessions.playout0_ms (tier-selected initial playout)
  0006_metrics_prune_idx.up.sql  -- (#148) INDEX session_metrics (session_id, created_at) for the retention prune
  0007_user_delete_cascade.up.sql -- (#154) sessions.user_id FK → ON DELETE CASCADE (admin user deletion)
  0008_storage_foundation.up.sql -- (P5-01) CREATE user_homes; ADD apps.managed_home, apps.home_container_path
  0009_user_homes_orphans.up.sql -- (P5-05 erratum) user_homes.user_id/app_id NULLable + ON DELETE SET NULL
  0010_host_settings.up.sql      -- (#212) CREATE host_settings (per-host runtime config overrides)
  0011_session_profile.up.sql    -- (AS10-03) ADD sessions.profile_id (selected AS10-01 stream profile)
  0012_stream_health.up.sql      -- stream-perf support (per-host stream-health)
  0013_user_device_profile_history.up.sql -- (AS10-08) per-device profile history
  0014_app_host_delete_cascade.up.sql -- (admin-delete) sessions.app_id/host_id → CASCADE, user_homes.host_id → SET NULL; widens session_metrics.source to include 'native' (P9-01)
  0015_profile_preferences.up.sql -- (CP-01) stream_profile_policy singleton + updated_by
  0016_session_trace.up.sql      -- (ST-01) CREATE session_trace_events + session_trace_clock + indexes
  0017_policy_updated_by_set_null.up.sql -- (#154 follow-up) stream_profile_policy.updated_by → ON DELETE SET NULL
  0018_host_encoder_certification.up.sql -- (SPT-05) CREATE host_encoder_certification (upsert-latest) + lookup index
  0019_encoder_cert_vulkan.up.sql -- (Vulkan adoption) encoder-certification vulkan encoder support
  0020_invites_device_binding.up.sql -- (LP-SEC-01) CREATE instance_settings (singleton) + invites; user_devices +name/+trusted; auth_tokens.device_id + index; sessions.device_id
  -- (migrations 0021 through 0030 predate this prose ledger's last refresh; the committed SQL in
  --  migrations/ is authoritative. The multi-codec amendment adds the two lines below.)
  0031_multi_codec.up.sql        -- (multi-codec) ADD sessions.codec (CHECK h264|h265|av1, DEFAULT 'h264') + stream_profiles.codecs JSONB NULL (per-profile codec-pref list, catalog vocab) + hosts.codecs JSONB NULL (host wire codec set; NULL ⇒ ['h264'])
  0032_codec_scoped_history.up.sql -- (multi-codec §3.4, codec-aware decode-failure verdicts) ADD user_device_profile_history.codec (CHECK ''|h264|h265|av1, DEFAULT ''); widen its UNIQUE to (user_id, device_key, profile_id, codec). codec='' = profile-level verdict (all pre-0032 rows, plus presentation_degrading fails and pass outcomes, which are codec-independent); codec IN ('h264','h265','av1') = a decode-side fail (client_unsupported / decode_degrading) scoped to the codec the session streamed. Eligibility (ProfileFailures) treats codec IN ('','h264') as profile-blocking (h264 is the guaranteed floor, so an h264 decode failure is a profile failure, pre-0032 behaviour); the launch codec resolver (clamp 4) skips a failed non-h264 codec so the session degrades to the next candidate / the h264 floor instead of the profile vanishing from the picker. A pass deletes the fail row for the codec that actually ran. The admin stream.codec override bypasses clamp 4 (the forced re-test whose sustained smooth run records the clearing pass is the recovery path for a fixed encoder).
  -- (migration 0033 also predates this ledger's last refresh; the committed SQL in migrations/
  --  is authoritative. The UI-P1 amendment adds the line below.)
  0034_app_kind_favourites.up.sql -- (UI-P1) ADD apps.kind (CHECK game|desktop, DEFAULT 'game', presentation-only — no scheduler/agent reader); CREATE user_app_favourites (user_id, app_id, created_at) PK (user_id, app_id), both FKs ON DELETE CASCADE, + INDEX (app_id) for the apps-cascade delete. The down migration drops the table and the column.
  0035_runtime_presets.up.sql     -- (UI-P3) CREATE runtime_presets (name UNIQUE, description, image, args/env/mounts JSONB, managed_home, home_container_path, timestamps + set_updated_at trigger); ADD apps.runtime_preset_id UUID NULL REFERENCES runtime_presets(id) ON DELETE RESTRICT (NULL = the app carries everything itself = pre-UI-P3 behaviour, no backfill) + INDEX apps_runtime_preset_id_idx for the in-use check and the "Used by" list. The down migration drops the column FIRST (it is the dependent FK) then the table.
  0036_launch_profiles.up.sql     -- (UI-P4, EXPAND half of an expand/contract pair; NOT purely additive) (a) snapshot stream_profiles / stream_profile_policy / user_profile_preferences / (apps.id, default_profile_id, profile_policy) into _backup_0036_* tables FIRST — the fan-out is lossy and `down` is a verbatim restore, not a computed collapse; (b) ASSERT no app has profile_policy='custom' and FAIL naming them if any does (it cannot be converted behaviour-neutrally: a custom app lands on the legacy tier path where its settings are min(tier, app defaults)), then narrow the CHECK to ('inherit','prefer','force'); (c) ADD stream_profiles.codec TEXT NULL CHECK (h264|hevc|av1) — catalog vocabulary; (d) CREATE launch_profiles (id TEXT PK, display_name, description, visibility CHECK user|debug|internal, sort_order, updated_at + set_updated_at trigger); (e) CREATE launch_profile_rungs (launch_profile_id -> launch_profiles ON DELETE CASCADE, stream_profile_id -> stream_profiles ON DELETE RESTRICT, position CHECK > 0, PK (launch_profile_id, position), UNIQUE (launch_profile_id, stream_profile_id), INDEX (stream_profile_id)); (f) FAN OUT by RULE, never by special-casing an id: per existing stream_profiles row create a launch profile with the SAME id, resolve NULL/empty/unparseable `codecs` to the in-code default [h264 launchable, hevc future, av1 future], keep only `launchable` entries IN STORED ORDER (reordering h264 to last would flip an already-enabled AV1 host and is forbidden here), synthesise a lone h264 rung when zero are launchable (today that profile still streams h264 via the resolver floor), and materialise each surviving codec as a NEW stream_profiles row id = '<parent>-<codec>' with visibility='internal' and EVERY other column copied verbatim; RAISE NOTICE per launch profile whose h264 rung is not last (rungs after it are unreachable); (g) ADD sessions.stream_profile_id TEXT NULL REFERENCES stream_profiles(id) — the resolved rung, vs profile_id which keeps the user's pick; (h) REPOINT THREE FKs to launch_profiles(id): stream_profile_policy.global_default_profile_id, apps.default_profile_id, AND user_profile_preferences.default_profile_id (the third has 0 rows, so omitting it passes every test and 500s on the first real user preference). Deliberately NOT done here: dropping stream_profiles.codecs, deleting the legacy stream_profiles rows, making codec NOT NULL, or giving the global default a value — the first three are the later CONTRACT migration, the fourth would change every `inherit` app's effective resolution at once. The down migration drops the three FKs, drops launch_profile_rungs + launch_profiles, drops sessions.stream_profile_id, restores stream_profiles / policy / preferences / the two apps columns from the _backup_0036_* snapshots, drops stream_profiles.codec, widens the profile_policy CHECK back, recreates the three FKs against stream_profiles(id), and drops the backups — emitting a NOTICE naming any launch profile created after `up` (it is dropped, not collapsed, because the fan-out cannot be inverted). Admin writes made after `up` are lost by `down`; both facts are stated verbatim in the migration file header.
  0037_app_launch_profiles.up.sql -- (UI-P5, purely additive) CREATE app_launch_profiles (app_id -> apps ON DELETE CASCADE, launch_profile_id -> launch_profiles ON DELETE CASCADE, PK (app_id, launch_profile_id), INDEX (launch_profile_id)) — the per-app allow-list of launch profiles a user may pick from the menu beside Play. A TABLE, NOT A COLUMN: a jsonb/text[] column carries no referential integrity, so an entry could name a launch profile that no longer exists and nothing would notice. EMPTY SET = today's behaviour (any launch profile the device is eligible for), which is every existing app — no backfill, no behaviour change on upgrade; non-empty INTERSECTS with eligibility and can only narrow. The app's own default (apps.default_profile_id under profile_policy='prefer') is IMPLICITLY always included and deliberately NOT stored here (a copy would need syncing on every default change), and the 'force' exclusion is a WRITE-PATH rule rather than a CHECK because it spans two tables. CASCADE on the launch_profile side is the deliberate choice over RESTRICT: an allow-list entry is a restriction naming a catalogue object, not a reference that would be left pointing at nothing, so it must not make retiring a launch profile harder the more carefully an operator curated their apps — the cost, that a cascade emptying a list turns it back into 'unrestricted', is bounded (the widened set is still only what the device is eligible for) and is written into the admin activity log. Touches NOTHING of 0036's expand/contract state: stream_profiles.codecs and the legacy rows stay. The down migration drops the table; any allow-list configured while 0037 was applied is discarded, because the pre-0037 schema has no representation for it.
  0039_app_artwork.up.sql -- (UI-P7, purely additive) ALTER apps ADD hero_url TEXT (the wide HERO crop, distinct from cover_url's TILE crop — 16:10 as shipped, 2:3 portrait since #385, which changed no column and needed no migration — two different source assets, not one image scaled) + CREATE app_artwork (app_id PK -> apps ON DELETE CASCADE, source CHECK ('provider','manual','none'), provider/provider_ref/matched_name, tile_asset/hero_asset content-addressed blob names, attribution, locked, updated_at) with partial indexes on the two asset columns. NOTHING existing changes: both new URL columns are NULL for every existing row, which is the pre-UI-P7 gradient-tile rendering, and no app_artwork row exists until the feature is switched on. source='none' is a NEGATIVE CACHE, not an error state — a desktop app is not in a games database, so the row records that fact and stops every later sweep re-asking a third party. locked=true marks an admin override the automatic fetcher must never overwrite. Touches NOTHING of 0036's expand/contract state: stream_profiles.codecs and the legacy rows stay, awaiting their own later contract migration. The down migration drops the table and the column; cached image files on disk survive (they are content-addressed, so a re-apply re-fetches rather than reusing them), and the provenance — including which matches an admin had corrected — is lost, which is acceptable for a rollback path whose only casualty is a cache and its bookkeeping.
  0040_instance_secrets.up.sql -- (encrypted-secrets facility, purely additive) CREATE instance_secrets (name PK, ciphertext BYTEA, nonce BYTEA, key_version INT CHECK >= 1, hint TEXT, updated_by -> users ON DELETE SET NULL, updated_at) + the shared set_updated_at trigger. NOTHING existing changes and no row exists until an admin stores a secret, so an upgraded deployment behaves byte-for-byte as before (the artwork key keeps coming from QUASAR_STEAMGRIDDB_API_KEY until someone sets one in the UI). Values are AES-256-GCM with a per-encryption random nonce and the row's NAME bound in as additional authenticated data, so a ciphertext moved to another name fails to decrypt instead of silently becoming that other secret. The MASTER KEY IS NOT IN THE DATABASE — it is QUASAR_SECRET_KEY in the environment — which is the entire point: a database dump cannot read these values. key_version is recorded per row so rotation is possible later with no schema change. The down migration drops the table and therefore every stored secret; that is unavoidable and honest, since the values are only readable with a key that lives outside the database. An operator rolling back keeps their env-var fallbacks and must re-enter anything set only through the UI.
  -- (migration 0041 also predates this ledger's last refresh; the committed SQL in migrations/ is
  --  authoritative, and its prose lives at §host_encoder_certification. Phase 1 adds the line below.)
  0042_app_external_ref.up.sql -- (Steam library discovery Phase 1, purely additive) ALTER apps ADD external_source TEXT NOT NULL DEFAULT '' CHECK (external_source IN ('', 'steam')) + external_id TEXT NOT NULL DEFAULT '' CHECK (external_id = '' OR external_id ~ '^[1-9][0-9]{0,9}$'); CREATE INDEX apps_external_ref_idx ON apps (external_source, external_id) WHERE external_id <> ''. Together the two columns say "this app IS provider X's title Y", today only ('steam', <appid>). NOTHING existing changes: both default to '' = "not a provider title", the state of every existing row, so no backfill and no behaviour change on upgrade. The apps_external_id_ck regex is ARGUMENT-INJECTION CONTAINMENT, not tidiness — the appid is eventually rendered into STEAM_STARTUP_FLAGS, which the quasar-steam entrypoint word-splits with `read -r -a`, so the constraint is what stops a stored '480 -foo' reaching the Steam client as two extra arguments; it is one of four validation points and the only one that survives an admin editing the column by hand later. The index is PARTIAL because every app in a pre-discovery catalogue has '' and a full index would be almost entirely one dead key. SCOPE: spec §4.1 lands five columns; parent_app_id, origin, library_provider, the derived-tile shape CHECK and the (parent_app_id, external_source, external_id) unique index are all PHASE 3 and deliberately not here — Phase 1 ships only what it reads, which is artwork resolution by id instead of by fuzzy title. The down migration drops the index, both CHECKs and both columns in symmetric order; which apps were tagged is lost, so a re-apply leaves every app back on the fuzzy title path until an admin re-tags them — a tagging, not a cache (the app_artwork rows are keyed on app_id and survive).
  0043_entitlements.up.sql -- (Steam library discovery Phase 2 — the TABLE is additive, WHAT READS IT IS NOT: GET /v1/apps becomes entitlement-filtered for every role and the launch/swap paths gain a terminal 403; see control-api.md. Opus + human sign-off.) CREATE entitlements (id UUID PK, subject_type TEXT CHECK IN ('user','all'), subject_id UUID NULL -> users ON DELETE CASCADE, app_id UUID -> apps ON DELETE CASCADE, granted_by TEXT CHECK IN ('admin','provider','migration'), granted_by_user UUID NULL -> users ON DELETE SET NULL, source_ref TEXT DEFAULT '', created_at) + CONSTRAINT entitlements_subject_shape_ck CHECK ((subject_type = 'all') = (subject_id IS NULL)) + UNIQUE INDEX entitlements_all_uk (app_id) WHERE subject_type='all' + UNIQUE INDEX entitlements_user_uk (subject_id, app_id) WHERE subject_type='user' + INDEX entitlements_app_idx (app_id) + THE BACKFILL. TWO PARTIAL UNIQUE INDEXES, NOT ONE PLAIN UNIQUE: Postgres does not treat NULLs as equal in a UNIQUE, so a single UNIQUE (subject_type, subject_id, app_id) would consider every ('all', NULL, <app>) row distinct and silently permit unlimited duplicate 'all' rows — quietly, because the read predicate is EXISTS, so the list would look right while a revoke that deletes "the" row left the app visible and a grant's ON CONFLICT had nothing to conflict on. THE BACKFILL (INSERT SELECT 'all', NULL, id, 'migration' FROM apps) IS IN THIS TRANSACTION AND IS THE MOST LOAD-BEARING STATEMENT IN THE MIGRATION: filtering goes live in the same deploy, and against an empty table it returns nothing — every user's library goes blank on every deployment simultaneously, and it would SHIP (migration applies, service boots, go-test-db passes because the tests create their own entitlements, web build passes). There is no automated gate between "empty table" and "every library is empty" other than this INSERT; with it, Phase 2's day-one behaviour change is EXACTLY ZERO. Disabled apps ARE backfilled — the filter is ANDed with enabled=true so they are invisible either way today, but skipping them would mean re-enabling an app silently failed to bring it back months later with nothing to point at. granted_by='migration' (not 'admin') keeps the backfill distinguishable forever. 'provider' is in the CHECK but NOTHING WRITES IT YET — that is Phase 4; it is here so Phase 4 needs no ALTER and so revoking one is already a working path. NUMBERING: the spec says 0042 throughout, written when Phases 1 and 2 shared a migration; Phase 1 shipped first and took it, and §13's "0042 continued, or 0043 if Phase 2 shipped separately" is the clause that applies. The down migration drops the table, and therefore the backfill AND every grant made since — a re-apply re-runs the backfill, so the catalogue returns FULLY OPEN rather than at whatever narrower state was configured: a widening, safe for availability and UNSAFE for confidentiality, so dump entitlements before rolling back any deployment where access has actually been restricted.
  0044_derived_tiles.up.sql -- (Steam library discovery Phase 3 — ADDITIVE, but Opus + human sign-off on TWO counts: it widens a FROZEN ENUM and it adds an app shape enforced by a DB CHECK. See control-api.md §Derived tiles.) ALTER apps DROP CONSTRAINT apps_kind_check + ADD CONSTRAINT apps_kind_check CHECK (kind IN ('game','desktop','launcher')); ALTER apps ADD parent_app_id UUID NULL -> apps(id) ON DELETE CASCADE + origin TEXT NOT NULL DEFAULT 'manual' + library_provider TEXT NOT NULL DEFAULT ''; + CONSTRAINT apps_origin_ck CHECK (origin IN ('manual','discovered')) + apps_library_provider_ck CHECK (library_provider IN ('', 'steam')) + apps_derived_shape_ck CHECK (parent_app_id IS NULL OR (external_source <> '' AND external_id <> '' AND managed_home = false AND runtime_preset_id IS NULL AND runtime_spec = '{}'::jsonb AND library_provider = '')); + UNIQUE INDEX apps_parent_external_uk ON apps (parent_app_id, external_source, external_id) WHERE parent_app_id IS NOT NULL + INDEX apps_parent_app_id_idx ON apps (parent_app_id) WHERE parent_app_id IS NOT NULL (Postgres does not auto-index the referencing side of an FK, and the unique index above cannot serve the plain "which tiles belong to this parent" lookup). NOTHING existing changes: every default (NULL, 'manual', '') makes every pre-0044 row valid with no backfill, and widening a CHECK is data-safe in itself since no row can violate a strictly larger allow-list — so the recreate is deliberately NOT `NOT VALID`. THE TRAP IS THE CONSTRAINT NAME, NOT THE DATA: 0034 declared apps.kind's CHECK INLINE and UNNAMED, so it carries a SERVER-GENERATED name — conventionally apps_kind_check, but that is a convention and not a guarantee. VERIFY THE REAL NAME AGAINST THE LIVE DATABASE (\d apps, or pg_constraint filtered on conrelid) BEFORE WRITING THIS MIGRATION: a DROP CONSTRAINT naming a constraint that does not exist fails the migration ON BOOT, which is the control-plane crash-loop class CLAUDE.md warns about. apps_derived_shape_ck IS THE POINT OF THE PHASE, NOT TIDINESS: runtime_spec = '{}' is what makes "the tile contributes no runtime" STRUCTURAL rather than conventional, so the parent's spec merges in AT LAUNCH and an edit to the parent (an image bump, a new GPU flag, a new mount) reaches every derived tile with no re-sync and no stale copies — the same call UI-P3 made for runtime presets. It is a DB CHECK and not a handler rule because a validated Tower experiment hardcoded a host path into a tile's runtime_spec.mounts, and because runtime_spec has NO schema validation anywhere on the control-plane write path (raw JSONB in, opaque on the wire), so there is no existing validation layer for it to live in and only a constraint survives an admin editing the row later. managed_home = false is load-bearing in the other direction too: the single-writer guard is GATED on managed_home, so it must read the PARENT's value or it does not fire for derived tiles at all — the exact inverse of its purpose. apps_parent_external_uk is PARTIAL and is what bounds the catalogue by the UNION of all users' installed titles rather than by users x titles. kind STAYS PRESENTATION-ONLY and that promise is now load-bearing: discovery is triggered by library_provider and NEVER by kind, or an operator flipping a presentation dropdown silently stops a background job; the single reader anywhere is artwork resolution, which short-circuits 'desktop' AND 'launcher'. This completes the five-column plan 0042 deliberately split. FORWARD COMPAT: a client pinned before this amendment holds AppKind as a closed two-value union and will receive kind:"launcher" — benign (TS unions are erased, the filter predicate never matches, the tile shows under All and no segment) but a DEGRADATION, so the quasar-client pin bump is scheduled WITH this phase. The down migration drops the index, the three CHECKs and the three columns in symmetric order and NARROWS kind back to ('game','desktop') — WHICH FAILS IF ANY ROW IS 'launcher', deliberately: a silent UPDATE apps SET kind='game' would rewrite operator classification during a rollback. Re-classify the launcher apps first. Dropping parent_app_id also destroys the parentage of every derived tile, leaving orphans with runtime_spec '{}' that CANNOT LAUNCH; delete or re-parent them before a deliberate down migration.
```
The golang-migrate CLI can target this path directly:
`migrate -path control-plane/migrations -database "$DATABASE_URL" up`
File naming is golang-migrate's `{version}_{description}.{up|down}.sql`. Future changes are new
numbered pairs; **0001 is frozen** like the rest of this contract (so `0002` is a new pair, not
an edit to `0001`). The default applies to existing rows, so no backfill is needed. The
committed SQL in `migrations/` is the authoritative DDL — this document is its prose companion;
if they ever disagree, that is a bug to fix under sign-off, not a silent divergence.
