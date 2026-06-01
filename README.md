# protocol/

Shared wire definitions between Quasar components. **These are frozen interfaces** (see `CLAUDE.md`): client and host implementations depend on them being stable, so changes go through Opus + explicit human sign-off.

Phase 0 uses plain JSON for everything to stay debuggable. A compact binary input encoding is a later optimization, noted in `input.md`.

## Phase 0 contracts
- `signaling.md` — WebRTC signaling handshake (WebSocket, browser <-> host). **Extended by P1-D** with the authenticated, control-plane-relayed layer (same message shapes).
- `input.md` — input events (DataChannel, browser -> host) + clock-sync for latency measurement

## Phase 1 contracts (P1-A..D)
The four interface contracts that let the control plane, node agent, and web client be built in
parallel on cheaper tiers. They are mutually consistent: the schema shapes the APIs, the APIs
shape signaling. See `../docs/phase1-plan.md`.

- `agent-api.md` (P1-A) — control-plane ↔ node-agent: registration, capacity, session
  assign/start/stop, state callbacks, signaling relay. The spine of the control-plane/node-agent split.
- `control-api.md` (P1-B) — the public HTTP/JSON API the web client uses: auth, library, launch
  (returns single-use signaling coordinates), session lifecycle.
- `schema.md` (P1-C) — the Postgres schema (`users`, `auth_tokens`, `apps`, `hosts`, `gpus`,
  `sessions`) that replaces Wolf's TOML, plus the migration tooling choice. Authoritative DDL lives
  in `../migrations/`. Defines the **session state machine** and **resource model** the other docs reference.
- `signaling.md` (P1-D section) — the authenticated-signaling extension described above.
