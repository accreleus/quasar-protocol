# Input protocol (Phase 0)

Input events travel **client -> host** over the WebRTC **DataChannel** named `input`. The same channel carries clock-sync pings used for latency measurement (ticket T7).

Encoding: JSON, one object per channel message, discriminated by `t`. (A compact binary encoding is a later optimization — keep JSON for Phase 0 debuggability.)

DataChannel transport: the agent-offered channel is **reliable + ordered** by default (`ordered:true`, the SCTP default retransmission). The original Phase-0 **unreliable + unordered** configuration (`ordered:false, maxRetransmits:0`) is retained as an A/B fallback via `QUASAR_INPUT_CHANNEL_MODE=legacy` on the agent. Both configurations coexist; the wire shapes are identical. Clock-sync pings tolerate loss under either mode.

> **Amendment (input-jitter investigation, 2026-06):** the reliable+ordered default was validated against the Moonlight common-c input path, which also sends relative mouse packets reliably (`ENET_PACKET_FLAG_RELIABLE`). A late or dropped relative-motion sample is more costly to perceived smoothness than the small head-of-line delay of reliable delivery, because the agent now coalesces bursty arrivals into one uinput write per ~1 ms (see Host-side batching below), so a dropped sample leaves a gap the batching cannot reconstruct.

## Coordinate + code conventions
- Relative motion (`dx`,`dy`) is in device pixels at the streamed resolution.
- Absolute motion (`x`,`y`) is normalized `0.0..1.0` across the streamed surface.
- Scroll (`dx`,`dy`) is in high-resolution wheel units.
- Mouse buttons use Linux input codes: left `0x110` (BTN_LEFT), right `0x111`, middle `0x112`.
- Keys use **Linux evdev keycodes**. The client maps `KeyboardEvent.code` -> evdev keycode (a static lookup table; lives in the client). The host passes the evdev code straight to the compositor.
- Gamepad: button/axis indexing follows the **W3C Standard Gamepad** layout from the browser Gamepad API. The host maps this to the virtual pad it presents (Phase 0 may map to compositor input directly; Phase 1 routes through inputtino so a containerized game sees a real evdev device).
- Unknown fields on any message MUST be ignored by the host. The host deserializes with `serde_json` which ignores unknown keys by default; clients MUST tolerate forward-compatible additive fields. This amendment relies on that rule (see `mm` below).

## Messages (client -> host)
```json
{ "t": "mm", "dx": 4,   "dy": -2 }                      // mouse move, relative
{ "t": "ma", "x": 0.51, "y": 0.42 }                     // mouse move, absolute (normalized)
{ "t": "mb", "button": 272, "pressed": true }           // mouse button (272 = 0x110 left)
{ "t": "ms", "dx": 0, "dy": 120 }                       // scroll
{ "t": "k",  "code": 30, "pressed": true }              // key (evdev code; 30 = KEY_A)
{ "t": "gp", "i": 0, "buttons": [0,1,0,...], "axes": [0.0,-0.3,...] } // gamepad state on change
```

### `mm` optional instrumentation fields (additive amendment, 2026-06)
A client MAY attach two optional fields to a relative-mouse-move message for input-jitter correlation:
```json
{ "t": "mm", "dx": 4, "dy": -2, "seq": 17, "tc": 123456.7 }
```
- `seq` (number, optional): per-session monotonic message counter, wraps allowed. Used to detect gaps/reordering on the agent side.
- `tc` (number, optional): client monotonic timestamp in milliseconds, same clock as the `ping`/`pong` `tc` field (e.g. `performance.now()` in the browser). Used to correlate send/receive timing.

Both fields are **optional**. A sender MAY omit them entirely; the host MUST treat absence as "not instrumented" and continue normally. A sender MUST NOT make correctness depend on the host reading them — they are observational only. The host ignores these fields on the live input-injection path; they are consumed only when input-trace logging is enabled (`QUASAR_INPUT_TRACE=1` on the agent). The wire shape of `mm` is otherwise unchanged: `dx` and `dy` are required and keep their Phase-0 meaning.

## Messages (host -> client) — gamepad force-feedback (PROPOSAL P-GP-FF, additive)

> **Status: PROPOSAL — not yet implemented on either side.** Additive, optional,
> backwards-compatible: a client that ignores these degrades to no force-feedback; a host
> that never sends them changes nothing. Carried on the same `input` DataChannel (video PC),
> reverse direction. `i` matches the `player` index in the capability report's
> `controllers[]` (`native-client.md`); a host MUST only send what that controller's
> advertised `rumble`/`haptics` capabilities accept.

```json
{ "t": "rumble", "i": 0, "low": 0.6, "high": 0.3, "duration_ms": 200 }
{ "t": "haptic", "i": 1, "left_trigger": { "mode": "resist", "start": 0.2, "strength": 0.8 },
                  "right_trigger": { "mode": "off" } }
```
- `rumble`: dual-rumble; `low`/`high` are `0.0..1.0` motor intensities (low-frequency /
  high-frequency), `duration_ms` bounded (host clamp 5000). A new `rumble` for the same `i`
  replaces the previous one; `low=high=0` stops immediately.
- `haptic`: DualSense adaptive-trigger effects; trigger objects are forward-extensible
  (`mode` discriminated: `off | resist | weapon | vibrate`, parameters per mode). Clients
  without trigger haptics ignore the message (or actuate rumble-equivalent if they choose).
- Unknown-field / unknown-`t` tolerance applies in both directions (see conventions above) —
  this proposal relies on the existing rule for the client side too: clients MUST ignore
  unknown `t` values received on the input channel.

## Clock sync + latency (T7)
Ping/pong over the same channel establishes the client<->host clock offset so glass-to-glass and interactive latency can be estimated. `tc` = client monotonic ms; `ts` = host monotonic ms.
```json
// client -> host
{ "t": "ping", "id": 17, "tc": 123456.7 }
// host -> client (host may also send pings; symmetric)
{ "t": "pong", "id": 17, "tc": 123456.7, "ts": 998877.1 }
```
Offset ~= `ts - (tc + RTT/2)`, RTT from matched ping/pong. Combine with a frame-index overlay on the video (rendered host-side, read client-side on decode) to attribute latency to encode / network+pacing / jitter buffer / decode+display.

## Host-side mapping (reference)
The host translates each message to a compositor input call (`gst-wayland-display`):
- `mm` -> pointer_motion(dx,dy);  `ma` -> pointer_motion_absolute(x*w, y*h)
- `mb` -> pointer_button(button, pressed);  `ms` -> pointer_axis(dx,dy)
- `k`  -> keyboard_input(code, pressed)
- `gp` -> diff against last state; emit button/axis changes

## Host-side relative-mouse batching (Moonlight-derived, 2026-06)
The agent coalesces bursty relative-mouse arrivals into a single uinput write per ~1 ms window, decoupled from both the browser render clock and per-arrival injection. This smooths the bursty per-message uinput cadence that was the dominant cause of perceived mouse steppiness (the streamed video appeared smooth while input felt jittery).

- A dedicated flush thread (`quasar-rel-flush`) accumulates integer relative deltas into a pending buffer and writes one `REL_X`/`REL_Y`/`SYN_REPORT` frame per window.
- Fractional motion is accumulated across `mm` messages and only the integer part is emitted (`trunc()` toward zero), carrying the fractional remainder forward — so 0.4+0.4+0.4 → 1, not 0+0+0 (per-message `round()` quantizes inconsistently and was a secondary steppiness source).
- Non-`mm` events (button/key/scroll/abs) drain any pending motion synchronously before being injected, so a click always lands at the cursor position the user saw.
- `QUASAR_INPUT_BATCH_MS` (default `1`, `0` = disabled → per-arrival writes with fractional accumulation still active) configures the window.
- The batching is purely host-side: the wire shape is unchanged, no new messages are introduced, and a browser that sends `mm` per-arrival (the Phase-0 behavior) is fully compatible.