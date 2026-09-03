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

> **Amendment — session-display-update (live render resolution / UI scale), additive, pre-approved
> 2026-08-15.** Adds one control→node message, `session_swap_app`'s sibling `session_display_update`
> (§Messages — control → node), with the same rejected-is-a-no-op ack contract as the swap. **Additive**
> — a new `ControlMsg` variant; no existing message, field, or the assign/start/stop/swap ack contract
> changes. The encode caps, interpipe boundary, and stream WxH are **untouched** — only the
> compositor's advertised `wl_output` logical mode and `wp_fractional_scale_v1 preferred_scale`
> change. Both values are **ephemeral** (agent-held only; not persisted in `sessions`, not on the
> `Session` resource) and are echoed back, when non-default, on `session_metrics` (§`session_metrics`)
> as the sole authoritative readback. See `control-api.md` `PATCH /v1/sessions/{id}/display`.

> **Amendment — session-display-stream (live external/encoded resolution), additive, approved 2026-08-16 (PR #15)
> 2026-08-16, approved 2026-08-16 (PR #15).** Sibling of `session_display_update`'s render-resolution half:
> adds `stream_width`/`stream_height` to that same message (§`session_display_update`) and two
> new optional `session_metrics` fields — `stream_width`/`stream_height` (present when ≠ launch)
> and `external_resize_supported` (bool, always present from a capable agent; absent from an
> older one, meaning *unknown*). **Additive** — new optional fields on an existing downstream
> message and an existing upstream report; no existing field, message, or ack contract changes.
> Unlike render resolution/UI scale, this pair changes what the **encoder** actually produces —
> the coded size moves at the **next IDR**, with **no WebRTC renegotiation** (`signaling.md`
> unchanged) — so it is gated on encoder support: an agent whose encoder cannot live-resize
> rejects with `ack{ok:false, error:"display_update_rejected: encoder does not support live
> resize"}` rather than attempting it. See `control-api.md` `PATCH /v1/sessions/{id}/display` for
> the INTERNAL-vs-EXTERNAL vocabulary and the fixed rung table. **The in-session ABR governor
> does not drive this yet** — manual/API lever only; a later amendment may add automatic stepping,
> and this message shape is what it would reuse. **APPROVED (Michael, 2026-08-16) — PR #15 merged; do
> implement against this until it is merged.**

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

> **Amendment — multi-codec (HEVC/AV1), additive, requires sign-off (signed off 2026-07-25).** Adds
> two optional, additive fields so a session can stream a codec other than H.264 while every
> existing H.264 session and every older agent behaves byte-identically. **(1)** The `session_assign`
> `stream` block gains **`codec`: `"h264" | "h265" | "av1"`** (omitted, or empty, defaults to
> `"h264"`): the single video codec the agent must encode for the session, resolved server-side at
> launch. `h264_profile` stays H.264-only and is ignored when `codec` names a non-h264 value. An
> unrecognised value fails the assignment (the agent must never silently produce a different codec
> than `sessions.codec` records). **(2)** The agent `capacity` report gains **`codecs`: `["h264",
> ...]`**, the wire codec set the host's *active encoder path* can actually produce (GStreamer
> registry probe: encoder element found AND payloader present). Omitted defaults to `["h264"]` so an
> old agent keeps working. Both are additive; no existing field, message, or ack contract changes.
> `h265` is the wire spelling of HEVC (the control-plane profile catalog's `hevc` maps to `h265` on
> this wire). See §`session_assign` and §`capacity`.

> **Amendment — per-codec encoder throughput hint (#506), additive, requires sign-off.** Adds ONE
> optional, additive field to the agent `capacity` report: **`codec_throughput`**, a map from a wire
> codec (`"h264" | "h265" | "av1"`) to `{ "max_pixel_rate_mpix_s": <number> }` — the sustained encode
> throughput, in megapixels per second, the host's *effective encoder element for that codec* can hold
> at session settings. A host advertises a codec (`capacity.codecs`) without any notion of how fast it
> encodes it, and the throughputs are not comparable: measured on an RTX 5090 (driver 610.57.04),
> `vulkanh265enc` sustains ~395 Mpix/s while `vulkanh264enc` sustains ~1400 and `vulkanav1enc` ~1215.
> 1440p120 needs 442 and 2160p60 needs 498, so an h265 session at either tier runs *below tier* with
> zero drops and zero freezes — the compositor is back-pressured by the encoder rather than dropping,
> so the shortfall is silent. The hint lets the control plane reject such a codec at launch and fall
> through its rung ladder to one the host can actually sustain. Absent field, absent codec key, or a
> non-positive value all mean **unknown**, and unknown gates nothing. See §`capacity`.

> **Amendment — image management P2 (ensure images onto hosts), additive, requires sign-off
> (signed off 2026-08-08).** Adds two downstream (control → node) messages — `image_ensure`
> (pull a prebuilt catalog image into the host's docker daemon) and `image_remove` (best-effort
> removal) — one upstream message, `image_state` (pull progress / terminal state), and one
> optional field on `register`: `images: [{image_id, version, state}]` so a reconnecting agent
> reports what it actually has and the control plane reconciles `host_images` against reality
> (same wholesale-snapshot principle as `capacity`). **Additive** — new `type` values and one
> optional field; no existing message, field, or ack contract changes. An older agent never
> receives `image_ensure` (the control plane only sends it to agents that reported `images` on
> register or acked a prior ensure — conservatively, it may send and tolerate silence, since an
> unknown downstream `type` is ignored by existing discipline) and never sends `image_state`;
> an older control plane ignores both unknown upstream shapes. `registry_ref` is always a
> concrete immutable sha- tag, never a floating tag. See §`image_ensure`, §`image_remove`,
> §`image_state`, §`register`, and `docs/design/plans/2026-08-08-image-management-p2-spec.md`
> in the quasar repo.

> **Amendment — image management P4 (template builds), additive, requires sign-off (delegated
> sign-off 2026-08-08 overnight campaign, flagged for review).** Adds one downstream (control →
> node) command, `image_build` — the template analogue of `image_ensure`: instead of pulling a
> prebuilt ref, the agent fetches a build context, runs `docker build` locally, and reports the
> same `host_images` state machine as a pull. It reuses `image_state` verbatim, sending the
> `building` state that P2 **already reserved** in the wire enum note and the schema CHECK — a P2
> control plane accepts `building` but never sent it; **P4 is the first sender.** No existing
> message, field, or ack contract changes: `image_ensure`, `image_remove`, and `image_state` keep
> their exact shapes; `image_build` is a new sibling `type`. An older agent that ignores
> `image_build` never becomes ready for a template image — wire-silent, exactly as for
> `image_ensure` (the control plane's ack-timeout "unsupported until next register" logic covers
> it). `image_remove` removes a template image by the `local_tag` the agent recorded, exactly as
> it removes a pulled `registry_ref`. See §`image_build`, §`image_remove`, §`image_state`, and
> `docs/design/plans/2026-08-08-image-management-p4-spec.md` in the quasar repo.

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
- **Enrollment (first contact):** the agent presents an `enrollment_token` plus its stable
  `node_name`. The token is **either** a per-host token minted by an admin
  (`control-api.md` §Host enrollment tokens — hashed, single-use by default, expiring,
  optionally bound to this `node_name`; the primary path since 2026-09-03) **or** the
  fleet-wide static value from control-plane config (delivered out of band; kept as a
  fallback so existing deployments upgrade untouched, and itself optional — a deployment may
  run with minted tokens only). The wire shape is identical for both. The control plane
  creates the `hosts` row, mints a per-node `node_secret`, returns it in `registered`, and
  stores `node_secret_hash`. The agent persists the secret locally.
  **Enrollment onto a `node_name` whose agent is currently live is refused** (#96):
  enrollment rotates `node_secret`, so allowing it against a live host is identity takeover.
  The credential is checked FIRST — this surface is pre-auth, so a bad token gets a plain
  `auth_failed` that says nothing about whether the `node_name` exists or is live.
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

**Optional `images` array (image-management P2 amendment).** The agent may include
`"images": [{ "image_id": "steam", "version": "2026.08.07", "state": "ready" }, ...]` — the
managed catalog images it actually has (verified against its docker daemon, by `image_id`s it
was previously told to ensure). When present, the control plane reconciles its `host_images`
rows for this host **wholesale for the reported ids**, and any id the control plane believed
`ready` on this host but absent from the report is flipped to `absent` (a reconnected agent
that lost an image must not read as ready). Absent field ⇒ the control plane keeps its stored
rows unchanged (older agent).

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
  "codecs": ["h264", "h265"],
  "codec_throughput": {
    "h264": { "max_pixel_rate_mpix_s": 1400 },
    "h265": { "max_pixel_rate_mpix_s": 395 }
  },
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

`codecs` *(NEW, multi-codec, optional, additive)* is the **wire** codec set the host's active
encoder path can produce: a subset of `["h264", "h265", "av1"]` (`h265` is HEVC on the wire; the
control-plane profile catalog's `hevc` maps to it in one place in the Go code). A codec is reported
only when both its encoder element is registered for the agent's active `EncoderChoice` (or, for the
Vulkan encoder configured with AV1, its per-session vendor-encoder fallback) AND its RTP payloader
is registered (`rtph265pay` for HEVC, `rtpav1pay` for AV1), so a missing plugin is self-describing
(the codec is simply absent from the set until the image carries it). The agent probes this from the
GStreamer registry at startup and **re-probes and re-sends `capacity`** whenever a `config_update`
flips its effective encoder (a restart-class encoder change reaches the process on reconnect, but a
live effective-encoder change is re-probed in place). Absent ⇒ the control plane assumes `["h264"]`
(an old agent that never reports the field is treated as H.264-only; the control plane keeps its
last-stored value rather than clobbering it when a later report omits the key). H.264 is always
producible, so the set is never empty in practice.

`codec_throughput` *(NEW, #506, optional, additive)* is the per-codec **sustained encode throughput
hint**: a map keyed by the same wire codec strings as `codecs`, each value an object carrying
`max_pixel_rate_mpix_s` — megapixels per second the codec's *effective encoder element on this host*
can sustain at session settings (CBR, one reference frame, no B-frames — the properties the agent
actually configures). An entry may be **measured** (an encoder probe on this host) or
**vendor-estimated** (a compiled-in table keyed by encoder element, which is what the agent ships
today); the wire does not distinguish the two, because a consumer must treat both the same way — as
a conservative capability ceiling, not a promise. The object form exists so a future measured probe
can add fields (a measurement timestamp, a confidence) without a shape change.

Semantics, all of which resolve toward *not* gating:

- **Absent field** ⇒ pre-amendment agent. The control plane keeps its last-stored value (same
  keep-if-absent discipline as `codecs`/`effective_settings`/`storage`); an explicit `{}` is a real
  overwrite to "nothing known". **Keep-if-absent means a DOWNGRADE leaves a stale hint in place:** a
  host rolled back to a pre-amendment agent cannot send `{}`, so whatever it last reported keeps
  gating that host's codec decisions until a hint-capable agent re-registers. The exposure is
  bounded — the stored value is a throughput ceiling, so a stale one costs at most a tier and never
  a failed launch — but an operator debugging an unexpected `encoder_throughput` rejection on a
  downgraded host should clear `hosts.codec_pixel_rates` rather than look for a live report.
- **A codec present in `codecs` but absent from `codec_throughput`**, or an entry whose
  `max_pixel_rate_mpix_s` is missing, non-numeric or ≤ 0 ⇒ **unknown**. Unknown never rejects
  anything.
- **A codec present in `codec_throughput` but not in `codecs`** is meaningless and ignored — the
  advertised set is what `codecs` says.

The agent re-sends `capacity` (and therefore this map) on the same triggers as `codecs`, including a
`config_update` that flips the effective encoder, since the map is keyed on the elements sessions
actually build. The hint is **advisory to the codec decision only**: it never blocks registration or
placement, and it never removes a codec from the host's advertised set. The control plane's use of it
is `control-api.md` `codec_decision.considered[].rejected_by = "encoder_throughput"`.

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

`readiness` *(NEW, first-run-experience §S1, optional, additive)* is the agent's **advisory**
host-readiness report: an array of `{ id, status, summary, remediation }`, where `id` is a stable
machine key (e.g. `"nvidia_egl_vendor_json"`), `status` is `"pass" | "fail" | "skip"` (`skip`
means "not applicable to this host", e.g. an NVIDIA check on an AMD box — never "we could not
tell"), `summary` is one operator-actionable sentence, and `remediation` is an exact fix command
(empty for `pass`/`skip`). **The check set is agent-owned** — a consumer must pass an
unrecognized `status` through rather than reject it, so adding, renaming, or removing a check
never needs a protocol amendment. Absent ⇒ the control plane keeps its last stored value
(keep-if-absent, exactly like `effective_settings`); an explicit `[]` is a real overwrite
("reported, nothing to say"), distinct from "never reported". **ADVISORY ONLY**: a failing check
MUST NOT affect registration, admission, or scheduling — a host with every check failing still
registers and still runs sessions. See `readiness.rs`'s probe set and `schema.md` `hosts.readiness`.

### `heartbeat` — liveness + live utilization
```json
{
  "type": "heartbeat",
  "running_sessions": ["<session_id>"],
  "ts_unix_ms": 1735689600000,
  "gpu_vram": [ { "index": 0, "used_mb": 1840, "free_mb": 30928 } ]
}
```
Sent every `heartbeat_interval_ms`. Updates `hosts.last_heartbeat_at`. `running_sessions` lets
the control plane reconcile its view against the agent's ground truth (detect orphans both ways).
Missing N consecutive heartbeats ⇒ host `offline`, its sessions `failed`, reservations released.

`gpu_vram` *(NEW, #383, optional, additive)* — a live per-GPU memory sample, `index` matching the
GPU's index in the `capacity` report. `used_mb` / `free_mb` are each **optional**: an omitted or
null value means the agent could not measure that figure, which is materially different from
zero, and the control plane must treat it as unknown rather than as "full". An agent that omits
`gpu_vram` entirely is simply an agent that predates this field.

Sources: AMD reads `<sysfs device>/mem_info_vram_used`; NVIDIA runs one
`nvidia-smi --query-gpu=pci.bus_id,memory.used,memory.free` for all GPUs. On an AMD APU
`mem_info_vram_total` is the BIOS UMA carve-out rather than a real pool, so the reported figures
are honest but not representative of where the workload's memory lives — consumers must not
assume the sample bounds a session's actual usage.

The control plane stores these on `gpus` (`schema.md`) and uses them for the live free-VRAM
admission veto (`control-api.md` §Admission control). Two properties are contractual:

- **The sample is advisory, not a reservation.** Encode slots remain the only race-safe
  reservation. A sample is a point-in-time observation and two launches a second apart see the
  same one.
- **Unknown or stale telemetry must fail open.** A missing field, a missing sample, or a sample
  older than the control plane's freshness window means the veto abstains and encode slots decide
  alone. An agent whose sampler breaks must never be able to strangle its own host.

Sampling must not delay the heartbeat. `nvidia-smi` can block for seconds on a loaded or
Xid-faulted GPU, and a heartbeat stalled past the read deadline marks the host offline and reaps
its sessions — so the agent samples off the control path and attaches whatever result is already
available.

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

> *(first-run-experience §S5, additive, 2026-08-09)* `session_state` may also carry two optional
> fields, both present only on a terminal report that warrants them:
> - **`reason_code`: string** — a machine-readable classification of a terminal failure. Today's
>   only defined value is `"app_exited_early"` (the app container exited before it ever presented
>   a frame — #463). **Never load-bearing for lifecycle**: state transitions key on `state` alone,
>   never on `reason_code`.
> - **`app_log_tail`: string** — the app container's own captured log tail, newline-joined,
>   oldest first, bounded to roughly the last 100 lines. Captured continuously while the
>   container ran, because app containers run `--rm` and the daemon has already discarded the
>   lines by the time anyone looks otherwise — this is the only surviving copy of the decisive
>   output (e.g. "Steam needs to be online to update").
>
> Both are omitted (not `null`) on every other report and on every pre-amendment agent — an older
> control plane simply never sees the keys. See `schema.md` `sessions.failure_code` /
> `sessions.app_log_tail` for the stored form (the wire spells the classification `reason_code`;
> the column is `failure_code`).

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
  "encode_ms_p50": 4.4,
  "encode_ms_p95": 5.2,
  "source_fps": 60.0,
  "compositor_fps": 60.0,
  "compositor_pts_delta_p50_ms": 16.67,
  "compositor_pts_delta_p95_ms": 16.72,
  "interpipe_queue_level_max": 1,
  "interpipe_queue_dwell_p50_ms": 0.08,
  "interpipe_queue_dwell_p95_ms": 0.16,
  "interpipe_queue_drops": 0,
  "rtp_fps": 60.0,
  "rtp_bitrate_kbps": 15040,
  "frames_encoded": 299,
  "frames_dropped": 1,
  "app_launch_state": "game_running",
  "bytes_used": 123456789,
  "render_width": 1280,
  "render_height": 720,
  "ui_scale": 1.5
}
```
- **Lightweight fields (always present):** realized `fps`, actual `bitrate_kbps`, mean
  `encode_ms` over the window (from the encode pad-probe FIFO — *timing only*, no overlay
  write), `frames_encoded`, `frames_dropped`. `window_ms` is the aggregation period — it
  **equals the emit cadence** (`heartbeat_interval_ms`) so windows are contiguous and
  non-overlapping. These are the always-on signal the design promised — collectible with **no
  overlay stamping**, so the default hot path is unchanged (the Phase-0 invariant; see `P4-03`).
  The exact `metrics` JSONB keys are enumerated in `schema.md`'s field dictionary — implementers
  use those names verbatim. Browser-side capture-to-display telemetry uses RVFC `captureTime`
  when available; it is not strict abs-capture-time RTP-extension evidence until the browser
  records SDP/RTP wire proof (`control-api.md`). No host-side overlay is used.
- **Correlated stage-budget fields (Wave 4.1):** `source_fps` is the delta-sampled cadence of
  new buffers committed by the mapped fullscreen application's top-level Wayland surface;
  `compositor_fps` and compositor PTS-delta p50/p95 measure frames emitted by
  `waylanddisplaysrc` before caps normalization and interpipe. Cursor, popup, subsurface, and
  configure-only commits do not increment `source_fps`. Maximum interpipe queue level, queue dwell p50/p95 and drop count; encode-time
  p50/p95; and RTP access-unit fps plus wire bitrate. Optional percentile fields are omitted
  in an empty window. Counters and maxima are per-window, bounded, and observational only.
- **The field dictionary of record is `docs/session-trace/metrics.json` in the quasar repo — the
  *metric manifest*.** Every key of this message is listed there with its unit, the clock its
  value sits on (`host_monotonic`, `gst_pts`, …), the window it summarises, the estimator that
  produced it and the key carrying its sample count. Three traps this message carries that no key
  name reveals, all now written down there: `compositor_pts_delta_p50_ms`/`_p95_ms` are **nominal**
  PTS spacing off the shared pipeline clock (a flat ~16.668 ms at 60 fps however badly the realized
  cadence jitters — `probe_compositor_frame_interval_p95_ms` is the realized one);
  `frames_dropped` here is **encoder-side**, detected by a 500 ms pending-FIFO timeout because no
  hardware or software H.264 encoder exposes a dropped-frame property, and shares its name with a
  completely different client-side key; and `window_ms` and `bytes_used` travel on this message but
  are **not written into the stored `metrics` JSONB** at all. Keys documented in this section that
  no agent has ever emitted are named as such in the manifest rather than left to look absent —
  today that is `app_launch_state` (below), which is specified here and implemented nowhere.
- **`app_launch_state`** *(optional string)* — coarse in-container application launch
  state, when the app image reports one:
  `"starting" | "client_only" | "game_running" | "game_exited"`.
  `starting`: the app container is up but the application has not finished its own
  startup. `client_only`: an intermediary client (e.g. the Steam client) is up but the
  target title is not running. `game_running`: the target title is running (and, where
  the image can tell, foreground). `game_exited`: the target title ran and has exited.
  **Omitted entirely** when the image does not report launch state — absence means
  *unknown*, and consumers MUST treat absence and `"unknown"`-like values identically
  (fall back to transport-level readiness). It is a **hint and a metric, never a
  session-state authority and never access control**: `session_state` remains the only
  progress signal, and a session MUST function identically if the field never appears.
  Detection is best-effort by construction (launcher-shim titles, out-of-tree
  executables, and client quirks produce false negatives); consumers MUST NOT fail a
  session on its value. An unrecognized value is treated as absent, not an error.
- **`bytes_used`** *(optional, P5-03)* — last reported managed-home usage in bytes: the
  agent's post-session measurement of the session's home directory (the `local` driver
  measures the host path under the effective `QUASAR_HOME_ROOT`; `volume`-driver usage is
  measured the same way via the mounted path). Emitted **once**, in the pre-terminal sample
  just before the session goes `stopped`, and omitted from every other sample and whenever
  the effective home root is empty. The control plane updates `user_homes.bytes_used` on
  receipt (`schema.md`); freshness is **advisory** — a quota, if ever added, is enforced
  against the last report, and storage usage is visibility-only today (`control-api.md`).
  *Documented here as doc-drift repair: this field has been on the wire since P5-03; no
  wire change.*
- **`render_width` / `render_height` / `ui_scale`** *(optional, session-display-update)* — the
  compositor's **current** app-facing `wl_output` logical mode and `preferred_scale`, present
  **only when non-default**, i.e. render size ≠ the session's pinned stream size, or
  `ui_scale ≠ 1.0` — same omit-when-default convention as the other optional correlated fields
  above. Set by `session_assign` (default = stream size / `1.0`) and moved only by
  `session_display_update` (§`session_display_update`). **These fields are the ONLY
  authoritative readback of the live render resolution / UI scale** — neither value lives in
  `session_state`, the `Session` resource, or the `sessions` table (both are agent-held,
  ephemeral state); a consumer that wants to know the current render size or UI scale reads the
  latest `session_metrics` sample, or otherwise trusts its own last-acked value.
- **`stream_width` / `stream_height`** *(optional, session-display-stream, approved 2026-08-16)* — the
  encoder's **current** external/streamed size, present **only when non-default**, i.e. ≠ the
  session's launch size — same omit-when-default convention as `render_width`/`render_height`
  above. Set by `session_assign` (default = launch size) and moved only by
  `session_display_update`'s `stream_width`/`stream_height` (§`session_display_update`). This is
  the **authoritative** readback the control plane's `Session.stream.external_width`/
  `external_height` in-memory cache is populated from (`control-api.md`); the cache is a
  convenience, this sample is the source of truth.
- **`external_resize_supported`** *(bool, session-display-stream, approved 2026-08-16)* — whether this session's
  encoder can live-resize the stream at all. **Always present from a capable agent** (unlike the
  omit-when-default fields above, this one is a capability flag, not a value that has a default to
  omit); **absent from an older agent**, which a consumer treats as *unknown*, not `false`.
  Readback surfaces as `Session.stream.external_resize_supported` (`control-api.md`).
- **`ladder_speed_bias`** *(int, optional)* — the SPT-08 ladder's current encoder
  speed-bias rung (0 = the session's configured quality posture). **Present only when
  non-zero**; absent means the baseline posture, never "unknown". Signal-only for the
  control plane: nothing server-side reacts to it.
- **`ladder_res_rung`** *(int, optional)* — the ladder's current EXTERNAL
  resolution rung index into the session's aspect family (`0` = the launch size).
  **Present only when non-zero.** The rung's realized pixel size is still reported by
  `stream_width`/`stream_height`; this is the ladder's own index, which is what makes a
  step distinguishable from a manual PATCH landing on the same size.
- **`external_owner`** *(string, optional, `"auto" | "pinned"`)* — who currently
  owns the external size. `"auto"` = the ABR ladder may move it; `"pinned"` = a user or
  admin set it via `PATCH /v1/sessions/{id}/display` and the ladder will not move it.
  **Present only alongside `stream_width`/`stream_height`** (at the launch size there is
  nothing to own). An agent that omits it on a non-default size is pre-ladder: treat the
  size as `"pinned"`, since only a manual PATCH could have moved it.
- **(approved 2026-08-16) `ladder_fps`** *(int, optional)* — the ladder's realized target
  frame rate when the fps rung has stepped below the session's launch fps. **Present only
  when below the launch fps.** Emitted since the fps rung shipped (2026-08-17); absent on
  hosts where the rung is disabled or inert.
- **(approved 2026-08-17) `abr_floor_kbps`** *(number, optional)* — the ABR governor's
  **current** lower bound in kbit/s, when the adaptation ladder has moved it off the floor
  the session launched with. **Present only when ≠ the launch floor**; absent therefore
  means "the floor this session launched with", never "unknown" — the same
  omit-when-default convention as `ladder_fps` above. The companion to
  `abr_setpoint_kbps`: without it a consumer cannot tell a setpoint that was *chosen* from
  one that is *pinned at the floor*, because since the floor-follows-rung amendment the
  floor is not a session constant. Signal-only for the control plane — nothing server-side
  reacts to it; it is stored verbatim into the metrics JSONB for the diagnostic bundle and
  soak reports.
- **(approved 2026-08-19) host-stage latency probe keys** — eleven optional numeric
  fields, emitted only while the node-agent's host-stage latency probe is enabled
  (`QUASAR_LATENCY_PROBE=1`, or the `latency_probe` host setting — Live class). All
  values are milliseconds on the host monotonic clock, aggregated over `window_ms`.
  The nine stage keys are absent when the probe is off **and** absent for any window in
  which that stage collected no sample — absent means "not measured", never zero (the
  `abr_floor_kbps` convention):
  - **`probe_capture_to_enc_in_p50_ms`** / **`probe_capture_to_enc_in_p95_ms`** — per-frame
    elapsed from the compositor element's src pad push to the encoder's sink pad,
    correlated on buffer PTS (interpipe boundary + leaky queue + GPU convert/scale).
  - **`probe_enc_out_to_send_p50_ms`** / **`probe_enc_out_to_send_p95_ms`** — encoder src
    pad to that frame's access unit leaving the post-`rtpbin` egress seam (immediately
    before SRTP). Differs from `probe_pay_to_send_*` by the parser + payloader cost.
  - **`probe_pay_to_send_p50_ms`** / **`probe_pay_to_send_p95_ms`** — first RTP packet of
    an access unit at the payloader to that access unit's marker packet at the egress
    seam (RTP session processing, TWCC insertion, pacer).
  - **`probe_pts_to_emit_p50_ms`** / **`probe_pts_to_emit_p95_ms`** — pipeline clock time
    at the compositor src pad push minus the buffer's PTS (mapped through base time): the
    compositor's own internal hold.
  - **`probe_compositor_frame_interval_p95_ms`** — p95 wall-clock interval between
    consecutive buffers at the compositor src pad (realized cadence; contrast
    `compositor_pts_delta_p95_ms`, which is PTS-derived and reports the nominal).
  - **`probe_send_desyncs`** / **`probe_pts_unmatched`** *(number, optional)* — the
    probe's own diagnostic counters for the window: encoder-src entries dropped without a
    pair (depth cap or age bound), and encoder-input buffers whose PTS the compositor ring
    did not hold. **These two deliberately break the absent-means-not-measured
    convention:** they are present whenever the probe is armed, *including as `0`*,
    because a zero is the machine-readable statement "this stage is healthy" and absence
    is the statement "the probe is off". Without them a sparse stage has no reason.

  Authority: none. Observability only — no control-plane behaviour, classifier input,
  scheduling decision, or ABR action may key on them; a control plane that does not
  recognise them ignores them, as with every other additive `session_metrics` field.
  Design + measurement: quasar `docs/superpowers/specs/2026-08-18-latency-probe-design.md`,
  `docs/reports/2026-08-18-latency-probe/REPORT.md` (host share of glass-to-glass = 5.4 ms).
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
  `encoder.drop_detected`, `webrtc.state_changed`, `app.launch_state`, `caps.negotiated`,
  `sdp.answer_applied`, `encoder.stall`, `host.xid`, `host.gpu_fault` — `trace-format.md` §3.2). **`payload`** is the
  per-type object (`trace-format.md` §3). `ts_unix_ms` is the agent wall-clock at the event (same
  convention as `session_metrics.ts_unix_ms`).
- **`webrtc.state_changed` — Connected/Completed both mean established.** ICE may jump
  `Checking → Completed`, skipping `Connected`; a reader treats both as transport-established.
- **`app.launch_state`** — emitted on each launch-state transition, with
  `payload = {"state": "<new state>", "appid": "<provider app id, when known>"}`.
  Event-driven counterpart of `session_metrics.app_launch_state` (same authority
  caveats: observability only).
- The control plane writes a `session_trace_events` row with `source='agent'` (`schema.md`) after
  validating host ownership (`GetSessionHostState`). A malformed or unparsable message is dropped
  without dropping the connection. An event whose `session_id` is not currently `running` on this
  host is **dropped, not stored** — events never resurrect or alter a session (identical posture to
  `session_metrics`).

**Negotiation, stall and host-fault events (approved by Michael 2026-08-23; additive; new
upstream trace event types).**
> *Additive amendment to this message. No existing field changes and no existing event type
> changes shape; the addition is five `event` values. An older control plane stores them like
> any other event, or ignores them — the forward-compatibility rule in `trace-format.md` §3.1
> already covers an unrecognised type.*

Each is emitted on the **reliable lane** (the agent's ordered lifecycle channel, the treatment
`session.effective_media` and `diag.*` get) rather than the bounded, droppable diagnostics lane.
The reason is the same for all five: each one is the *only* statement of a fact, and a dropped
copy leaves a stale or absent record that reads as the opposite of what happened.

| `event` | when | payload |
|---|---|---|
| `caps.negotiated` | every time the encode branch (re)negotiates | `trigger` (`"launch"` \| `"rung_step"` \| `"scale_rebuild"` \| `"encoder_restart"` \| `"source_swap"`), `encoder_factory` (string\|null), `codec` (`"h264"`\|`"h265"`\|`"av1"`\|null), `profile` (string\|null — the LIVE value), `level` (string\|null), `encoder_sink_caps` / `encoder_src_caps` / `payloader_src_caps` (string\|null), `size` (`{w,h,fps}`) |
| `sdp.answer_applied` | after a remote **answer** applied successfully | `pc` (`"video"`\|`"audio"`), `m_lines` (array of `{mid, kind, codec, port, direction, rejected}`), `rejected_count` (int), `ice_ufrag_changed` (bool, optional) |
| `encoder.stall` | encoder output silence ≥ the host's stall threshold while input keeps arriving, and again on recovery | `phase` (`"detected"`\|`"recovered"`), `reason` (`"no_output"`\|`"input_starved"`\|`"negotiation"`), `since_ms` (int, on `detected`), `stalled_ms` (int, on `recovered`) |
| `host.xid` | the NVIDIA kernel driver recorded an Xid fault | `code` (int), `pci` (string\|null), `pid` (int\|null), `name` (string\|null), `text` (string, ≤256 chars), `ts_unix_ms` (int) |
| `host.gpu_fault` | a non-NVIDIA GPU fault the kernel recorded (amdgpu ring timeout / reset) | `vendor` (`"amd"`), `text` (string, ≤256 chars), `ts_unix_ms` (int) |

- **`caps.negotiated` is NOT a replacement for `session.effective_media`.** That message keeps
  its exact shape and its one-shot, first-offer semantics. This event exists because a one-shot
  snapshot stops being true: a scale-stage rebuild renegotiates the branch, and on the Vulkan
  path every resolution rung step is an encoder restart. After the first step the snapshot
  describes caps that no longer exist.
- **`sdp.answer_applied` is the success counterpart of `webrtc.remote_description_failed`.** An
  answer can apply cleanly and still refuse media: `port: 0` or `direction: "inactive"` on an
  m-line the agent offered to send. `rejected: true` marks exactly that, and `rejected_count > 0`
  means that media will never flow on that PeerConnection even though nothing failed.
- **`host.*` events belong to the HOST, not to a session.** The kernel does not know which
  session's work faulted, so the agent emits the same event **once per running session** and not
  at all when no session is running. A reader must treat two sessions reporting the same
  `host.xid` as one fault seen twice, never as two faults. Host-ownership validation is
  unchanged: the control plane still drops an event whose `session_id` is not `running` here.
- **Authority: none.** Like every other `session_trace_event`, these are observability. No
  session state, scheduling decision, ABR action or admission choice may key on them.

**`diag.*` — on-demand capture results (session-capture, approved by Michael 2026-08-23).**
> *Additive amendment to this message. No existing field changes; the addition is a family of
> `event` values and the handling rules that go with them. An older control plane stores a
> `diag.*` row like any other event and simply never prunes differently — the exemption below
> is a control-plane behaviour, not a wire field.*

The single result of a `session_capture` command (§`session_capture`) rides this message, with
`event` one of `diag.pipeline_dot` | `diag.encoder_props` | `diag.burst_stats`, `ts_unix_ms` the
agent wall-clock at **completion**, and this payload:
```json
{
  "capture_id": "<uuid, echoed from the command>",
  "kind": "pipeline_dot",
  "encoding": "gzip+base64",
  "content_type": "text/vnd.graphviz",
  "data": "<base64 of the gzip stream>",
  "bytes": 48213,
  "compressed_bytes": 6104,
  "original_bytes": 48213,
  "truncated": false,
  "duration_ms": 42
}
```
- **`encoding`** is `"gzip+base64"` (the payload is in `data`) or `"json"` (the payload is the
  object in `json`, used for small results). `content_type` names what the decoded bytes are.
- **`bytes`** is the uncompressed size **after** any truncation, `original_bytes` the size
  before it, `compressed_bytes` what is actually on the wire. `truncated` says which of the two
  a reader is holding.
- **`error`** *(string, optional)* — present only when the capture was **accepted** and then
  failed (deadline, element gone). The event is still emitted: a capture that acked `ok` always
  terminates in an event, so a poller never has to distinguish "still running" from "lost".
- **Three handling rules, all control-plane side:**
  1. **Reliable lane, never coalesced.** A `diag.*` event is persisted **synchronously**, the
     same treatment `session.effective_media` already gets — diagnostic-queue saturation must
     not be able to erase the one artifact an admin explicitly asked for.
  2. **Exempt from the rolling prune.** The 1 h per-session window that reaps ordinary trace
     events (`trace-format.md` §6) excludes `type LIKE 'diag.%'`, and so does the terminal
     prune. A capture is deleted with its session's `ON DELETE CASCADE` and by nothing else —
     an admin who captures a pipeline graph and then stops the session must still be able to
     read it.
  3. **Bounded on the wire.** The control plane rejects a `session_trace_event` whose payload
     exceeds **1 MiB** — belt and braces beside the agent's own `budget.max_bytes`, because the
     exemption above means a `diag.*` row lives as long as the session does. The cap is on the
     JSON payload, not on `compressed_bytes`, and the two are deliberately far apart: a capture
     at the 256 KiB byte budget is base64-encoded on the wire (×4/3 ≈ 350 KiB) inside a JSON
     object, so a cap set at the budget itself would reject a perfectly legal maximum capture.
- Host-ownership validation, the drop-never-fatal posture, and the "never session authority"
  rule are exactly as for every other `session_trace_event` — a `diag.*` event from a host that
  does not own the session is dropped, and no capture ever moves a session.

### `image_state` — managed-image pull/remove progress (image-management P2)
> *Additive amendment. A new upstream message; an older control plane ignores the unknown
> `type`. Fire-and-forget (no `ack`), image-presence authority only — it never touches
> sessions.*

Sent on every state transition of a managed image on this host, and throttled during a pull
(at most every 2 s or per ≥5 % progress delta, whichever is coarser — never per docker layer
event).
```json
{
  "type": "image_state",
  "image_id": "steam",
  "version": "2026.08.07",
  "state": "pulling",
  "progress_pct": 42,
  "bytes": 1234567,
  "error": ""
}
```
- **`state` ∈ `{absent, pulling, ready, failed}`** for prebuilt images. (`building` is reserved
  for the template-build amendment — a P2 control plane accepts it in the schema CHECK but no
  P2 agent sends it.)
- **`progress_pct`** (0–100, int) and **`bytes`** (downloaded so far) are best-effort and only
  meaningful while `pulling`; `bytes` on `ready` is the image's on-disk size when cheaply known,
  else omitted/0.
- **`error`** is non-empty only when `state = "failed"` and is a short operator-readable
  message ("insufficient disk: 3.2 GiB free, image needs ~9 GiB", "registry auth denied",
  "manifest not found"), never a raw docker error blob.
- The control plane upserts `host_images(host_id, image_id)` from this message. Unknown
  `image_id` (not in `image_catalog`) is dropped, not stored.

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
    "bitrate_kbps": 15000, "h264_profile": "constrained-baseline", "codec": "h264"
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

> *(multi-codec, additive)* The `stream` block may also carry an optional
> **`codec`: `"h264" | "h265" | "av1"`** (omitted, or empty, defaults to `"h264"`): the single
> video codec the agent encodes and offers as the one video codec in SDP. The control plane resolves
> it server-side at launch (profile codec preference, clamped by the host's reported `codecs`, the
> launching device's decode probe, and decode-failure history; guaranteed H.264 floor) and stores it
> in `sessions.codec` (`schema.md`). `h264_profile` applies to the `h264` codec only and is ignored
> for `h265`/`av1`. An `h265`/`av1` value the host cannot encode is a hard session-launch failure
> reported via `ack{ok:false}` (the agent never silently substitutes a different codec, which would
> desync `sessions.codec` from reality); the control plane will not send such a value because it has
> already clamped against the host's `codecs` set. Omitted (legacy/tier launch, or an older control
> plane) ⇒ H.264, so every pre-multi-codec session and older agent is unaffected. Additive; no
> existing field or shape changes.

> *(microphone capture, additive, 2026-08-02)* The `stream` block may also carry an optional
> **`mic`: bool** (omitted ⇒ `false`). When `true`, the agent adds a `recvonly` Opus
> transceiver (clock-rate 48000) to the audio webrtcbin **before** the first offer, decodes the
> client's microphone track, and plays it into the session's PulseAudio sidecar `quasar_mic`
> sink, whose remapped source (`quasar_mic_src`) the app container records for voice chat (see
> `signaling.md` §Microphone m-line). The control plane sends `true` only when the instance
> setting `mic_capture_enabled` is on **and** the client requested a microphone at launch
> (`control-api.md` `POST /v1/sessions` `mic`). Omitted/`false` (older control plane, feature
> disabled, or not requested) ⇒ today's single-m-line audio offer, byte-identical wire.
> `local_only` topology has no WebRTC pipeline and ignores the flag. The agent reports the
> microphone state in its `session.effective_media` trace payload (`mic`:
> `"off" | "negotiated" | "active"`). Additive; no existing field or shape changes.

> *(first-run-experience §S2, additive, 2026-08-09)* The `app` object may also carry an optional
> **`network`: `"none" | "bridge"`** — the docker network mode for the app container.
> This is a **per-app requirement, not a host setting**: app containers default to `--network
> none` (correct for almost everything), but Steam's first boot must reach the internet to
> download `steamui.so` or it clean-exits and the session dies as "media path interrupted"
> (#463). The control plane resolves it server-side (`apps.runtime_spec.network`, else the app's
> runtime preset's `network` column, `schema.md`) and sends the winner here — **the agent does no
> merging**. Omitted/`null`/empty ⇒ the agent's existing host fallback chain
> (`QUASAR_CONTAINER_NETWORK`, else `none`) — byte-identical behaviour for every older control
> plane and every app that states no preference.
>
> Accepted values: **`none` | `bridge`**. Anything else fails the session via `ack{ok:false}` with
> a message naming the offending value; the agent never passes an unrecognised string to the
> container runtime.
>
> **`host` is not accepted over the wire**, and this is deliberate rather than an oversight.
> `--network host` does not widen the container's reach — it removes the network namespace, so the
> app shares the host's stack: every service on host loopback (the control plane, Postgres, the
> docker proxy, any admin-only port an operator assumed a tenant workload could not reach) becomes
> reachable, and the container can bind host ports itself. This field is portable — the control
> plane resolves it from an app's `runtime_spec` or a runtime preset materialized from a catalog
> image manifest authored on another machine — so honouring `host` here would let a manifest
> dissolve the isolation boundary on every host that installs it.
>
> The agent **does** accept `host` from its own `QUASAR_CONTAINER_NETWORK` environment knob, which
> is set by the operator of that specific machine and travels nowhere. The asymmetry is the point:
> host networking stays a host-administration decision and never becomes an app property. An agent
> that receives `"host"` in `AppSpec.network` must reject the session even when its own knob is set
> to `host`. The control plane constrains the value at earlier layers too (the `runtime_presets.network`
> `CHECK`, the admin API's `400 validation_failed`, and P5 manifest install validation) — this is
> the agent's **independent backstop**, because the wire is not a trusted input regardless of how
> many earlier layers exist. Additive; no existing field or shape changes.

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

### `session_display_update` — live render resolution / UI scale / external resolution (session-display-update; session-display-stream, approved 2026-08-16)
> *Additive amendment. New downstream message; no existing message, field, or ack contract
> changes. An older agent that does not recognise `session_display_update` treats it as an
> unknown `type` (existing discipline) — wire-silent, exactly like `image_build` on a P2 agent.*
>
> **Amendment — session-display-stream (live external/encoded resolution), additive, approved 2026-08-16 (PR #15)
> 2026-08-16, approved 2026-08-16 (PR #15).** Adds `stream_width`/`stream_height` below. **Approved 2026-08-16 (PR #15).**

Tells the agent to change a `running` session's **compositor-side app-facing presentation**
and/or its **encoded/streamed size**: the app-facing `wl_output` **logical mode** (the "render
resolution" / INTERNAL size the composited scene is produced at) and/or the **UI scale** driven
to toplevels via `wp_fractional_scale_v1`'s `preferred_scale` — unchanged from session-display-
update — and, the **EXTERNAL** (encoder) size the composited scene is upscaled into
before it is streamed.
```json
{
  "type": "session_display_update",
  "id": "<cmd-id>",
  "session_id": "<uuid>",
  "render_width": 1280,
  "render_height": 720,
  "ui_scale": 1.5,
  "stream_width": 1920,
  "stream_height": 1080
}
```
- `render_width` / `render_height` *(int, optional, both-or-neither)* — the new app-facing
  logical mode, in pixels. Omitted (both) ⇒ unchanged.
- `ui_scale` *(number, optional)* — the new `preferred_scale` hint. Omitted ⇒ unchanged.
- **`stream_width` / `stream_height`** *(int, optional, both-or-neither)* — the new
  encoder/external size, in pixels. Omitted (both) ⇒ unchanged.
- **The agent enforces, and rejects out-of-range values for:** `16 ≤ render_width ≤` the
  session's **pinned LAUNCH** width, `16 ≤ render_height ≤` the session's pinned LAUNCH height,
  both **even**; `1.0 ≤ ui_scale ≤ 3.0`. (2026-08-16 amendment: render and external/stream size
  are **independent axes** — render is bounded only by the session's pinned launch size, never by
  the current or any past external size. This replaces the earlier "render ≤ current external,
  never raised past it" rule.)
- **The agent additionally enforces, for `stream_width`/`stream_height`:** both **even**,
  `16 ≤` each `≤` the session's **launch** stream width/height, and — **the load-bearing new
  check** — that its own encoder actually **supports a live resize**. An encoder that does not
  (e.g. a hardware encoder whose driver requires a full re-init to change coded size) rejects
  every `stream_width`/`stream_height` request outright, regardless of value, with
  `ack{ok:false, error:"display_update_rejected: encoder does not support live resize"}`. A
  capable encoder that rejects only because of an out-of-range value follows the same
  `display_update_rejected: <reason>` convention as the render-size rejection above. If the new
  external size lands **below** the current render size, the agent does **not** touch the render
  size — the encoder's scale stage simply downsamples the (unchanged, larger) compositor
  framebuffer into the smaller encoded frame; the app never sees a mode change from a
  stream-only update, and stepping the external size back up later is a passthrough.
- **Encode caps, interpipe, and (absent `stream_width`/`stream_height`) stream `WxH` are NOT
  changed by this message.** Only the compositor's advertised `wl_output` mode and
  `preferred_scale` move by default; the pinned encode resolution, bitrate ladder, and codec are
  exactly as `session_assign` (and any subsequent `session_swap_app`) left them **unless**
  `stream_width`/`stream_height` is present, in which case the coded size — and only the
  coded size, never the bitrate ladder or codec — moves at the **next IDR**. No renegotiation:
  the `webrtcbin`/DataChannel/ICE state is untouched (`signaling.md` unchanged); the client
  `<video>` element follows the new coded size automatically.
- **Ack semantics — same contract as `session_swap_app` (differs from assign/start).** The
  agent replies `ack{ id, ok, error? }`:
  - `ack{ok:true}` — the values (whichever were present) were validated and applied.
  - **`ack{ok:false, error}` — the agent rejected the command (e.g.
    `display_update_rejected: render_height 8 below floor 16`, or
    `display_update_rejected: encoder does not support live resize`). The session is left running
    with its *previous* render size / UI scale / external size and stays `running` — the update
    is a no-op.** A rejected `session_display_update` **never fails the session and never changes
    session state**, exactly like a rejected swap.
- **Ephemeral, agent-held state.** No value here is stored in the `sessions` table; render size
  and UI scale are never exposed on the `Session` resource, while the external size is
  additionally mirrored into an **in-memory, best-effort** control-plane cache
  (`Session.stream.external_width`/`external_height`, `control-api.md`) purely for `GET`
  convenience — the agent remains the sole authoritative holder for all three. The **only**
  authoritative readback for any of them is `session_metrics` (§`session_metrics`, above), never
  `session_state` and never the `Session` resource being treated as a source of truth.
- **Any host may receive this message** — an older agent that does not understand
  `session_display_update` silently ignores it (unknown `type` ⇒ unknown enum variant on
  deserialize, the same wire-silent behavior documented for `image_build`/`config_update`). The
  control plane treats the absence of an `ack` within its normal command timeout as a rejection.
  An agent that recognises `session_display_update` but predates session-display-
  stream ignores unknown `stream_width`/`stream_height` fields on deserialize (additive-field
  discipline) and behaves exactly as a session-display-update-only agent — i.e. it never resizes
  the stream and never sets `external_resize_supported` on `session_metrics`, which a caller
  correctly reads as *unknown*.

### `session_capture` — a bounded, on-demand observation of a live session (session-capture, approved by Michael 2026-08-23)
> *Additive, admin-only, observability-only amendment. New downstream message; no existing
> message, field, or ack contract changes. An older agent that does not recognise
> `session_capture` treats it as an unknown `type` — the existing wire-silent discipline
> (`ControlMsg::Unknown`), exactly like `image_build` on a P2 agent. It carries **no session
> authority**: a capture never changes session state, never touches the media path, and a
> refused or failed capture is a no-op on a still-`running` session.*

Tells the agent to **observe** a `running` session once — arm, gather within a byte **and**
time budget, report as a single upstream `session_trace_event` — and stop. A capture is not a
probe: nothing is inserted into the media path, no pad probe is added, no element is
reconfigured. It reads what the pipeline already is.
```json
{
  "type": "session_capture",
  "id": "<cmd-id>",
  "session_id": "<uuid>",
  "capture_id": "<uuid minted by the control plane>",
  "kind": "pipeline_dot",
  "budget": { "max_bytes": 262144, "max_ms": 10000 },
  "params": { "windows": 20, "window_ms": 250 }
}
```
- **`capture_id`** *(uuid, required)* — minted by the **control plane**, before dispatch, and
  echoed in the result payload. It is the join key between the `202` an admin already holds
  and the `diag.*` event that arrives later; the agent never invents one.
- **`kind`** *(string, required)* — one of `pipeline_dot` | `encoder_props` | `burst_stats`
  (below). **The control plane owns this vocabulary and grows it**; an agent that does not
  know a string nacks `unknown_kind` rather than guessing.
- **`budget`** *(object, required)* — `max_bytes` (the **compressed** payload cap, default and
  ceiling 262144) and `max_ms` (wall-clock arm→done, default 10000, ceiling 30000). The agent
  clamps both to its own ceilings; it never honours a larger number than it can afford.
- **`params`** *(object, optional)* — `burst_stats` only: `windows` (clamped to `1..40`) ×
  `window_ms` (clamped to `100..1000`), and the product additionally clamped to
  `budget.max_ms`. Ignored for the other kinds.

**Ack semantics — same contract as `session_swap_app` / `session_display_update`.** The agent
replies `ack{ id, ok, error? }`, and **a rejected capture never fails the session**:
- `ack{ok:true}` — accepted and armed. The result arrives later as the `diag.*` event; the ack
  is not the result.
- `ack{ok:false, error:"busy"}` — a capture is already in flight for that session.
  **Single-flight per session: refuse, never queue.** One observation at a time is the whole
  cost bound.
- `ack{ok:false, error:"unsupported"}` — this agent build knows the kind but cannot do it
  *here, now* (e.g. the session has no encode pipeline yet).
- `ack{ok:false, error:"unknown_kind"}` — the string is not in this agent's vocabulary.
- `ack{ok:false, error:"no_such_session"}` — the agent does not hold that session.
- **No ack at all ⇒ the agent predates `session_capture`.** An unknown `type` is wire-silent
  (it does not produce an `error` reply), so the control plane's ack timeout is the only
  signal available and it maps that to `501 capture_unsupported` — "rebuild this host's agent",
  not "retry". This is the one deliberate difference from `session_display_update`, where a
  timeout reads as a rejection: a capture mutates nothing, so attributing a silent host to an
  old build costs nothing and names the actual remedy.

**The safety envelope — what a capture may never contain.** These are properties of the
capture, not of any one kind, and they hold for every kind added later:
- **No media.** No pixels, no audio samples, no encoded bitstream, no frame contents.
- **No input.** No key/pointer/gamepad events, no microphone.
- **No credentials or identity.** Never `node_secret`, an enrollment token, a bearer, or the
  environment wholesale. `encoder_props` reads an **allow-list** of property names — never
  "every property on the element" — and elides any string value longer than 256 characters.
- **No filesystem reach.** No path outside the session's own scratch directory is read or
  named.
- **Bounded by construction.** `compressed_bytes ≤ budget.max_bytes`. Over the cap, the agent
  truncates the **uncompressed** text at a line boundary, recompresses, and sets
  `truncated: true` with `original_bytes` — a capture is never allowed to grow past its budget
  by being interesting.

**Kinds.**
- **`pipeline_dot`** — the encode pipeline's graph, `gst::debug_bin_to_dot_data` with
  `CAPS_DETAILS | STATES`. **Deliberately not `NON_DEFAULT_PARAMS`**: element parameters can
  be string properties carrying paths, which the envelope above forbids, and the allow-list
  that makes `encoder_props` safe does not exist for a dot dump.
  `content_type: "text/vnd.graphviz"`, `encoding: "gzip+base64"`.
- **`encoder_props`** — the live encoder snapshot, `encoding: "json"`:
  `{ "encoder_factory", "codec", "properties": { <allow-listed name>: <value> },
  "caps": { "encoder_sink", "encoder_src", "payloader_src" }, "scale_stage"?: {…},
  "ring"?: {…} }`. The same fields the `session.effective_media` snapshot carries, read
  **now** rather than at launch — which is the point: a knob that moved after launch is
  exactly what this answers.
- **`burst_stats`** — a short, dense window series, `encoding: "json"`:
  `{ "window_ms", "windows": [ { "t_ms", "encode_ms": {"n","p50","p95","max","samples":[…≤200]},
  "dwell_ms": {…}, "fps", "bitrate_kbps", "frames_dropped", "queue_level_max",
  "abr_setpoint_kbps", "gcc_estimate_kbps" } … ] }`. Denser than the `session_metrics`
  heartbeat cadence for a bounded stretch, and only when asked — the reason it is a capture
  and not a new always-on metric.

**Result.** Exactly one upstream `session_trace_event` with `event: "diag.<kind>"` (below).
A capture that is accepted and then fails still reports — the same event with `error` set —
so an admin polling on `capture_id` always terminates on something other than a timeout.

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
    "output_id": "card0:DP-4",
    "mode": {"width": 3840, "height": 2160, "refresh_millihz": 119879},
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
  `output_id` and `mode` are configured together and must exactly match the latest typed DRM
  inventory. The agent writes a session-owned Weston config selecting that connector and timing.
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
- **The `settings` block is an OPAQUE passthrough of the control plane's knob
  catalog.** This document deliberately does not enumerate the keys — the catalog is the
  contract (`GET /v1/admin/config/catalog`, `control-api.md`), and knobs are added there
  without a protocol amendment. The 2026-08-16 adaptation wave adds `abr_mode` (enum
  `off|protective|smooth`, which **supersedes** the lossy `abr_enabled` bool — `false` ⇒
  `off`, `true` ⇒ defer to `abr_mode`) plus the `abr_ladder*` family; all are live-class
  and all reach the agent through this same block with no message-shape change.
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

### `image_ensure` — pull a prebuilt catalog image onto this host (image-management P2)
> *Additive amendment. A new downstream command; an older agent silently ignores the unknown
> `type` (existing discipline).*

Reserve/prepare semantics like `session_assign`: the agent acks acceptance immediately, then
does the pull asynchronously and reports progress via `image_state`.
```json
{
  "type": "image_ensure",
  "id": "<command-id>",
  "image_id": "steam",
  "registry_ref": "ghcr.io/accreleus/quasar-steam:sha-969cc14ea168",
  "version": "2026.08.07"
}
```
- Agent replies `{type:"ack", id, ok:true}` (accepted — not "done") and starts/continues the
  pull. An ensure for an `(image_id, version)` already in flight or already `ready` is
  **idempotent**: ack, no second pull, current state re-emitted via `image_state`.
- `ack{ok:false, error}` only when the command is un-actionable on its face (malformed ref,
  managed-image support disabled on this agent). Runtime failures (disk, registry, network)
  are reported as `image_state{state:"failed", error}` after an `ok:true` ack.
- **`registry_ref` is a concrete immutable ref** (a `sha-` tag or digest), never a floating
  tag — "ensure" is deterministic and re-runnable.
- The agent checks free disk on the docker data filesystem **before** pulling and fails fast
  with a readable `image_state` error when headroom is insufficient.
- The agent records `image_id → registry_ref` so `image_remove` and the `register` `images`
  report can speak in `image_id` terms.

### `image_remove` — best-effort removal of a managed image (image-management P2)
```json
{ "type": "image_remove", "id": "<command-id>", "image_id": "steam" }
```
`ack{ok:true}` then a best-effort `docker rmi` of the ref the agent recorded for that
`image_id`; terminal `image_state` follows (`absent` on success, `failed` with a readable
`error` if the daemon refused — e.g. the image backs a container). The agent **never**
force-removes an image backing a live container. An `image_id` the agent has no record of
acks `ok:true` and reports `absent` (already gone — idempotent).

### `image_build` — build a template catalog image on this host (image-management P4)
> *Additive amendment. A new downstream command; an older agent silently ignores the unknown
> `type` (existing discipline) and so never becomes ready for a template image.*

The template analogue of `image_ensure`. Reserve/prepare semantics are identical: the agent
acks acceptance immediately, then fetches the build context and runs `docker build`
asynchronously, reporting progress via `image_state` — reusing the `building` state P2 already
reserved.
```json
{
  "type": "image_build",
  "id": "<command-id>",
  "image_id": "xfce-desktop",
  "context_url": "https://codeload.github.com/accreleus/quasar-images/tar.gz/<commit-sha>",
  "context_subdir": "xfce-desktop",
  "dockerfile": "Dockerfile",
  "build_args": { "BASE": "..." },
  "local_tag": "quasar-local/xfce-desktop:2026.08.08",
  "version": "2026.08.08"
}
```
- Agent replies `{type:"ack", id, ok:true}` (accepted — not "done") and starts/continues the
  build. A build for an `(image_id, version)` already in flight or already `ready` is
  **idempotent**: ack, no second build, current state re-emitted via `image_state`.
- `ack{ok:false, error}` only when the command is un-actionable on its face (malformed
  `context_url`, disallowed host, managed-image support disabled on this agent). Runtime
  failures (download, extraction, disk, build) are reported as `image_state{state:"failed",
  error}` after an `ok:true` ack.
- Progress rides `image_state{state:"building", progress_pct, ...}`; terminal `ready` (built and
  tagged `local_tag`, `bytes` = image size when cheaply known) or `failed` (short
  operator-readable error, never a raw build-log blob). `progress_pct` for a build is
  best-effort — docker build has only coarse step output, so the agent parses `Step N/M` when
  present and omits `progress_pct` otherwise.
- **`context_url`** is a tarball of the source repo pinned to a **commit sha**, never a floating
  ref — the control plane resolves the catalog ref to a sha at sync, so a template build is as
  deterministic as a digest-pinned pull (the sha is stored in `image_catalog.context_sha`,
  `schema.md`). It is a GitHub codeload tarball for the configured repo; the agent downloads it,
  extracts `context_subdir` as the build context, and builds `dockerfile` within it.
- **`local_tag`** is CP-assigned and deterministic: `quasar-local/<image_id>:<version>`. It is
  **never a registry ref** — a template image is built locally and **never pushed** anywhere. The
  agent records `image_id → local_tag` in the same state map it uses for `image_id → registry_ref`
  (P2), so `image_remove` and the `register` `images` report speak in `image_id` terms exactly as
  for a pulled image. `image_remove` for a template `docker rmi`s the recorded `local_tag`, same
  path as a pulled ref.
- **SSRF containment.** Before downloading, the agent validates the `context_url` host against an
  allowlist (`QUASAR_IMAGE_SOURCE_HOSTS`, default `codeload.github.com,github.com`), requires
  HTTPS, enforces a download **size cap**, and guards tar extraction against path traversal
  (zip-slip) so an entry cannot escape the scratch context dir. A build with insufficient free
  disk fails fast with a readable `image_state` error (a build needs more headroom than a pull),
  and the scratch dir is always cleaned.

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
