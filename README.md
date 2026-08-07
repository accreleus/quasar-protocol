# protocol/

Shared wire definitions between Quasar components. **These are frozen interfaces** (see `CLAUDE.md`): client and host implementations depend on them being stable, so changes go through Opus + explicit human sign-off.

Phase 0 uses plain JSON for everything to stay debuggable. A compact binary input encoding is a later optimization, noted in `input.md`.

## Phase 0 contracts
- `signaling.md` — WebRTC signaling handshake (WebSocket, browser <-> host). **Extended by P1-D** with the authenticated, control-plane-relayed layer (same message shapes).
- `input.md` — input events (DataChannel, browser -> host) + clock-sync for latency measurement

## Later contracts
- `transport-webtransport.md` — media transport #2 (EXPERIMENTAL): QUIC datagrams over
  WebTransport + WebCodecs decode, replacing the RTP/WebRTC receive stack for sessions that opt
  in. WebRTC stays the default and guaranteed fallback. Additive amendments in `agent-api.md`,
  `control-api.md`; `signaling.md` is bypassed, not changed. Spec: quasar
  `docs/design/plans/2026-08-02-webtransport-webcodecs-spec.md` (quasar#430).
- `native-client.md` — native-client capability/metrics/certification report (AS10-12).

## Phase 1 contracts (P1-A..D)
The four interface contracts that let the control plane, node agent, and web client be built in
parallel on cheaper tiers. They are mutually consistent: the schema shapes the APIs, the APIs
shape signaling. See `../docs/completed/phase1-plan.md` (archived; Phase 1 is complete). Phase 2 amends these contracts additively — see `../docs/phase2/`.

- `agent-api.md` (P1-A) — control-plane ↔ node-agent: registration, capacity, session
  assign/start/stop, state callbacks, signaling relay. The spine of the control-plane/node-agent split.
- `control-api.md` (P1-B) — the public HTTP/JSON API the web client uses: auth, library, launch
  (returns single-use signaling coordinates), session lifecycle.
- `schema.md` (P1-C) — the Postgres schema (`users`, `auth_tokens`, `apps`, `hosts`, `gpus`,
  `sessions`) that replaces Wolf's TOML, plus the migration tooling choice. Authoritative DDL lives
  in `../migrations/`. Defines the **session state machine** and **resource model** the other docs reference.
- `signaling.md` (P1-D section) — the authenticated-signaling extension described above.
