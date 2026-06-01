# Signaling protocol (Phase 0)

WebRTC signaling between the browser **client** and the **host** (the `spike/host` app; later the node agent). Transport: a single WebSocket. Encoding: JSON, one message object per WebSocket frame, discriminated by `type`.

## Roles
- The **host is the offerer.** It has media to send (one video track, optionally one audio track) and it creates the input **DataChannel** as part of the offer. This keeps the client a thin answerer.
- The **client answers** and attaches the incoming track to a `<video>` element. It sends input on the DataChannel the host created.

## Handshake
```
client                         host
  | ---- WS connect --------->  |
  |                             |  (creates PeerConnection, video track,
  |                             |   DataChannel "input"; createOffer)
  | <--- {type:"offer"} ------- |
  | ---- {type:"answer"} -----> |
  | <--> {type:"ice"} <-------> |   (trickle ICE, both directions)
  |                             |
  | ====== media + DataChannel flowing ======
```

## Messages

Host -> client:
```json
{ "type": "offer", "sdp": "<SDP string>" }
{ "type": "ice",   "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } }
```

Client -> host:
```json
{ "type": "answer", "sdp": "<SDP string>" }
{ "type": "ice",    "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 } }
```

Either direction (optional, diagnostics):
```json
{ "type": "error", "message": "..." }
{ "type": "bye" }
```

## Phase 0 simplifications (revisit later)
- No authentication on the WebSocket. Phase 1 puts signaling behind the control plane with an authenticated, single-use session token in the connect URL.
- One client per host process. Multi-session is the node agent's job (Phase 1+).
- STUN/TURN: host-local / LAN needs neither. For WAN testing, configure a STUN server in the PeerConnection config on both ends; TURN is a Phase 3 concern.

---

# P1-D — Authenticated signaling (Phase 1, FROZEN)

In Phase 1, signaling sits **behind the control plane**. The Phase 0 message shapes above are
**unchanged** — P1-D adds only an auth + routing layer around them. This is additive; the
`offer`/`answer`/`ice`/`bye`/`error` objects and the host-is-offerer handshake are identical.
Co-defined with `control-api.md` (mints the token) and `agent-api.md` (relays the messages).

## What changes vs Phase 0
1. **The WebSocket endpoint is the control plane, not the host.** The browser connects to the
   control plane's signaling endpoint (`wss://<control-plane>/v1/signal`), the same origin it
   uses for the control API. The node agent is never contacted directly (architecture
   invariant #1); it has no public address.
2. **The connect URL carries a single-use session token.** The token is minted by
   `POST /v1/sessions` (`control-api.md`) and passed as a query parameter:
   ```
   wss://<control-plane>/v1/signal?token=<single-use>
   ```
   It is a query param (not a header) because browser `WebSocket` cannot set headers. The token
   is single-use, short-TTL, and bound to one session — exposure in a URL is acceptable given
   those properties; it is not a bearer credential and grants nothing after first use.
3. **The control plane validates, consumes, and bridges.** On connect, before any signaling
   message flows, the control plane:
   - hashes the token and looks it up in `sessions.signaling_token_hash` (`schema.md`);
   - rejects (close code **4401**, see below) if not found, expired
     (`signaling_token_expires_at < now()`), already used (`signaling_token_consumed_at` set), or
     the session is terminal;
   - **atomically** sets `signaling_token_consumed_at = now()` (single-use enforced here — a
     second connect with the same token is rejected);
   - looks up the session's assigned host and **relays** signaling to/from that node agent over
     the persistent agent-API connection (`agent-api.md` §Signaling relay), tagging each message
     with `session_id`.

   The browser ↔ control-plane leg speaks the **exact Phase 0 messages**; the control-plane ↔
   node leg wraps them in the agent-API `signaling` envelope. Neither the client nor the node's
   `webrtcbin` needs to know the relay exists.

## Authenticated handshake
```
browser                         control-plane                         node-agent
   | -- WS connect ?token= ----->  |                                      |
   |                               |  validate + consume token (schema)   |
   |                               |  resolve session -> assigned host     |
   |                               | -- signaling{session_id, ...} ------> |  (relay opens)
   |                               | <- signaling{session_id, offer} ----- |  (host is offerer)
   | <----- {type:"offer"} ------- |                                      |
   | ------ {type:"answer"} -----> | -- signaling{session_id, answer} ---> |
   | <----> {type:"ice"} <-------> | <----> signaling{session_id, ice} <-> |  (trickle, both ways)
   |                               |                                      |
   | ===== media + input DataChannel: browser <-> node webrtcbin (peer-to-peer) =====
```
After ICE establishes, media and the input DataChannel (`input.md`) flow **peer-to-peer**
between the browser and the node's `webrtcbin` — they do **not** traverse the control-plane
relay. Only signaling does. (NAT traversal for the media path is STUN now, TURN in Phase 3, per
the Phase 0 note above — unchanged.)

## Token lifecycle (single-use)
- **Mint:** `POST /v1/sessions` generates a cryptographically random token, stores its SHA-256 in
  `signaling_token_hash` with `signaling_token_expires_at = now() + TTL` (default **60 s**),
  returns the plaintext once in the launch response.
- **Connect:** first valid WS connect consumes it (`signaling_token_consumed_at` set
  atomically). The signaling session lives as long as the WebSocket; closing the WS ends
  signaling but not the media session (that's the session lifecycle, `control-api.md` /
  `agent-api.md`).
- **Reuse/expiry/invalid:** rejected with a WS close. One token ⇒ one signaling attempt; a
  reconnect needs a fresh token (re-launch). Phase 1 keeps a single token per session row
  (`schema.md`); a `session_tokens` child table is the documented extension if true reconnect is
  added later.

## WebSocket close codes (control plane → browser)
Application close codes in the 4000–4999 range so the client can react precisely:
| code | meaning |
|---|---|
| `4401` | token invalid / expired / already used |
| `4404` | session not found or terminal (`stopped`/`failed`) |
| `4409` | session not yet assigned to a host (retry shortly) |
| `4500` | relay to node agent unavailable (host offline) |
A normal end is code `1000`; the Phase 0 `{type:"bye"}`/`{type:"error"}` diagnostic messages
still apply in-band before close.

## Phase 1 simplifications (revisit, not workarounds)
- One signaling WebSocket per session; the control plane fans messages by `session_id`.
- Token in the URL query (browser `WebSocket` header limitation); single-use + short-TTL is the
  mitigation. A subprotocol-based scheme can replace it later without touching message shapes.
- The relay is in-process at N=1 (one control-plane instance, one node). Horizontal scale routes
  by the agent connection that owns the session's host — no message-shape change.
