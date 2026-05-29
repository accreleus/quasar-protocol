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
