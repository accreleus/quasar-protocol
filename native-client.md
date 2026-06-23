# Native-client capability / metrics / certification report (AS10-12, FROZEN)

The wire contract a **future native client** uses to report its capabilities, its
observed playback metrics, and its per-profile certification results to the control
plane. It is the native-side counterpart of the web client's login-time probe
(`web/src/webrtc/capability.ts` `probeCapabilities()`), and it is deliberately an
**additive superset of that web probe** — *not* a new endpoint, table, or column.

**Frozen** (see `CLAUDE.md`): like `agent-api.md` and `signaling.md`, this document is
a load-bearing wire contract. Cheap-model tickets on the native-client and
control-plane sides will depend on it being stable, so **changes need Opus + explicit
human sign-off**. The one exception — additive, append-only growth — is described under
*Versioning* below and still wants sign-off.

> **Status: SPIKE (AS10-12, #208).** This is the *contract*; no native client implements
> it yet (the native-client spike `#186`/`#191` validated the transport — see
> `spike/native-client/RESULTS.md` and `docs/phase9/native-client-architecture.md`). The
> control-plane ingest is in place additively (it parses the known fields, stores the
> full blob opaquely, and is byte-identical for an existing web blob); a *consumer* of
> the native-specific fields is a later phase, exactly as `user_devices` has no live
> consumer of the web probe today.

## Why this is the web probe, extended (and not a new thing)

Invariant #1 (control plane / node-agent split) and the existing telemetry design already
give us the right home for this:

- **Same endpoint.** The report is posted to **`POST /v1/me/devices`** — the exact
  endpoint the web SPA already uses (`control-api.md` §Telemetry & devices). Owner is the
  bearer identity; `device_key` is the stable per-device id.
- **Same column.** It is stored in the **same `user_devices.capabilities` JSONB column**,
  which `schema.md` documents as *deliberately schema-free* so the probe can grow. There is
  **no DB migration** — adding a column would be a misstep (it forfeits the documented
  growth path and trips the migration-rollback hazard in `CLAUDE.md`).
- **Same sanitizer.** The control-plane sanitizer (`control-plane/internal/devices/
  sanitize.go`) already (a) whitelists `client_type: "native"` and (b) passes unmodelled
  fields through verbatim within generic depth/width/string bounds. A native report flows
  through it unchanged in structure; AS10-12 only additively validates the new
  `report_version` field and confirms the new sub-objects fit the existing bounds.

### The "web subset is a valid report" invariant

A native report is the web `DeviceCapabilities` shape **plus** optional native-only fields.
Every field the web client sends keeps the same name, type, and meaning. Therefore:

1. **An existing web blob is, unchanged, a valid (minimal) native-family report.** It stores
   byte-identically after the AS10-12 changes — this is the project's #1 additive-safety
   gate (`TestNativeWebSubsetUnaffected`).
2. **A native client may omit any native-only field.** Absent ≠ false; a missing field means
   "not reported", never "unsupported" (mirrors the web probe's `null`-is-valid rule and the
   eligibility `Probe` "zero field → unknown → allow" rule).
3. **The flat `codecs` bool map stays the eligibility surface.** `codecs: {h264,hevc,av1,vp9}`
   is *both* the web subset *and* the input the eligibility evaluator reads
   (`profile.Probe.Codecs`). The richer per-codec `decode{}` object (below) is
   **forward-data only** — it is NOT a new eligibility hard gate. (HEVC/AV1 are placeholders:
   `profile.go` marks them `CodecFuture`; `LaunchableCodec` only returns H.264.)

## Transport model this report targets

This report describes a client that connects through the **existing WebRTC signaling +
media model**, exactly as the browser does and as the native-client spike validated:

- control plane for auth + launch (`control-api.md`),
- `signaling.md` as the WebRTC answerer (host is offerer; client answers; input on the
  DataChannel),
- `input.md` for the input wire format.

The spike (#191) settled the transport question: a **tuned `webrtcbin` + VideoToolbox**
native client is the answer; **str0m was ruled out** (won the median, lost the p95 tail).
A frame-boundary receive path (WebTransport / WebCodecs / a custom UDP transport) is
**discussed-not-required future work** — when one lands it is a new `render_path` value and,
if it needs its own negotiation, a *new* additive message, never a change to this report's
existing shape. See `docs/phase9/native-client-architecture.md` for the roadmap.

## The report schema

Posted as the `capabilities` object of `POST /v1/me/devices`:

```json
{ "device_key": "<stable per-device id>",
  "capabilities": {

    "report_version": 1,
    "client_type": "native",
    "client_name": "quasar-native-macos",
    "client_version": "0.1.0",

    "platform": "macOS",
    "os": { "name": "macOS", "version": "15.5", "arch": "arm64" },

    "display": {
      "width": 3456, "height": 2234, "device_pixel_ratio": 2.0,
      "refresh_hz": 120, "hdr": true, "vrr": true
    },

    "codecs": { "h264": true, "hevc": true, "av1": true, "vp9": false },
    "max_decode_height": 2160,
    "decode": {
      "h264": { "hw": true, "profiles": ["constrained-baseline","main","high"],
                "levels": ["4.2","5.1"], "max_height": 2160 },
      "hevc": { "hw": true, "profiles": ["main","main10"],
                "levels": ["5.1"], "max_height": 2160 },
      "av1":  { "hw": true, "profiles": ["main"],
                "levels": ["5.1"], "max_height": 2160 }
    },

    "audio": { "channels": 2, "sample_rate": 48000, "codecs": ["opus"] },

    "input": {
      "raw_mouse": true,
      "keyboard": true,
      "high_rate_input": true,
      "controllers": [
        { "type": "xbox", "rumble": true, "haptics": false, "player": 0 },
        { "type": "dualsense", "rumble": true, "haptics": true, "player": 1 }
      ]
    },

    "metrics": {
      "decode_ms": 1.8,
      "present_fps": 59.9,
      "present_interval_sd_ms": 1.2,
      "glass_to_glass_ms_p50": 45.0,
      "glass_to_glass_ms_p95": 104.0,
      "interactive_ms_p50": 54.0,
      "jitter_buffer_ms": 20.0,
      "render_path": "webrtcbin+videotoolbox"
    },

    "health": { "class": "smooth", "reason": "" },

    "profiles": {
      "1080p60": {
        "h264_profile_decoded": "high",
        "decode_pass": true, "present_pass": true,
        "decode_ms": 1.6, "present_fps": 59.8, "dropped_ratio": 0.0,
        "measured_at": "<RFC3339>"
      }
    },

    "bandwidth_kbps": 92000,
    "rtt_ms": 6
  } }
```

### Identity

| field | type | notes |
|---|---|---|
| `report_version` | integer | **append-only** schema version of this report. `1` for AS10-12. A non-integer value is dropped by the sanitizer (the rest of the blob still stores). Absent ⇒ treat as the web-probe baseline. |
| `client_type` | string | `"native"` for native clients (`"web"` for the SPA). Validated against the known set; anything else normalises to `"web"` server-side. |
| `client_name` | string | implementation id, e.g. `"quasar-native-macos"`. Best-effort. |
| `client_version` | string | the native client's own version. Best-effort. |

### Platform / OS

| field | type | notes |
|---|---|---|
| `platform` | string | coarse platform label; same field the web probe sends (kept for the subset invariant). |
| `os` | object | `{ name, version, arch }` — all best-effort strings (`arch` e.g. `"arm64"`, `"x86_64"`). |

### Display

`display` extends the web `DisplayInfo` (`width`, `height`, `device_pixel_ratio`,
`refresh_hz`) with two native-detectable booleans:

| field | type | notes |
|---|---|---|
| `width`, `height` | number | display geometry in physical pixels. |
| `device_pixel_ratio` | number | scale factor. |
| `refresh_hz` | number | refresh rate; omitted when not measurable (web rule). |
| `hdr` | bool | HDR-capable display. Forward-data; no consumer yet. |
| `vrr` | bool | variable-refresh-rate (e.g. ProMotion / FreeSync / G-Sync). Forward-data. |

### Decode capability

Two coexisting representations, by design:

- **`codecs`** — the flat `{ h264, hevc, av1, vp9 }` bool map. This is the **subset shared
  with the web probe** and the **eligibility surface** (`profile.Probe.Codecs`). KEEP it.
  This map is a **closed set by design**: the four keys (`h264`/`hevc`/`av1`/`vp9`) are the
  complete set reflected in the TypeScript `CodecCapabilities` interface; future codecs require
  a contract amendment, not an open-map addition.
- **`decode`** — a richer per-codec object, **forward-data only** (NOT an eligibility gate):

  ```
  decode.<codec> = { hw: bool, profiles: [string], levels: [string], max_height: int }
  ```

  - `hw` — hardware decode available (e.g. VideoToolbox). The web probe cannot tell HW from
    SW; a native client can.
  - `profiles` / `levels` — decodable H.264/HEVC/AV1 profiles + levels (≤ 64 entries each,
    within the sanitizer's `maxArrayLen`).
  - `max_height` — per-codec max decode height.

  HEVC/AV1 entries are **placeholders**: present so the shape is stable, but not consumed —
  the launchable codec is H.264-only today (`profile.go` `CodecFuture`).

  **VP9 is deliberately absent from `decode{}`**: it appears only in the flat `codecs` bool map
  (the web-probe subset) and is not a decode-matrix target. VP9 hardware encode is a dead-end on
  current AMD VCN and NVIDIA NVENC; AV1 is the planned future codec rung.

`max_decode_height` (flat) is kept from the web probe (the eligibility decode-height check).

### Audio

| field | type | notes |
|---|---|---|
| `audio.channels` | number | output channels (e.g. `2`). |
| `audio.sample_rate` | number | Hz (e.g. `48000`). |
| `audio.codecs` | [string] | decodable audio codecs, e.g. `["opus"]` (≤ 64). |

### Input

| field | type | notes |
|---|---|---|
| `input.raw_mouse` | bool | raw/relative mouse motion available (no OS acceleration). |
| `input.keyboard` | bool | full keyboard capture available. |
| `input.high_rate_input` | bool | high-polling-rate input path (the native equivalent of the web `coalesced_pointer_events`). |
| `input.controllers` | [object] | connected controllers, **≤ 16** in practice (well within `maxArrayLen=64`). Each: `{ type, rumble, haptics, player }` — `type` e.g. `"xbox"`/`"dualsense"`; `rumble`/`haptics` bool; `player` int index. |

### Metrics

Self-measured playback metrics, reusing the `web/src/api/types.ts` `BrowserMetrics`
vocabulary so the two clients speak one metrics language:

| field | type | notes |
|---|---|---|
| `decode_ms` | number | mean per-frame decode time. |
| `present_fps` | number | presentation frame rate (display-side). |
| `present_interval_sd_ms` | number | present-interval standard deviation (smoothness / judder; the #108 pacing metric). |
| `glass_to_glass_ms_p50` | number | glass-to-glass latency, p50. |
| `glass_to_glass_ms_p95` | number | glass-to-glass latency, p95. **Both p50 and p95 matter** — the native spike's residual was a p95 tail, not the median (`docs/phase9/native-client-architecture.md`). |
| `interactive_ms_p50` | number | input-to-photon (interactive) latency, p50. |
| `jitter_buffer_ms` | number | the receiver jitter-buffer depth the client is holding. |
| `render_path` | string | the receive/decode/present path, e.g. `"webrtcbin+videotoolbox"`. A future `"webtransport+webcodecs"` is just a new value here. |

These are **forward-data**, like the web probe's stored metrics — no optimizer consumes them
in AS10-12.

### Health

`health` reuses the AS10-11 `ClientHealth` vocabulary (`web/src/api/types.ts`):

| field | type | notes |
|---|---|---|
| `health.class` | string | one of `smooth` / `decode_degrading` / `presentation_degrading` / `backgrounded_or_hidden` / `client_unsupported`. Client-classified; the server stores it and never re-derives it. |
| `health.reason` | string | optional human explanation. |

### Profiles (certification map)

`profiles` is the **same per-profile certification map** as the web probe
(`web/src/webrtc/capability.ts` `ProfileCertification`), keyed by stream-profile id
(`GET /v1/me/profiles`). Each entry: `h264_profile_decoded`, `decode_pass`, `present_pass`,
`decode_ms`, `present_fps`, `dropped_ratio`, `measured_at`. A native client can certify the
**higher** H.264 profiles (`main`/`high`) the browser cannot decode — which is precisely why
per-session profile negotiation (`P1-11`) and this contract exist.

### Network

| field | type | notes |
|---|---|---|
| `bandwidth_kbps` | number\|null | rough downstream bandwidth; `null`/absent ⇒ unmeasured. Same field + meaning as the web probe (feeds eligibility `Probe.BandwidthKbps`). |
| `rtt_ms` | number\|null | round-trip time; `null`/absent ⇒ unmeasured (feeds `Probe.RTTMs`). |

`measured_at` (top-level) is **server-stamped at upsert** — any client value is overwritten
(`control-api.md`).

## Server handling (additive, AS10-12)

- The server **parses the modelled fields** it cares about (today: `client_type`,
  `report_version`) and **stores the full JSON opaquely**; unknown fields round-trip verbatim
  within the sanitizer's depth(8) / width(64) / string(512) bounds.
- **`report_version`** is additively validated: a non-integer value is dropped (the rest of
  the blob still stores). It is append-only; never repurpose a version number.
- The new sub-objects (`os`, `decode`, `audio`, `input`, `metrics`, `health`) are passed
  through within the existing generic bounds — `levels` / `controllers` / `profiles` are all
  well under `maxArrayLen` / `maxObjectKeys = 64`.
- **Body size cap.** `POST /v1/me/devices` caps the body at **8 KB**
  (`handler.go` `maxDeviceBodyBytes`). A measured worst-case native payload — full per-codec
  `decode` matrix (3 codecs × 6 profiles × 20 levels), 16 controllers, an 8-entry profile
  certification map, and every metric/health/os/audio field populated — is **≈ 4.2 KB**
  (the full POST body incl. `device_key`), i.e. **~51 % of the cap, leaving ~3.9 KB of
  headroom**. Per-section breakdown (to allow auditing the total):

  | Section | Approx. bytes |
  |---|---|
  | `decode` matrix (3 codecs × 6 profiles × 20 levels) | ≈ 600 B |
  | `input.controllers` (16 entries) | ≈ 960 B |
  | `profiles` cert map (8 entries incl. metrics per rung) | ≈ 1 200 B |
  | `metrics`, `os`, `audio`, `health`, `display`, `identity` fields | ≈ 700 B |
  | `codecs`, `features`, `device_key`, envelope keys/overhead | ≈ 740 B |
  | **Total** | **≈ 4 200 B** |

  The cap is therefore **unchanged**. (If a future `report_version` adds enough
  to overflow it, the cap is raised additively and noted here — it never changes the
  request/response shape.)

No endpoint, request/response shape, status code, or migration changes for this contract.

## Reference type definitions

These mirror the schema for implementers; they are documentation/forward-data, not a new
storage path:

- **TypeScript** — `NativeCapabilities extends DeviceCapabilities`
  (`web/src/webrtc/capability.ts`, re-exported from `web/src/api/types.ts`).
- **Go** — `devices.NativeCapabilities` (`control-plane/internal/devices/native.go`): an
  **optional typed view** for tests / future mapping only. **Storage stays
  `json.RawMessage` opaque** — it is never routed through this struct.
- **Rust** — a documented **placeholder** serde struct mirroring the schema, in
  `spike/native-client/native_capabilities.rs`. It is **not wired to any client**; it exists
  so a future native client has a starting point.

## Versioning

`report_version` is an **append-only integer**. Rules:

1. **Never remove or repurpose a field** — only add. An older server ignores fields it does
   not model (they round-trip opaquely); a newer server treats an absent field as "not
   reported".
2. **Bump `report_version`** when you add fields, so a consumer can branch on it. Absent ⇒
   the web-probe baseline (version 0, implicitly).
3. The flat `codecs` map and `max_decode_height` are the **stable eligibility surface** and
   must remain present and unchanged in meaning across versions.

Any change beyond append-only growth (renaming a field, changing a type, adding a new
eligibility hard gate, or a new transport-negotiation message) requires **Opus + explicit
human sign-off**, per the frozen-contract rule.
