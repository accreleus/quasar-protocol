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

> **Note (#509, 2026-08-26) — ICE servers are not signaled here.** The STUN/TURN servers a client
> configures its `RTCPeerConnection` with arrive on the signaling **coordinates** (`control-api.md`
> `SignalingCoords.ice_servers`), which the client already holds before it opens this WebSocket.
> That is the only moment the list can be applied, since the peer connection is constructed from
> it. **No message in this contract changes**: this file stays a relay of host↔client
> offer/answer/ICE, and a host with no STUN of its own still offers host candidates exactly as
> before.

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

### `pc` field — separate audio/video PeerConnections (amendment, #304)

The `offer`, `answer`, and `ice` messages gain an **optional** `pc` field identifying which
PeerConnection the message belongs to:

```json
{ "type": "offer", "pc": "video", "sdp": "<SDP string>" }
{ "type": "offer", "pc": "audio", "sdp": "<SDP string>" }
{ "type": "answer", "pc": "video", "sdp": "<SDP string>" }
{ "type": "answer", "pc": "audio", "sdp": "<SDP string>" }
{ "type": "ice", "pc": "video", "candidate": { ... } }
{ "type": "ice", "pc": "audio", "candidate": { ... } }
```

- `pc` is `"video"` or `"audio"`. The **video** PeerConnection carries the video track and the
  `"input"` DataChannel; the **audio** PeerConnection carries the host→client audio track and,
  when negotiated, the client→host microphone m-line (see the microphone amendment below).
- `pc` is **optional**. When absent, the message belongs to the `"video"` PeerConnection (this
  is the pre-amendment behaviour — a single PeerConnection carrying video + audio + DataChannel).
  This keeps the field backwards-compatible: an old client that doesn't send `pc` still works
  with a host that only creates the video PeerConnection.
- Splitting audio into a separate PeerConnection eliminates browser A/V clock-coupling latency:
  Chrome's NetEQ + RTCP SR A/V sync pulls video playout toward the audio jitter buffer, inflating
  video latency by ~15ms even with 10ms Opus frames and `mode=synced` jitter buffers (measured on
  Quasar, issue #304 Phase 0). Separate PeerConnections give the browser no audio track to couple
  against on the video PC, reclaiming that floor.
- The host creates **two** `webrtcbin` instances (one per PeerConnection). Each fires its own
  `on-negotiation-needed` → its own offer, and its own ICE candidates. The single-offer guard is
  per-webrtcbin (each negotiates exactly once).
- The `"input"` DataChannel is created on the **video** webrtcbin only. Input latency is unaffected
  by audio jitter buffer behaviour.
- Both PeerConnections use the same STUN/TURN config. ICE gathering runs in parallel for both.
- An audio-disabled session (`QUASAR_AUDIO_DISABLED=1` on the agent) creates only the video
  PeerConnection; no `pc: "audio"` messages are sent.

### Microphone m-line on the audio PeerConnection (amendment, 2026-08-02)

When a session is launched with microphone capture granted (see `control-api.md`
`POST /v1/sessions` `mic` and `agent-api.md` `session_assign` `stream.mic`), the host's
`pc:"audio"` offer carries **two** audio m-lines instead of one:

1. the existing host→client audio m-line (host `sendonly`), and
2. a client→host microphone m-line (host `recvonly`), Opus, clock-rate 48000.

Rules:

- The mic m-line is present in the **first** (and only) audio offer or not at all. The
  single-offer-per-PeerConnection rule is unchanged — microphone availability is decided at
  launch and never renegotiated mid-session.
- The client answers the mic m-line with `sendonly` (its perspective). It MAY answer with no
  live track attached and attach/replace the microphone track later
  (`RTCRtpSender.replaceTrack`); enabling/disabling the mic mid-session is a client-local
  track operation and produces **no signaling messages**.
- A session launched without mic (or on a host/instance where the feature is disabled)
  produces exactly today's single-m-line audio offer — the pre-amendment wire is unchanged.
- Message vocabulary, `pc` values, and all other signaling rules are unchanged.
- An audio-disabled session (`QUASAR_AUDIO_DISABLED=1`) has no audio PeerConnection and
  therefore no microphone path.

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

> **Amendment — P2-02 (launcher↔game swap): signaling is UNCHANGED.** The app-swap operation
> (`control-api.md` `POST /v1/sessions/{id}/swap`, `agent-api.md` `session_swap_app`) swaps a
> session's source container **behind** the encoder via a GStreamer interpipe boundary while the
> encode tail and `webrtcbin` stay up. There is **no new offer, no answer, no ICE restart, and no
> new DataChannel** — the same PeerConnection carries the same media track and input DataChannel
> across the swap; only the pixels flowing into the encoder change. This "no renegotiation" is the
> **defining constraint** of swap (it is what makes it seamless to the browser): if the P2-07
> implementation ever finds it needs to renegotiate, that is a contract change — **stop and
> escalate** (Opus + sign-off), do not add a signaling message to make swap work. No shape on this
> page changes. See `docs/phase2/P2-02-contract-app-swap.md`.

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

## Token lifecycle (single-use, repeatable per session)
- **Mint:** `POST /v1/sessions` generates a cryptographically random token, stores its SHA-256 in
  `signaling_token_hash` with `signaling_token_expires_at = now() + TTL` (default **60 s**),
  returns the plaintext once in the launch response.
- **Connect:** first valid WS connect consumes it (`signaling_token_consumed_at` set
  atomically). The signaling session lives as long as the WebSocket; closing the WS ends
  signaling but not the media session (that's the session lifecycle, `control-api.md` /
  `agent-api.md`).
- **Reconnect mint:** an authenticated owner may call
  `POST /v1/sessions/{id}/signaling-token` while the session is `assigned`, `starting`, or
  `running`. Each call creates a new independently single-use token; it does not create, stop, or
  reschedule the session.
- **Reuse/expiry/invalid:** rejected with a WS close. One token ⇒ one signaling attempt; a
  reconnect obtains a fresh token for the same session.

## Mid-session reconnection
Two bounded recovery paths are defined:

- **Transient media-path loss, signaling still connected.** The answerer sends
  `{ "type":"restart_ice", "pc":"video" }`. The host, as offerer, performs an ICE restart and
  sends a new `offer` for that PC; the normal answer/trickle flow follows. Audio may be requested
  independently with `pc:"audio"`. Duplicate requests are idempotent while negotiation is in
  progress.
- **Signaling or PeerConnection loss.** The authenticated client mints a replacement token for
  the same session, opens a new signaling WebSocket, and recreates its peer connections. Client
  attachment causes the still-running host pipeline to emit fresh offers. The session id,
  scheduler reservation, container, and application remain unchanged.

Clients use bounded retry/backoff and expose cancellation. A terminal session returns `409
session_not_reconnectable`; an unavailable host remains distinguishable through close code 4500.
The `restart_ice` request is additive; all existing offer/answer/ICE shapes remain valid.

## WebSocket close codes (control plane → browser)
Application close codes in the 4000–4999 range so the client can react precisely:
| code | meaning |
|---|---|
| `4401` | token invalid / expired / already used |
| `4404` | session not found or terminal (`stopped`/`failed`) |
| `4409` | session not yet assigned to a host (retry shortly) |
| `4410` | this attachment was taken over by a later attach (terminal for this client: render "session taken over elsewhere"; do NOT mint a replacement token or reconnect) |
| `4500` | relay to node agent unavailable (host offline) |
A normal end is code `1000`; the Phase 0 `{type:"bye"}`/`{type:"error"}` diagnostic messages
still apply in-band before close.

Attach is explicit takeover: signaling tokens are single-use but not single-active, the last
attach for a session wins, and the displaced connection is closed with `4410` (added 2026-08-25;
previously this path closed with `1000`, which a client cannot distinguish from an ordinary
hang-up — auto-recovering from it produced a two-tab displacement loop). A client that does not
recognise `4410` falls into its unrecognised-code handling, same as before the amendment.

## Phase 1 simplifications (revisit, not workarounds)
- One signaling WebSocket per session; the control plane fans messages by `session_id`.
- Token in the URL query (browser `WebSocket` header limitation); single-use + short-TTL is the
  mitigation. A subprotocol-based scheme can replace it later without touching message shapes.
- The relay is in-process at N=1 (one control-plane instance, one node). Horizontal scale routes
  by the agent connection that owns the session's host — no message-shape change.
