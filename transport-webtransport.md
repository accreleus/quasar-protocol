# WebTransport media transport (transport slot #2, EXPERIMENTAL, FROZEN)

The wire contract for the second media transport: QUIC datagrams over
WebTransport, replacing the RTP/WebRTC receive stack for sessions that opt in.
Design rationale, gates, and milestones:
`../docs/design/plans/2026-08-02-webtransport-webcodecs-spec.md` (quasar#430).

**Frozen** (see `CLAUDE.md`): like the other documents here this is load-bearing —
the node agent, web client, and future native client implement against it, so
changes need Opus + explicit human sign-off. The protocol carries its own version
(`wt_proto`, §Hello) precisely so that v0 can be corrected by a versioned v1
rather than by mutating this document.

> **Status: EXPERIMENTAL (M0 contract; no implementation yet).** The WebRTC path
> (`signaling.md`) remains the default and the guaranteed floor for every
> session. A client that never sends `transport` in `POST /v1/sessions` sees no
> change anywhere. `wt_proto=0` is a spike-grade contract: resilience (NACK,
> FEC) and congestion-feedback frames are **reserved but unspecified** here and
> land as additive amendments (spec §8, M3/M4) before the path exits
> experimental.

## Relationship to the invariants

- **Invariant #1 (client never contacts the node agent) is about the control
  API, not the media path.** Today's WebRTC media already flows directly
  browser↔agent (ICE-established UDP); the control plane never proxies media.
  WebTransport is the same shape with an explicit URL instead of ICE: the
  client still gets *authorization* exclusively from the control plane (the
  single-use `wt` token minted at launch), and the agent accepts a connection
  only when it can validate that token against the assignment the control
  plane delivered. All account/session/API traffic stays on the control plane.
- **Invariant #3 (transport is an interface):** this is implementation #2 of
  the transport slot. Compositor and encoder contracts are untouched; nothing
  upstream of the encoded-frame boundary appears in this document.
- **`input.md` is reused verbatim** (§Input channel). The input wire format is
  not forked per transport.

## Connection

One WebTransport session per streaming session:

```
https://<agent_host>:<wt_port>/wt/session/<session_id>
```

- `<agent_host>`/`<wt_port>`: from the launch response
  (`control-api.md` §`POST /v1/sessions`, `transport_options.webtransport`).
  One QUIC listener per agent serves all sessions.
- TLS: the agent's existing certificate material (`QUASAR_TLS_HOSTS`
  discipline applies). The launch response MAY carry `cert_hashes`
  (SHA-256, base64) for `serverCertificateHashes`-pinned dev setups.
- The client MUST enable datagrams and MUST close the WebTransport session on
  teardown (`bye`, §Control stream) — QUIC close is the authoritative
  disconnect signal for the agent.

### Authentication

The first and only credential is the single-use `wt` token:

- Minted by the control plane at launch (and by the reconnect endpoint),
  stored **hashed** with the session, delivered to the agent inside
  `session_assign` (`agent-api.md` amendment) and to the client in the launch
  response. Plaintext appears exactly once per mint, in the API response.
- Short TTL (default 60 s, same policy as `signaling.token`).
- Presented in the `hello` message (first message on the control stream —
  §Hello), NOT in the URL query (tokens do not belong in URLs; the path's
  `session_id` is routing, not authorization).
- The agent validates hash + TTL + session-state locally against the
  assignment; failure ⇒ `error {code:"auth_failed"}` on the control stream,
  then close. No agent→control-plane round trip is on the connect path.
- One live WebTransport session per token; a second connect with a consumed
  token is refused (`auth_failed`). Reconnect = mint a fresh token
  (`control-api.md` §`POST /v1/sessions/{id}/transport-token`).

## Channels

Two QUIC primitives are used:

1. **One bidirectional control stream** (reliable, ordered) — opened by the
   client immediately after session establishment. JSON, length-prefixed
   (u32-LE byte length, then UTF-8 JSON), discriminated by `type` — the
   signaling.md/agent-api.md discipline, kept for debuggability.
2. **Datagrams** (unreliable, unordered) — all media and input. Binary.
   First byte is the channel id:

   | id     | channel | direction      |
   |--------|---------|----------------|
   | `0x01` | video   | agent → client |
   | `0x02` | audio   | agent → client |
   | `0x03` | input   | client → agent |
   | `0x04` | FEC repair (reserved, M3) | agent → client |

   Unknown channel ids MUST be ignored (additive growth).

All multi-byte binary fields are little-endian. Timestamps are the session
media clock: 90 kHz for video, 48 kHz for audio — the same clocks the trace
pipeline (`control-api.md` trace ingest) already uses, so glass-to-glass
decomposition works unchanged.

## Control stream messages

### Hello (client → agent, first message)
```json
{ "type": "hello", "wt_proto": 0, "token": "<single-use wt token>",
  "client": "browser" }
```
`client ∈ {"browser","native"}` (informational; mirrors the `client_type`
vocabulary of `control-api.md`). The agent replies:

```json
{ "type": "hello_ack", "wt_proto": 0,
  "video": { "codec": "h264",
             "description_b64": "<avcC/hvcC/av1C for VideoDecoder.configure>",
             "width": 1920, "height": 1080, "fps": 60 },
  "audio": { "codec": "opus", "channels": 2, "sample_rate": 48000,
             "ptime_ms": 10 } }
```
- `video.codec` uses the **wire** vocabulary of `agent-api.md` `capacity.codecs`
  (`h264|h265|av1`); the session's codec was resolved at launch exactly as for
  WebRTC — this transport does not renegotiate codecs.
- `description_b64` is the codec description blob in the form
  `VideoDecoder.configure({description})` expects. Absent for codecs that
  carry config in-band (AV1). A new blob (e.g. after an encoder restart) is
  announced by re-sending `hello_ack`; the client MUST reconfigure its decoder
  before decoding subsequent frames.
- Version mismatch: the agent answers with the highest `wt_proto` it speaks;
  if the client cannot proceed it sends `bye` and falls back to WebRTC. An
  agent that cannot speak the client's version at all replies
  `error {code:"unsupported_version"}` and closes.

### Keyframe request (client → agent)
```json
{ "type": "keyframe_request", "reason": "loss" | "decode_error" | "startup" }
```
Replaces PLI. The agent forces an IDR through the same path ABR/PLI uses
today. Agents MAY coalesce requests (≥1 IDR after ≥1 request; no 1:1
guarantee).

### Feedback (client → agent, reserved shape — M4 specifies the fields)
```json
{ "type": "feedback", "...": "reserved" }
```
Reserved for the receive/decode/present statistics that drive ABR and
congestion control (spec §8). v0 clients MAY omit it entirely; v0 agents MUST
ignore it.

### Error / bye (either direction)
```json
{ "type": "error", "code": "auth_failed" | "unsupported_version" | "internal",
  "message": "..." }
{ "type": "bye" }
```
`bye` then QUIC close is the clean teardown. Session lifecycle state remains
owned by the control plane / agent-api exactly as for WebRTC sessions; this
channel's disconnect feeds the same disconnect handling the WebRTC peer
teardown does today.

## Video channel (`0x01`, agent → client)

Encoded access units, fragmented to fit a datagram (payload budget ≈1200 B —
the agent MUST respect the connection's actual max datagram size):

```
offset  size  field
0       u8    channel = 0x01
1       u16   frame_seq        // per-session AU counter, wraps
3       u8    flags            // bit0 keyframe, bit1 discardable
4       u16   frag_index       // 0-based
6       u16   frag_count       // ≥1
8       u32   ts_90khz         // media clock
12      ...   payload
```

- Payload is the AU in **length-prefixed** form (AVCC/HVCC-style 4-byte NAL
  lengths for H.264/HEVC; low-overhead OBU stream for AV1) — the form
  `EncodedVideoChunk` consumes; no Annex-B start codes on the wire.
- Reassembly is keyed by `frame_seq`; all fragments of one AU carry identical
  `flags`, `frag_count`, `ts_90khz`.
- v0 loss policy (client): a frame still incomplete when a fragment for
  `frame_seq + 2` (mod wrap) arrives is abandoned; if the abandoned frame was
  not `discardable`, send `keyframe_request {reason:"loss"}` and drop frames
  until the next keyframe. NACK/RTX and FEC supersede parts of this in M3
  (additive amendment).
- The agent sends frames in encode order; `frame_seq` increments by exactly 1
  per AU.

## Audio channel (`0x02`, agent → client)

One Opus packet per datagram, 10 ms ptime (`hello_ack.audio`):

```
0       u8    channel = 0x02
1       u16   packet_seq       // wraps
3       u32   ts_48khz
7       ...   Opus packet
```

Lost audio datagrams are concealed client-side (Opus PLC or silence); there is
no audio retransmission at any version. The client owns the audio buffer
(spec §7).

## Input channel (`0x03`, client → agent)

```
0       u8    channel = 0x03
1       ...   one input.md JSON event, UTF-8, verbatim
```

The `input.md` wire format — including its clock-sync events — is carried
unchanged, one event per datagram. This is deliberately identical to the
DataChannel payload so the agent-side input path and the client-side capture
path (`web/src/input/capture.ts`) are transport-agnostic.

v0 accepts input-datagram loss (events are absolute-state-heavy; see
`input.md`). A reliable input fallback, if measurement shows it is needed, is
an M2 decision and would be an additive amendment (mirroring events on the
control stream).

## Versioning

- `wt_proto` is a monotonically increasing integer; 0 is this document.
- Within a `wt_proto`, growth is **additive only**: new control-message
  `type`s (ignored when unknown), new datagram channel ids (ignored when
  unknown), new optional JSON fields. Layout changes to the binary headers
  above require a `wt_proto` bump.
- Reserved-but-unspecified pieces of v0 (channel `0x04`, `feedback`) are
  specified by additive amendment to this document (sign-off as usual) before
  M3/M4 implement them.
