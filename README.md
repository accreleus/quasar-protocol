# protocol/

Shared wire definitions between Quasar components. **These are frozen interfaces** (see `CLAUDE.md`): client and host implementations depend on them being stable, so changes go through Opus + explicit human sign-off.

Phase 0 uses plain JSON for everything to stay debuggable. A compact binary input encoding is a later optimization, noted in `input.md`.

- `signaling.md` — WebRTC signaling handshake (WebSocket, browser <-> host)
- `input.md` — input events (DataChannel, browser -> host) + clock-sync for latency measurement
