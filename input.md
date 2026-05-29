# Input protocol (Phase 0)

Input events travel **client -> host** over the WebRTC **DataChannel** named `input`. The same channel carries clock-sync pings used for latency measurement (ticket T7).

Encoding: JSON, one object per channel message, discriminated by `t`. (A compact binary encoding is a later optimization — keep JSON for Phase 0 debuggability.)

DataChannel is configured **unreliable + unordered** for input events (`ordered:false, maxRetransmits:0`) so a late packet never head-of-line-blocks newer input. Clock-sync pings tolerate loss too.

## Coordinate + code conventions
- Relative motion (`dx`,`dy`) is in device pixels at the streamed resolution.
- Absolute motion (`x`,`y`) is normalized `0.0..1.0` across the streamed surface.
- Scroll (`dx`,`dy`) is in high-resolution wheel units.
- Mouse buttons use Linux input codes: left `0x110` (BTN_LEFT), right `0x111`, middle `0x112`.
- Keys use **Linux evdev keycodes**. The client maps `KeyboardEvent.code` -> evdev keycode (a static lookup table; lives in the client). The host passes the evdev code straight to the compositor.
- Gamepad: button/axis indexing follows the **W3C Standard Gamepad** layout from the browser Gamepad API. The host maps this to the virtual pad it presents (Phase 0 may map to compositor input directly; Phase 1 routes through inputtino so a containerized game sees a real evdev device).

## Messages (client -> host)
```json
{ "t": "mm", "dx": 4,   "dy": -2 }                      // mouse move, relative
{ "t": "ma", "x": 0.51, "y": 0.42 }                     // mouse move, absolute (normalized)
{ "t": "mb", "button": 272, "pressed": true }           // mouse button (272 = 0x110 left)
{ "t": "ms", "dx": 0, "dy": 120 }                       // scroll
{ "t": "k",  "code": 30, "pressed": true }              // key (evdev code; 30 = KEY_A)
{ "t": "gp", "i": 0, "buttons": [0,1,0,...], "axes": [0.0,-0.3,...] } // gamepad state on change
```

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
