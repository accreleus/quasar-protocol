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

## Transport: one persistent, node-initiated WebSocket
The node agent **dials** the control plane and holds open a single WebSocket; all agent-API
traffic flows over it, in both directions. JSON, one message object per WS frame, discriminated
by `type` — identical encoding discipline to `signaling.md`/`input.md` (debuggable, cheap-tier
friendly).

Why node-initiated and persistent:
- **NAT / K8s friendly.** The control plane never has to reach *into* a node. Nodes behind NAT,
  or as a DaemonSet pod dialing a control-plane Service, work with zero extra plumbing. This is
  exactly why Wolf-on-K8s was all workarounds and this is not.
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
  "host": { "cpu_cores": 16, "mem_mb": 64000 },
  "gpus": [
    { "index": 0, "vendor": "amd", "model": "Radeon Pro V520",
      "vram_mb_total": 16384, "encode_slots_total": 2 }
  ]
}
```
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
  "resources": { "vram_mb": 1024, "encode_slots": 1 }
}
```
`app` mirrors `apps.runtime_spec`; `stream` mirrors the `sessions` launch-param columns;
`resources` mirrors the reserved amounts. The agent must not exceed `resources` (it's the
budget the control plane reserved). `gpu_index` maps to the agent's local GPU enumeration
(same index it reported in `capacity`).

### `session_start` — bring the pipeline up
```json
{ "type": "session_start", "id": "<command-id>", "session_id": "<uuid>" }
```
The agent builds compositor→encode→`webrtcbin` (the graduated P1-5 pipeline), then reports
`session_state: starting` → `running`. Once `running`, the agent's `webrtcbin` is the offerer and
the signaling relay (below) can carry the offer to the client.

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

---

## Signaling relay (ties P1-A to P1-D)
WebRTC signaling is a **control-plane** responsibility (architecture §"Control plane"), but the
SDP offer/ICE originate at the node agent's `webrtcbin`. So the control plane **bridges**: the
browser connects to the control-plane signaling WebSocket (authenticated by the single-use
session token, see `signaling.md`), and the control plane relays each signaling message to/from
the assigned node over *this* agent connection, tagged with `session_id`. The node never needs a
public address — the control plane is the single ingress (essential for NAT/K8s).

A `signaling` envelope wraps the **verbatim Phase-0 signaling message** (`offer`/`answer`/`ice`/
`bye`/`error` — shapes unchanged):
```json
// control -> node (client's answer / ICE, and a "client attached" notice)
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "answer", "sdp": "..." } }
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "ice", "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } } }

// node -> control (host is the offerer; forwarded to the browser)
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "offer", "sdp": "..." } }
{ "type": "signaling", "session_id": "<uuid>", "msg": { "type": "ice", "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } } }
```
The inner `msg` is exactly what `signaling.md` defines — the relay adds only the `session_id`
routing tag. Input events and clock-sync (`input.md`) ride the WebRTC DataChannel directly
peer-to-peer (browser ↔ node `webrtcbin`); they do **not** traverse this relay.

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
