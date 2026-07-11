# P1-A — Control-plane ↔ node-agent API (FROZEN)

The spine of architecture invariant #1 (*control plane / node agent split*). The control
plane (Go) owns accounts, scheduling and state; the node agent (Rust) owns a host's GPUs and
runs sessions. This document is the **only** interface between them. Get it right and
multi-host (Phase 3) and the K8s DaemonSet model (Phase 4) are added hosts, not a rewrite.

**Frozen** (see `CLAUDE.md`): P1-4 (agent skeleton), P1-6 (lifecycle), P1-8 (resource model)
implement against this freely; changes need Opus + human sign-off. Co-defined with
`schema.md` (the `hosts`/`gpus`/`sessions` tables and the **session state machine** are the
source of truth) and with `signaling.md` (this channel relays signaling — see §Signaling relay).

> **Amendment — P2-02 (launcher↔game swap), additive, requires sign-off.** Adds one
> control→node message, `session_swap_app` (§Messages — control → node), with its own ack +
> `session_state` callback semantics. **Additive** — a new `ControlMsg` variant; no existing
> message, field, or the assign/start/stop ack contract changes. The swap keeps the encode +
> `webrtcbin` tail live (no signaling renegotiation — `signaling.md` is unchanged). See
> `docs/phase2/P2-02-contract-app-swap.md`. **Stops at the contract — P2-07 implements the swap.**

> **Amendment — host-runtime-settings-admin: additive, requires sign-off.** Adds two new
> downstream (control → node) messages: `config_update` (push the resolved knob block after
> registration and after every admin change) and `restart` (ask the agent to exit cleanly so
> its container restart policy restarts it with the new config). **Additive** — new `ControlMsg`
> variants; no existing message, field, ack contract, or upstream message changes. An older agent
> that does not understand the new `type` values silently ignores them (existing discipline).
> See `control-api.md` §"Host Runtime Settings" for the admin surface that drives these messages.

> **Amendment — P3-01 (host-lifecycle + multi-host scheduling): no wire change.** Host drain/cordon
> is a **control-plane** concern (`control-api.md` `POST /v1/hosts/{id}/drain`; the host-status state
> machine lives in `schema.md`). A **force**-drain reuses the existing `session_stop`
> `reason:"host_draining"` (already enumerated under §`session_stop`) — no new or changed message.
> The offline reaper (§Reconnection & reconciliation) is unchanged; it now additionally stamps the
> session `state_detail = 'host_lost'`, a control-plane-side `sessions` write that does not touch
> this wire. **This contract is therefore unchanged by P3-01; the note is recorded for
> traceability.** See `docs/completed/phase3/P3-01-contract-host-lifecycle.md`.

> **Amendment — AS10-04 (profile ABR floor), additive, requires sign-off.** Adds one optional
> field to the `session_assign` `stream` block: **`abr_floor_kbps`: int** (kbit/s, omitted ⇒
> `0`). When a session was launched from a stream profile (AS10-01) the control plane resolves
> the profile's ABR floor from its in-code catalog and sends it so the agent's in-session ABR
> governor honours the profile's stated minimum. **Additive** — a new optional field; no existing
> field, message, or ack contract changes. Omitted/`0` (legacy/tier/override launch, or an older
> control plane) ⇒ the agent keeps its env/ratio-derived floor. See §`session_assign` and
> `docs/phase-as10/`.

> **Amendment — ST-01 (Observability v2 — session trace), additive, requires sign-off.** Adds one
> upstream (node → control) message, `session_trace_event` (§Messages — node → control), by which
> the agent emits host-side discrete trace markers (ABR retarget, source swap, encoder drop burst,
> host webrtcbin ICE/connection transitions) event-driven, not on a clock. **Additive** — a new
> upstream `type`; no existing message, field, or ack contract changes. Like `session_metrics` it
> is **fire-and-forget** (no `ack`), carries **no** session-state authority (`session_state` stays
> the only progress signal), and is host-ownership validated (`GetSessionHostState`) exactly like
> `session_metrics` — a cross-host or not-running-here event is dropped, not stored, never fatal to
> the WS. An older control plane simply ignores the unknown `type`; an older agent never sends it.
> `signaling.md`, `input.md`, `native-client.md` are unchanged. See `docs/session-trace/contract-amendment.md` §C
> and `docs/session-trace/trace-format.md` §3.2 (the agent event allow-list).

## Transport: one persistent, node-initiated WebSocket
The node agent **dials** the control plane and holds open a single WebSocket; all agent-API
traffic flows over it, in both directions. JSON, one message object per WS frame, discriminated
by `type` — identical encoding discipline to `signaling.md`/`input.md` (debuggable, cheap-tier
friendly).

Why node-initiated and persistent:
- **NAT / K8s friendly.** The control plane never has to reach *into* a node. Nodes behind NAT,
  or as a DaemonSet pod dialing a control-plane Service, work with zero extra plumbing. This is
  exactly why this split makes the Kubernetes story fall out cleanly.
- **Single liveness signal.** Connection up + heartbeats = host `online`; disconnect = `offline`
  and its sessions are reaped. No separate health-check fabric.
- **Bidirectional.** Upstream events (capacity, heartbeat, session state, signaling) and
  downstream commands (assign/start/stop, signaling) share one ordered channel.

```
node-agent                         control-plane
   | --- WS connect (TLS) --------->  |
   | --- {type:"register"} -------->  |   (auth: enrollment or node secret)
   | <-- {type:"registered"} ------   |   (host_id, heartbeat_interval_ms)
   | --- {type:"capacity"} -------->  |   (GPUs, host cpu/mem)  -> upserts hosts+gpus
   | --- {type:"heartbeat"} ------->  |   (periodic; liveness + live utilization)
   |                                  |
   | <-- {type:"session_assign"} --   |   (reserve done; prepare)   ─┐ command
   | --- {type:"ack", id} --------->  |                              │ + ack
   | <-- {type:"session_start"} ---   |                              │
   | --- {type:"session_state"} -->   |   (starting -> running ...)  ┘ callbacks
   | <===== signaling relay (both directions, see below) =====>      |
   | <-- {type:"session_stop"} ----   |
   | --- {type:"session_state"} -->   |   (stopping -> stopped; reservation released)
```

### Reconnect & message correlation
- If the WS drops, the agent re-dials with backoff and re-`register`s, then re-sends a full
  `capacity` report. The control plane maps the reconnect to the existing `hosts` row by
  `node_name` (see Auth). Sessions that were `running` are reconciled (§Reconnection).
- Commands that need a reply (`session_assign`, `session_start`, `session_stop`) carry an
  `id` (string, agent-unique per connection is sufficient as the control plane generates it);
  the agent replies with `{type:"ack", id, ok, error?}`. `session_state` callbacks are
  unsolicited (no `id` required) and carry their own `session_id`.

## Auth / enrollment
A node must prove it's allowed to join before it can register.
- **Enrollment (first contact):** the agent presents a pre-shared `enrollment_token`
  (control-plane config, delivered out of band) plus its stable `node_name`. The control
  plane creates the `hosts` row, mints a per-node `node_secret`, returns it in `registered`,
  and stores `node_secret_hash`. The agent persists the secret locally.
- **Reconnect:** the agent presents `node_name` + `node_secret`; the control plane checks it
  against `node_secret_hash`. No new row is created.
- End state (Phase 3/4) replaces the shared enrollment token with mTLS / SPIFFE identities;
  the message shape (`register` carrying a credential) does not change.

---

## Messages — node → control (upstream)

### `register` — first message after every (re)connect
```json
{
  "type": "register",
  "node_name": "gpu-host-01",
  "agent_version": "0.1.0",
  "auth": { "enrollment_token": "<pre-shared>" }
}
```
On reconnect, `auth` is `{ "node_secret": "<issued-earlier>" }` and `node_name` must match.
The control plane replies `registered` (below) or `{type:"error", ...}` and closes on auth failure.

### `capacity` — full capacity report
Sent immediately after `registered`, and again whenever hardware/topology changes. Replaces
the previous report wholesale (idempotent upsert of `hosts` + `gpus`).
```json
{
  "type": "capacity",
  "host": {
    "cpu_cores": 16, "mem_mb": 64000,
    "cpu_model": "AMD Ryzen 9 7950X 16-Core Processor",
    "storage": [
      { "label": "agent-data", "path": "/var/lib/quasar-agent",
        "total_mb": 819200, "available_mb": 512000 }
    ]
  },
  "effective_settings": { "encoder": "nvenc", "render_node": "/dev/dri/renderD128" },
  "gpus": [
    { "index": 0, "vendor": "amd", "model": "Radeon Pro V520",
      "vram_mb_total": 16384, "encode_slots_total": 2,
      "render_node": "/dev/dri/by-path/pci-0000:04:00.0-render" }
  ],
  "console_capabilities": {
    "connectors": ["DP-4", "HDMI-A-1"],
    "audio_sinks": [
      { "id": "hw:1,3", "label": "GPU HDA (DP-4)" },
      { "id": "hw:0,0", "label": "Motherboard" }
    ],
    "input_devices": [
      { "path": "/dev/input/event4", "label": "Keyboard" },
      { "path": "/dev/input/event5", "label": "Mouse" }
    ]
  }
}
```
`console_capabilities` *(NEW, CM-01, optional, additive)* enumerates what the host can do in
console mode — DRM display connectors, host audio sinks (ALSA/pipewire), and physical input
devices — so the admin console-config UI can populate its selectors instead of guessing device
paths. It rides `capacity` because these are **hardware/topology** facts: the agent re-sends
`capacity` when topology changes (e.g. a display hotplug), keeping the UI's lists fresh. Absent
⇒ the control plane reports empty capability arrays and the UI offers only `auto`. The control
plane stores the latest report and returns it in `GET /v1/admin/hosts/{id}/console-config`.

`host.storage` *(NEW, host-observability, optional, additive)* reports the filesystem
capacity/availability (statvfs) of the storage roots the agent can see — at minimum its data
root (`/var/lib/quasar-agent`, which on the reference compose deployments is a named volume on
the Docker data filesystem, so it reflects the space available to images/containers/homes), plus
the managed-homes root when the `local` storage provider is configured. Each entry:
`label` (stable short id, e.g. `"agent-data"`, `"homes"`), `path` (agent-side mount point),
`total_mb`, `available_mb`. Because availability drifts, the agent re-sends `capacity` when a
report would change materially (checked on a debounced ~60 s cadence, re-sent on ≥1 GB or ≥1%
delta). Absent ⇒ the control plane keeps its last stored value (or null if never reported).

`effective_settings` *(NEW, host-observability, optional, additive)* is the agent's **resolved
runtime settings** — the `env ← overrides` overlay it is actually running with (values
stringified), reported **as configured** (verbatim overlay values). Path canonicalisation
(e.g. a `/dev/dri/by-path/...` render node resolving to its `renderD*` target) is an
agent-internal detail and is *not* reflected here — this keeps `effective` directly
comparable with the admin API's `resolved`/`overrides` views (same value space), which is the
field's purpose (host-observability-2 clarification). The agent re-sends `capacity` after applying each
`config_update`, so the control plane always holds the current effective view. This closes the
gap noted under `config_update` (the admin `resolved` field is a *display* view that cannot see
the agent's env): the control plane stores the latest map and returns it as `effective` in
`GET /v1/admin/hosts/{id}/settings` (`control-api.md`). Restart-class knobs are reported as
**latched** (the values the process is actually using), so a pending-restart discrepancy is
visible by comparing `effective` against `resolved`. Absent ⇒ `effective` is null.

`host.cpu_model` *(NEW, host-observability-2, optional, additive)* — the CPU marketing name
(`/proc/cpuinfo` `model name`). `gpus[].render_node` *(NEW, host-observability-2, optional,
additive)* — the **stable** render-node device path for this GPU, in the reboot-safe by-path
form constructed from the device's PCI address
(`/dev/dri/by-path/pci-<addr>-render`). This string is valid as a `render_node` setting value
(the agent canonicalises by-path values, with a sysfs PCI-address fallback for containers that
lack `/dev/dri/by-path`), so the admin UI can offer the reported values as a picker instead of
free-text device paths. Absent ⇒ null on the API.

`encode_slots_total` is the concurrent encode-session cap (the NVENC/VCN limit — architecture
§"Resource governance"). At N=1 this is one GPU with generous slots; the field is mandatory so
the scheduler's reservation path is exercised from day one. The control plane upserts `gpus` by
`(host_id, index)`; GPUs absent from the report are removed (cascade-safe — a GPU with active
sessions should not vanish, that's a fault the control plane logs).

> **P2-01 review verdict — no agent-api change required.** The Phase-2 resource governor
> (`control-api.md` §Admission control) needs only per-GPU `vram_mb_total`, `encode_slots_total`,
> `vendor`, `model` to compute availability — all already carried here. The per-user session
> quota lives entirely in the control plane (`schema.md` `users.max_concurrent_sessions`) and
> never reaches the agent. This contract is therefore **unchanged** by P2-01; the note is
> recorded only for traceability.

### `heartbeat` — liveness + live utilization
```json
{ "type": "heartbeat", "running_sessions": ["<session_id>"], "ts_unix_ms": 1735689600000 }
```
Sent every `heartbeat_interval_ms`. Updates `hosts.last_heartbeat_at`. `running_sessions` lets
the control plane reconcile its view against the agent's ground truth (detect orphans both ways).
Missing N consecutive heartbeats ⇒ host `offline`, its sessions `failed`, reservations released.

### `session_state` — lifecycle callback (the authoritative progress signal)
```json
{
  "type": "session_state",
  "session_id": "<uuid>",
  "state": "running",
  "detail": "pipeline live; webrtcbin offer ready",
  "error": null
}
```
`state` is one of the **`schema.md` session states** the agent can report:
`starting`, `running`, `stopping`, `stopped`, `failed`. (`pending`/`assigned` are control-plane
states; the agent enters the picture at `starting`.) The control plane writes the `sessions`
row, stamps `started_at`/`ended_at`, and **releases the GPU reservation** when the agent reports
`stopped`/`failed`. `error` is set (string) only with `failed`.

### `ack` — reply to a downstream command
```json
{ "type": "ack", "id": "<command-id>", "ok": true, "error": null }
```
`ok:false` with `error` means the agent rejected/failed the command (e.g. cannot satisfy the
launch spec). The control plane then transitions the session to `failed` and releases the
reservation. An `ack` only confirms *receipt/acceptance*; actual progress comes via
`session_state`.

### `session_metrics` — per-session telemetry (P4-01)
> *Additive, observability amendment. A new upstream message; it changes no existing
> message shape, and the control plane that predates it simply ignores an unknown `type`.
> It is **fire-and-forget like `heartbeat`** (no `ack`), and carries **no** session-state
> authority — `session_state` remains the only progress signal. Telemetry is observability,
> never access control.*

Emitted by the agent **once per `running` session, on the `heartbeat_interval_ms` cadence**,
carrying only **host-observable** numbers from the GStreamer pipeline. (The browser's own
`getStats()` telemetry does **not** travel through the agent — the client posts it straight
to the control plane, `control-api.md` `POST /v1/sessions/{id}/stats`. The two sources are
kept distinct and reconciled in `session_metrics.source`, `schema.md`.)
```json
{
  "type": "session_metrics",
  "session_id": "<uuid>",
  "ts_unix_ms": 1735689600000,
  "window_ms": 5000,
  "fps": 59.8,
  "bitrate_kbps": 14820,
  "encode_ms": 4.6,
  "frames_encoded": 299,
  "frames_dropped": 1
}
```
- **Lightweight fields (always present):** realized `fps`, actual `bitrate_kbps`, mean
  `encode_ms` over the window (from the encode pad-probe FIFO — *timing only*, no overlay
  write), `frames_encoded`, `frames_dropped`. `window_ms` is the aggregation period — it
  **equals the emit cadence** (`heartbeat_interval_ms`) so windows are contiguous and
  non-overlapping. These are the always-on signal the design promised — collectible with **no
  overlay stamping**, so the default hot path is unchanged (the Phase-0 invariant; see `P4-03`).
  The exact `metrics` JSONB keys are enumerated in `schema.md`'s field dictionary — implementers
  use those names verbatim. Glass-to-glass latency is measured **always-on, browser-side** via the
  abs-capture-time RTP header extension (`control-api.md`), not by a host-side overlay.
- The control plane writes a `session_metrics` row with `source='agent'` (`schema.md`).
  A malformed or unparsable message is dropped without dropping the connection. A sample whose
  `session_id` is not currently `running` on this host (e.g. it arrived as the session went
  terminal) is **dropped, not stored** — metrics never resurrect or alter a session.

### `session_trace_event` — host-side discrete trace marker (ST-01)
> *Additive, observability amendment. A new upstream message; it changes no existing message
> shape, and a control plane that predates it simply ignores the unknown `type`. Unlike
> `session_metrics` (heartbeat cadence) this is **event-driven** — sent when the event happens,
> not on a clock. It is **fire-and-forget** (no `ack`), carries **no** session-state authority
> (`session_state` remains the only progress signal), and is host-ownership validated exactly like
> `session_metrics`. Telemetry is observability, never access control.*

Emitted by the agent when a host-side trace event occurs during a `running` session — an in-session
ABR retarget, a launcher↔game source swap completing, an encoder frame-drop burst, or a host
`webrtcbin` ICE/connection transition. The discrete counterpart to the periodic `session_metrics`
samples; the control plane joins the two on one timeline for the diagnostic bundle (`control-api.md`).
```json
{
  "type": "session_trace_event",
  "session_id": "<uuid>",
  "ts_unix_ms": 1735689600000,
  "event": "abr.retarget",
  "payload": { "from_kbps": 14000, "to_kbps": 11000, "reason": "gcc_downshift" }
}
```
- **`type`** is the WS message discriminator (`"session_trace_event"`). **`event`** is the trace
  event type from the agent allow-list (`abr.retarget`, `pipeline.source_swapped`,
  `encoder.drop_detected`, `webrtc.state_changed` — `trace-format.md` §3.2). **`payload`** is the
  per-type object (`trace-format.md` §3). `ts_unix_ms` is the agent wall-clock at the event (same
  convention as `session_metrics.ts_unix_ms`).
- **`webrtc.state_changed` — Connected/Completed both mean established.** ICE may jump
  `Checking → Completed`, skipping `Connected`; a reader treats both as transport-established.
- The control plane writes a `session_trace_events` row with `source='agent'` (`schema.md`) after
  validating host ownership (`GetSessionHostState`). A malformed or unparsable message is dropped
  without dropping the connection. An event whose `session_id` is not currently `running` on this
  host is **dropped, not stored** — events never resurrect or alter a session (identical posture to
  `session_metrics`).

---

## Messages — control → node (downstream)

### `registered` — reply to `register`
```json
{
  "type": "registered",
  "host_id": "<uuid>",
  "node_secret": "<issued-on-enrollment-only>",
  "heartbeat_interval_ms": 5000
}
```
`node_secret` is present only on first enrollment; omitted on subsequent reconnects.

### `session_assign` — reserve + prepare (control plane has already reserved in Postgres)
The scheduler has picked this host+GPU and written the reservation to the `sessions`/`gpus`
rows (`state=assigned`) **before** sending this. Assign tells the agent to prepare the session
(pull image, allocate, be ready to start). Kept distinct from `start` so the scheduler's
reserve→prepare→go-live steps are real (P1-6), even though at N=1 `start` may follow immediately.
```json
{
  "type": "session_assign",
  "id": "<command-id>",
  "session_id": "<uuid>",
  "gpu_index": 0,
  "app": {
    "image": "ghcr.io/quasar/app-foo:latest",
    "args": [], "env": {}, "mounts": [], "gpu": true
  },
  "stream": {
    "width": 1920, "height": 1080, "fps": 60,
    "bitrate_kbps": 15000, "h264_profile": "constrained-baseline"
  },
  "resources": { "vram_mb": 1024, "encode_slots": 1 },
  "video_topology": "stream_only"
}
```
`app` mirrors `apps.runtime_spec`; `stream` mirrors the `sessions` launch-param columns;
`resources` mirrors the reserved amounts. The agent must not exceed `resources` (it's the
budget the control plane reserved). `gpu_index` maps to the agent's local GPU enumeration
(same index it reported in `capacity`).

`video_topology` is an additive per-session output plan:

- `stream_only` (default when omitted): encode plus WebRTC; no local display output.
- `local_only`: local display/audio/input only; the control plane reserves zero encode slots
  and creates no signaling token, and the agent must not construct an encoder, RTP, WebRTC, or
  streamed-audio pipeline.
- `dual_output`: local display plus the normal WebRTC stream from the same source.

The field is assignment-scoped. `console_config.enabled` advertises/configures host console
capability but does not by itself mirror ordinary browser sessions to a physical display.

> *(AS10-04, additive)* The `stream` block may carry an optional **`abr_floor_kbps`: int**
> (kbit/s, omitted ⇒ `0`). When the session was launched from a stream profile (AS10-01) the
> control plane resolves the profile's ABR floor from its catalog and sends it here, so the
> agent's in-session ABR governor never starves below the profile's stated minimum. The ABR
> *ceiling* stays the profile's nominal bitrate = the existing `bitrate_kbps` (no separate
> ceiling field). **Omitted or `0`** (a legacy/tier/override launch, or an older control plane)
> ⇒ the agent falls back to its env/ratio-derived floor (`QUASAR_ABR_FLOOR_KBPS`, else `ceiling
> × QUASAR_ABR_FLOOR_RATIO`) exactly as before. Additive — no existing field or shape changes.

### `session_start` — bring the pipeline up
```json
{ "type": "session_start", "id": "<command-id>", "session_id": "<uuid>" }
```
The agent realizes the assigned `video_topology`, then reports
`session_state: starting` → `running`. Once `running`, the agent's `webrtcbin` is the offerer and
the signaling relay (below) can carry the offer to the client for `stream_only`/`dual_output`.
For `local_only`, `running` means the local display pipeline reached PLAYING; no offer is emitted.

### `session_stop` — teardown + release
```json
{ "type": "session_stop", "id": "<command-id>", "session_id": "<uuid>", "reason": "user_requested" }
```
`reason` ∈ `user_requested | idle_timeout | host_draining | admin | error`. The agent tears down
and reports `session_state: stopping` → `stopped`. The reservation is released when `stopped`
arrives.

### `session_swap_app` — swap the source app, transport stays live (P2-02)
Tells the agent to swap a `running` session's **source container** behind its interpipe boundary
while the encode + `webrtcbin` tail (and thus the WebRTC transport) stay up. `app` is the same
`AppSpec` shape as `session_assign.app`.
```json
{
  "type": "session_swap_app",
  "id": "<command-id>",
  "session_id": "<uuid>",
  "app": { "image": "ghcr.io/quasar/game-bar:latest", "args": [], "env": {}, "mounts": [], "gpu": true }
}
```
The control plane has already validated ownership, swappable state, and that the new app **fits
within the held reservation** (`control-api.md`) before sending this — the agent does not re-check
the budget; it must keep the swapped container within the same `resources` the session was
assigned.

**Ack semantics (swap-specific — differs from assign/start).** The agent replies
`ack{ id, ok, error? }`:
- `ack{ok:true}` — the agent accepted the swap and will attempt it; progress comes via
  `session_state`.
- **`ack{ok:false, error}` — the agent rejected the command (e.g. it does not hold this session,
  or the spec is unusable). The session is left running the *previous* app and stays `running` —
  the swap is a no-op.** This is the load-bearing difference from `session_assign`/`session_start`,
  where `ack{ok:false}` fails the session: **a rejected swap never fails the session.** The control
  plane surfaces the swap as failed to the caller but does not change session state.

**`session_state` callbacks during a swap** (top-level `state` stays `running` throughout — swap
does not change the reservation; see `schema.md`):
- `session_state{ state:"running", detail:"swapping" }` — swap in progress.
- `session_state{ state:"running", detail:"<running detail>" }` — **swap succeeded**; the new app
  is the source. The control plane sets `sessions.app_id` to the new app on this callback.
- `session_state{ state:"running", detail:"swap failed; rolled back" }` — swap failed but the
  agent **restored the previous app**; the session keeps running it. `app_id` is unchanged. (Per
  the state machine, `error` is set only with `failed`, so the failure reason rides in `detail`.)
- `session_state{ state:"failed", error:"<reason>" }` — swap failed and the previous app could
  **not** be restored; the session is terminal and its reservation is released (the normal `failed`
  path). **Roll back when possible; only fail the session when rollback is impossible.**

### `config_update` — push host override knobs (host-runtime-settings-admin)
> *Additive amendment. New downstream message; no change to any existing message. Fire-and-
> forget — no `ack` is expected. An older agent that does not recognise the `type` ignores it
> (existing discipline for unknown message types).*
>
> **Amendment — #194 (host_settings env precedence): `settings` is now SPARSE.** It previously
> carried the *full resolved knob block* (override-or-catalog-default for every key). That made
> the catalog default authoritative over the agent's `QUASAR_*` env: a GPU host with
> `QUASAR_ENCODER=nvenc` and **no** admin encoder override received `encoder: "openh264"` (the
> catalog default) and silently did software encode. `settings` now carries **only the host's
> explicit overrides**; the agent overlays them onto its **env-derived baseline**. Resolution
> precedence is **stored-override → agent env → catalog default**. No message shape change (still
> `{type, settings}`); only the *content* (sparse) and the agent's overlay semantics (onto env,
> re-based each push) change. Requires sign-off (frozen interface).

Delivers the host's **sparse override block** to the agent: only the knobs an admin has
explicitly set via `PATCH /v1/admin/hosts/{id}/settings`. A knob that is **not** present keeps
the value the agent derived from its own `QUASAR_*` environment (its baseline). The agent
re-derives that baseline and overlays the overrides on **every** push, so clearing an override
(it disappears from the block) reverts the knob to the agent's env value — never to the catalog
default. An empty `settings` (`{}`) means "no overrides" and the agent runs purely on its env.

The control plane sends `config_update`:
1. **Once, immediately after `registered`** — so the agent has its override snapshot before any
   session is assigned, without a synchronous REST fetch.
2. **Again on every admin `PATCH /v1/admin/hosts/{id}/settings`** that changes a live-class
   knob — the agent re-derives its env baseline, overlays the overrides, and applies the result
   to the next session build.

```json
{
  "type": "config_update",
  "settings": {
    "encoder": "va",
    "gop": 120
  },
  "console_config": {
    "enabled": true, "connector": "DP-4", "compositor": "weston",
    "audio_output": "hw:1,3", "stream": false, "stream_audio": false,
    "input_devices": "auto", "grab": true,
    "auto_start_on_display": false, "auto_connect_controller": false,
    "default_app": "<uuid|null>", "fullscreen": true
  }
}
```
- **`console_config`** *(NEW, CM-01, optional, additive)* is the host's resolved console-mode
  configuration (`schema.md` `console_config`; `control-api.md`
  `PATCH /v1/admin/hosts/{id}/console-config`). **Absent ⇒ console mode disabled** — an agent
  that never receives it behaves exactly as today. The node-agent reads this block **instead of**
  the spike's `QUASAR_LOCAL_DISPLAY` env hardcode (env retained as a dev/bootstrap fallback only).
  Unlike `settings`, `console_config` is the **full resolved object** (not sparse) — the control
  plane applies defaults before sending. It is **not restart-class**: it applies on the next
  session build, and live changes to `auto_start_on_display`/`auto_connect_controller` re-arm the
  agent's hotplug watcher on receipt — no agent restart. `stream:false` (local-only) means the
  session's encode/`webrtcbin` leg is simply not built.
- **`capacity.console_capabilities.outputs`** *(Wave 3.2, optional, additive)* is the typed DRM
  inventory. Each output is card-scoped (`id`, `card`, associated `render_node`, connector and
  connection state) and contains exact DRM mode timing identity (`width`, `height`, integer
  `refresh_millihz`, preferred/interlaced flags, pixel clock and totals). Legacy
  `connectors: string[]` remains for hotplug and older-control-plane compatibility.
- **`settings`** is the host's **sparse override map** — only admin-set knobs appear. Keys absent
  from the map are left at the agent's env baseline. A nullable knob that an admin clears is
  simply omitted (not sent as `null`). The agent recomputes `RuntimeSettings::baseline()` (env)
  then overlays this map on each push, so the result is `env ← overrides`, not a full replace.
  The admin API's `resolved` response field still reports the *display* view
  (catalog-default ← overrides); it does not reflect the agent's env (the control plane does not
  see it). The agent's effective config is surfaced via the `capacity` message's
  `effective_settings` field (host-observability amendment above) and returned as `effective`
  in `GET /v1/admin/hosts/{id}/settings`.
- **Live-class knobs** (`abr_enabled`, `gop`, `slices`, etc.) take effect on the **next session
  build**. Running sessions are never modified mid-life — the pipeline is static once built.
- **Restart-class knobs** (`encoder`, `render_node`, `cuda_device`) are read at the agent's
  **first session build** (inside the `ensure_gst_init` call, which runs the `gst::init` process-
  wide `Once`). A `config_update` carrying a new restart-class value after sessions have already
  run cannot retro-actively change them — those values are latched for the process lifetime.
  Changing them requires the `restart` command below (the admin PATCH triggers it).
- **Env baseline is authoritative absent an override.** If the agent has not yet received a
  `config_update` when its first session is assigned (race window on a newly enrolled agent), it
  runs on its env-var baseline — and that same baseline is the floor under every push. An existing
  deployment that never touches `PATCH /v1/admin/hosts/{id}/settings` behaves exactly as its
  `QUASAR_*` env dictates (this is the #194 fix: the catalog default no longer overrides env).

### `restart` — graceful agent exit to apply restart-class config (host-runtime-settings-admin)
> *Additive amendment. New downstream command; no change to any existing message. Unlike
> session commands, `restart` is handled directly in the agent's WS receive loop (not in
> `handle_control`) so the ack can be flushed before exit.*

Instructs the agent to acknowledge and then exit cleanly. The container's restart policy
(typically `always` or `unless-stopped`) restarts the process, which re-registers with the
control plane and receives a fresh `config_update` — so the new process builds its first
session with the updated encoder/render-node/CUDA-device.

Only used for **restart-class knobs** (`encoder`, `render_node`, `cuda_device`), because those
are read once inside `ensure_gst_init` (which calls the process-wide `gst::init` `Once`) and
cannot be changed in a running process without a restart.

```json
{ "type": "restart", "id": "<cmd-id>" }
```
**Ack semantics.** The agent replies immediately before exiting:
```json
{ "type": "ack", "id": "<cmd-id>", "ok": true }
```
The agent flushes the ack frame, then calls `std::process::exit(0)` (or equivalent). The
control plane should not treat the subsequent WS disconnect as a fault — it is the expected
outcome and sets `hosts.pending_restart = true` until the agent reconnects. On reconnect, the
agent re-registers, the control plane sends `config_update` with the new settings, and
`pending_restart` is cleared.

Running sessions are **not** forcibly stopped by the `restart` command. Operators who want a
clean cut should drain the host first (`POST /v1/hosts/{id}/drain`, `control-api.md`) before
changing a restart-class knob.

---

## Signaling relay (ties P1-A to P1-D)
WebRTC signaling is a **control-plane** responsibility (architecture §"Control plane"), but the
SDP offer/ICE originate at the node agent's `webrtcbin`. So the control plane **bridges**: the
browser connects to the control-plane signaling WebSocket (authenticated by the single-use
session token, see `signaling.md`), and the control plane relays each signaling message to/from
the assigned node over *this* agent connection, tagged with `session_id`. The node never needs a
public address — the control plane is the single ingress (essential for NAT/K8s).

A `signaling` envelope wraps the signaling message (`offer`/`answer`/`ice`/`restart_ice`/
`bye`/`error`):
```json
// control -> node (client's answer / ICE, and a "client attached" notice)
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "answer", "sdp": "..." } }
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "ice", "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } } }
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "restart_ice", "pc": "video" } }

// node -> control (host is the offerer; forwarded to the browser)
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "offer", "sdp": "..." } }
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "ice", "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } } }
```
The inner `msg` is exactly what `signaling.md` defines — the relay adds only the `session_id`
routing tag. Input events and clock-sync (`input.md`) ride the WebRTC DataChannel directly
peer-to-peer (browser ↔ node `webrtcbin`); they do **not** traverse this relay.

When a replacement signaling WebSocket attaches to an already-running session, the control plane
notifies the node through the existing session-scoped relay. The node emits fresh offers for its
live video and audio peer connections; no container or media pipeline restart is implied.

## Reconnection & reconciliation
- On agent reconnect, the control plane trusts the agent's `heartbeat.running_sessions` /
  fresh `capacity` as ground truth for that host: sessions the control plane thinks are
  `running` but the agent doesn't list are marked `failed` (reservation released); sessions the
  agent runs but the control plane lost are stopped (`session_stop`) — the control plane is the
  scheduling authority.
- If the agent connection is lost for longer than the heartbeat-miss threshold, the host goes
  `offline` and all its non-terminal sessions become `failed`. This is the multi-host failure
  model at N=1.

## Phase 1 simplifications (revisit, do not bake in workarounds)
- One agent connection per host process; the control plane addresses sessions by `session_id`.
- Enrollment is a shared token; mTLS/SPIFFE is the Phase 3/4 upgrade (same message shape).
- The scheduler is trivial (one host, one GPU) but the assign/reserve/release path is the real
  one — N=1 is one row, not a special case.
