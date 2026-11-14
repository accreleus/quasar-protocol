# WebTransport media transport (transport #2, EXPERIMENTAL, FROZEN)

*(Amendment 54, accreleus/quasar#1026, spec accreleus/quasar#1024, pre-approved by the owner
2026-11-06; independent Opus review before merge.)*

The wire contract for the second implementation of the transport slot: media and input as QUIC
datagrams over WebTransport, decoded by the client itself (WebCodecs in a browser). WebRTC
(`signaling.md`) stays the default and the guaranteed floor: every session that may use this
transport also gets WebRTC signaling coordinates, and a client that cannot connect here, or loses
the stream, falls back to WebRTC.

**Frozen** like every document here: the node agent, the web client and a future native client
implement against it, so a change needs Opus and the owner's sign-off. The protocol carries its
own version, `wt_proto` (§Hello), so a correction is a new version rather than an edit.

This document replaces the unmerged draft of accreleus/quasar-protocol#12 (`wt_proto` 0, never
implemented). The first version anything implements is **`wt_proto` 1**.

Co-defined with `agent-api.md` (the endpoint report, the token hash, the takeover rule),
`control-api.md` (the launch response, the token endpoint, the transport choice, the host endpoint
override) and `schema.md` (migration 0111).

## Relationship to the invariants

- **Invariant 1 (control plane / node agent split) and invariant 6 (the client talks only to the
  control plane).** The API and all control traffic stay on the control plane. The **media
  transport** may connect to the agent, or to an edge in front of it, with a short-lived token the
  control plane mints; the agent never authorizes anything on its own. WebRTC media already flows
  browser↔agent over ICE today; this transport makes the address explicit. The quasar-side ADR
  0010 (accreleus/quasar#1027) rewords `CLAUDE.md` invariant 6 to say exactly that: **media may
  connect to the agent with a control-plane-minted token; API and control never.**
- **Invariant 3 (transport is an interface).** This is transport #2. Nothing upstream of the
  encoded-frame boundary appears here: the codec, the encoder, the ABR ladder and the compositor
  are the session's, identical on both transports, so a fallback never re-encodes or renegotiates
  a codec. *(Amendment 55, accreleus/quasar#1050.)* The one exception is PyroWave
  (`codec-pyrowave.md`), a WebTransport-only codec a session may switch to and from
  mid-session; a WebRTC peer always gets the session's standard codec.
- **`input.md` is reused verbatim** (§Input). The input format is not forked per transport.

## Endpoint and URL

The client gets one **opaque** `https` URL from the control plane
(`control-api.md` `transport_options.webtransport.url`) and opens a WebTransport session to it with
datagrams enabled. It never builds, parses or rewrites the URL.

The control plane picks the URL per host (`control-api.md` §WebTransport endpoint):

- **Direct.** The agent's own QUIC listener: `https://<address>:<port>/wt`, from the agent's
  `capacity.webtransport` report (`agent-api.md`). The agent presents a self-signed certificate and
  the client pins it with `serverCertificateHashes` (§Certificates).
- **Edge.** An admin-set public URL for the host, e.g. a reverse proxy with a publicly trusted
  certificate in front of the agent's listener. The client trusts the certificate normally and gets
  no hashes. How the edge reaches the agent, and how it trusts the agent's certificate, is the
  operator's and is outside this contract.

A future Quasar relay (the WebTransport counterpart of TURN) is one more URL source: it needs no
change here, because the URL is opaque and the session is named by the token, not the URL.

The agent accepts a WebTransport session at any path on its listener. **The path is not routing
and not authorization**: the token in `hello` names the session. One listener serves every session
on the host.

## Certificates

In direct mode the agent's listener presents a certificate the agent made itself, under the
browser rules for `serverCertificateHashes`:

- ECDSA with the P-256 curve; X.509 v3;
- a validity period (`notAfter` − `notBefore`) of **at most 14 days**; the agent SHOULD use 13
  days, with `notBefore` up to one hour before it was made, to absorb clock skew.

**Rotation, with an overlap.** The agent reports every certificate it may present, current first,
as `capacity.webtransport.certificates` (`agent-api.md`), each with its SHA-256 hash and validity.

1. When the current certificate has **72 hours or less** left, the agent makes the next one and
   reports both.
2. When the current one has **24 hours or less** left, the agent presents the next one on new
   connections. Live connections keep theirs.
3. Once the old one has expired the agent stops reporting it.

The agent re-sends `capacity` whenever that list changes. The control plane sends the client the
hash of every reported certificate that has not expired (at most two, during the overlap), so a
client connecting just before or just after the switch accepts either one. An agent restart may
make a new certificate at once; it is reported before the listener accepts connections with it.

An agent configured to present an operator-supplied, publicly trusted certificate reports
`certificates: []`, and the control plane then sends no hashes.

## Authentication and tokens

The only credential is the single-use **transport token**:

- Minted by the control plane at launch and by `POST /v1/sessions/{id}/transport-token`
  (`control-api.md`); plaintext exactly once, in that response; stored hashed (`schema.md`
  `session_tokens`, `kind = 'transport'`); TTL 60 s, the `signaling.token` policy.
- Delivered to the agent as a hash: lowercase hex SHA-256 of the token's UTF-8 bytes, the form the
  control plane stores. It rides `session_assign` at launch and `session_transport_token` for a new
  mint (`agent-api.md`). The agent validates locally: no control-plane round trip on the connect
  path.
- Presented in `hello`, **never in the URL**.
- **Expiry.** The agent accepts a token until its `wt_token_expires_at`, read on the agent's own
  clock with an allowed skew of 15 s (so until `wt_token_expires_at` + 15 s).
- **Single use.** A `hello` that the agent accepts consumes the token. A `hello` refused with
  `session_not_ready` or `takeover_rate_limited` does not, so the client may retry with the same
  token inside its TTL.
- **Tokens do not cross.** A transport token is stored as `session_tokens.kind = 'transport'` and a
  signaling token as `kind = 'signaling'` (`schema.md`). `/v1/signal` redeems only `signaling`
  rows, and only `transport` rows' hashes ever reach the agent, so the agent redeems only transport
  tokens. A transport token presented to `/v1/signal` is closed `4401`, and a signaling token in a
  `hello` is `auth_failed` (`4401`). An implementation test proves both directions.

**Reconnect** (a reload, a lost connection): the client mints a fresh token from
`POST /v1/sessions/{id}/transport-token` and connects again; the new connection takes over the old
one (§Takeover).

## Takeover

One media client per session, across both transports. The last client to authenticate wins, as on
WebRTC (`signaling.md` "Attach is explicit takeover", amendment 27):

- **WebTransport takes over WebTransport.** On a `hello` with a valid token while another client is
  connected, the agent accepts the new one, sends the old one `bye {reason: "taken_over"}` and
  closes it with code `4410`. The displaced client MUST NOT mint a token or reconnect; it shows
  "session taken over elsewhere", as for signaling close `4410`.
- **WebTransport takes over WebRTC.** The agent stops sending media to the WebRTC peer and sends
  the client that holds signaling `{ "type": "error", "code": "taken_over" }` through the relay; the
  control plane then closes that signaling WebSocket with `4410` (`signaling.md`, amendment 54).
- **WebRTC takes over WebTransport.** A new WebRTC peer that the host accepts under amendment 27
  displaces the WebTransport client the same way: `bye {reason: "taken_over"}`, close `4410`.
- **Bound.** At most 6 takeovers of a session in any 60 s, counted across both transports. Past
  that the agent refuses a new `hello` with `error {code: "takeover_rate_limited"}` and closes
  it, without consuming its token; the connected client keeps streaming.

## Falling back to WebRTC

A client that falls back closes its WebTransport session first (`bye {reason: "client_closed"}`,
close `0`, or abandoning a connect that never completed) and sends no `hello` once it has opened
signaling for the session. So a late `hello` from the same client can never take over its own
WebRTC peer (§Takeover). `control-api.md` §Transport fallback says when a client falls back.

## Channels

Two QUIC primitives:

1. **One bidirectional control stream**, opened by the client as soon as the WebTransport session
   is up. Reliable and ordered. Each message is a u32 little-endian byte length, then that many
   bytes of UTF-8 JSON, one object discriminated by `type`. **No message is longer than 64 KiB**
   (65 536 bytes): a receiver that reads a larger length closes the connection with `4400`. A
   receiver ignores a `type` it does not know, and fields it does not know.
2. **Datagrams**, unreliable and unordered: media, FEC, input and microphone. Binary. The first
   byte is the channel:

   | id | channel | direction |
   |---|---|---|
   | `0x01` | video | agent → client |
   | `0x02` | audio | agent → client |
   | `0x03` | input | client → agent |
   | `0x04` | FEC repair | agent → client |
   | `0x05` | microphone | client → agent |
   | `0x06` | audio redundancy | agent → client |

   A receiver drops a datagram on a channel it does not know.

Every multi-byte binary field is little-endian.

**Datagram size.** The agent sizes every datagram so that its QUIC packet fits a **1200-byte UDP
payload**, the size QUIC guarantees on any path. Only on a direct path where the agent's QUIC
stack has completed path-MTU discovery may it use more, and never more than the connection's
current maximum datagram size. A path through an edge proxy or a relay stays at 1200. Keeping to
1200 also keeps a retransmission valid if the maximum shrinks. With the worst-case QUIC short
header, AEAD tag, DATAGRAM frame and WebTransport session id (about 52 bytes), a datagram carries at
least about 1148 bytes, so a video fragment carries at least about 1123 bytes of access unit
after its 25-byte header.

**Clocks.** Media timestamps are 90 kHz (video) and 48 kHz (audio), as on RTP. `capture_us` is the
agent's wall clock (Unix epoch, microseconds). For audio it is when the packet's samples began.
For video it is when the frame's access unit **left the encoder** *(amendment 63, signed off 2026-11-10; erratum
quasar-protocol#29 item 13: the 0.6 agent has no capture-time stamp to carry, and the WebRTC
path's abs-capture-time is stamped after the encoder too)*, so capture and encode are not inside
any interval measured from it; the
client maps it to its own clock with the control-stream `ping`/`pong` offset (§Ping and pong), which
is what the deep glass-to-glass trace needs.

**`pkt_seq`.** Every agent → client datagram (video, audio, audio redundancy, FEC) carries a u32
`pkt_seq`, one counter for the whole connection, +1 per datagram sent, retransmissions included.
It is what receiver feedback reports on (§Feedback).

**Wrap-around.** `pkt_seq` (u32), `frame_seq` and every other sequence number here wrap, and are
compared with modulo (serial-number) arithmetic: `a` is after `b` when `(a − b) mod 2ⁿ` is
non-zero and below `2ⁿ⁻¹`. The same holds for `first_pkt_seq + count` in §Feedback.

## Before `hello`

Until the agent sends `hello_ack`, the connection is unauthenticated, and the agent bounds what it
does for it:

- It reads only the **first bidirectional stream** the client opens, as the control stream. It
  refuses every other stream (unidirectional, or a second bidirectional one) with
  `STOP_SENDING`/`RESET_STREAM`, before and after `hello_ack`.
- It **drops every datagram** that arrives before it has sent `hello_ack`, and **sends no datagram**
  before it, so before authentication it sends nothing beyond the QUIC/WebTransport handshake and
  its reply to `hello`. (QUIC's anti-amplification limit stops applying once the handshake
  validates the client's address, which is before `hello`; it is not what bounds this.)
- It closes the connection (`4400`) when no complete `hello` has arrived **5 s** after the
  WebTransport session was established, and on a control message over 64 KiB (§Channels).
- It holds at most **64 unauthenticated connections** per host by default (agent-configured); past
  that it refuses new WebTransport sessions until one authenticates or closes. It MAY also cap
  unauthenticated connections per source address, so one source reconnecting every 5 s cannot
  hold every slot; behind an edge proxy all clients share the proxy's address, so this cap is
  optional and agent-configured.

The client sends nothing but `hello` (and then waits) until it receives `hello_ack`.

## Control stream

### Hello (client → agent, the first message)

```json
{ "type": "hello", "wt_proto": 1, "token": "<transport token>",
  "session_id": "<uuid>", "client": "browser", "fec": ["xor"] }
```

- `client`: `"browser"` or `"native"`, informational (the `client_type` vocabulary of
  `control-api.md`).
- `session_id`: a cross-check; a token for another session is `auth_failed`.
- `fec`: the FEC schemes the client can decode (§FEC). Absent or `[]`: the agent sends no FEC.
- `audio_red` *(amendment 67, optional)*: `true` when this client takes channel `0x06`, the repeat
  of the previous audio packet (§Audio redundancy). Absent or `false`: the agent sends none.
  An older client must leave it out: it would drop the channel as unknown and then report its
  `pkt_seq` as lost.
- `pyrowave` *(amendment 55, optional)*: `true` when this client can decode PyroWave on this
  connection (`codec-pyrowave.md`). Absent: never PyroWave.

The agent replies `hello_ack`, or `error` then closes:

| `error.code` | meaning | client |
|---|---|---|
| `auth_failed` | token unknown, expired, consumed, or not this session's (close `4401`). An assigned session with a token it does not know is `auth_failed`: its first token arrives with `session_assign`, and a reconnect token reaches the client only after the agent acked it | mint a fresh token once; then fall back |
| `session_not_ready` | the agent does not hold this session's assignment yet (a `hello` raced `session_assign`) | retry the same token after 250 ms, inside its TTL |
| `session_ended` | the session is stopping or ended (close `4404`) | end; do not reconnect |
| `unsupported_version` | the agent speaks no `wt_proto` the client offered | fall back to WebRTC |
| `takeover_rate_limited` | §Takeover | fall back or retry later |
| `internal` | anything else | fall back to WebRTC |

`code` is an open string: a client treats an unknown `code` as `internal`.

### Hello ack (agent → client)

```json
{ "type": "hello_ack", "wt_proto": 1, "config_id": 0,
  "video": { "codec": "h264", "description_b64": "<avcC>",
             "width": 1920, "height": 1080, "fps": 60,
             "color_space": { "primaries": "bt709", "transfer": "bt709",
                              "matrix": "bt709", "full_range": false } },
  "audio": { "codec": "opus", "channels": 2, "sample_rate": 48000, "ptime_ms": 10 },
  "mic":   { "codec": "opus", "channels": 1, "sample_rate": 48000, "ptime_ms": 10 },
  "fec": "xor",
  "retransmit_window_ms": 250 }
```

- `video.codec` is the session's codec in the wire vocabulary `h264 | h265 | av1`, resolved at
  launch exactly as for WebRTC. This transport never renegotiates it. *(Amendment 55,
  accreleus/quasar#1050.)* Or `pyrowave`, for a client that sent `hello.pyrowave: true`: such a
  session may switch between `pyrowave` and its standard codec mid-session, each switch a new
  `config_id` (`codec-pyrowave.md` §Switching codec).
- `description_b64`: the decoder configuration record, base64: `avcC` for H.264, `hvcC` for HEVC,
  `av1C` (optional) for AV1. It is what `VideoDecoder.configure({description})` takes.
- `color_space`: the `VideoColorSpaceInit` the encoder signals (`bt709` or `smpte170m` for
  BT.601). Optional; when absent the client uses what the bitstream signals.
- `mic`: present only when the session was granted a microphone (`session_assign.stream.mic`);
  `null` otherwise, and the agent drops channel `0x05`.
- `fec`: the scheme the agent will send, one of the client's `hello.fec`, or `null`.
- `audio.redundancy` *(amendment 67, optional)*: present, `1`, only when the agent will send channel
  `0x06` (the client's `hello.audio_red` was `true`): the number of earlier audio packets each
  audio datagram repeats. `1` is the only value defined.
- `retransmit_window_ms`: how long the agent keeps video datagrams for retransmission (§NACK).

**`config_id` and a new description.** `config_id` (0–255) names the decoder configuration in
`video`; every video datagram carries the `config_id` its frame was encoded under (§Video). When
the encoder's configuration changes (an external resize, an encoder restart), the agent sends
`hello_ack` again with the new `video` and `config_id` + 1 (mod 256), **before** it sends the first
frame under it, which is a keyframe. A re-sent `hello_ack` repeats every field.

Datagrams can still overtake the control stream, so the client reconfigures by `config_id`, not
on receipt: a frame whose `config_id` it has no `hello_ack` for is **held** until that `hello_ack`
arrives, then decoded under it. A client holds such frames for at most 1 s; past that it drops
them and sends `keyframe_request {reason: "decode_error"}`. A frame whose `config_id` is older than
the current one is dropped.

**Version.** The client sends the highest `wt_proto` it speaks; the agent answers with the highest
it speaks that is not higher. A client that cannot use the answer sends `bye` and falls back.

### Keyframe request (client → agent)

```json
{ "type": "keyframe_request", "reason": "startup" | "loss" | "late" | "decode_error" }
```

The agent forces an IDR through the same path PLI uses. It MAY coalesce requests: at least one IDR
follows one or more requests, as soon as the encoder allows, and the agent SHOULD produce it within
one frame interval.

### NACK (client → agent), and nack_miss (agent → client)

```json
{ "type": "nack", "frame_seq": 4711, "frags": [[3, 5], [9, 9]] }
{ "type": "nack_miss", "frame_seq": 4711, "frags": [[9, 9]] }
```

`frags` are inclusive `frag_index` ranges of one frame, at most **32 ranges** per message, each
inside `0 … frag_count − 1`; the agent ignores ranges past the 32nd and any index out of range.
`frame_seq` is compared with wrap-around arithmetic (§Channels). The agent resends each fragment it still
holds as a new datagram: same header and payload, a new `pkt_seq`, flag `retransmit` set. It
answers `nack_miss` for the ones it no longer holds. §Loss recovery says when the client sends
these.

### Feedback (client → agent)

Receiver feedback for congestion control, sent every 50 ms (20–100 ms) while datagrams arrive.

```json
{ "type": "feedback", "seq": 812,
  "first_pkt_seq": 120400, "count": 310,
  "ref_us": 7340219000,
  "arrivals_b64": "<count × u16>",
  "recv_kbps": 14200, "lost": 1, "interval_ms": 50 }
```

- `seq`: per-connection feedback counter, +1 per message.
- `first_pkt_seq`, `count`: the `pkt_seq` range this message reports, `count` at most 8192, with
  wrap-around arithmetic (§Channels); without gaps between consecutive messages (a range may be
  reported again; the agent uses the latest).
- `ref_us`: the client's monotonic clock, microseconds, that arrival offsets count from.
- `arrivals_b64`: base64 of `count` u16 values, one per `pkt_seq` in order: the arrival time in
  units of 50 µs after `ref_us`, or `0xFFFF` for not received (yet).
- `recv_kbps`, `lost`, `interval_ms`: the receive rate, datagrams lost, and the length of the
  interval the message covers, for the agent's logs and metrics.

The agent keeps the send time of every `pkt_seq` and estimates the available rate from the delay
and loss pattern, together with the QUIC connection's own congestion state. The estimate feeds the
session's existing ABR (bitrate, then rung, `smooth` mode); there is no second bitrate controller.
*(Amendment 55, accreleus/quasar#1050.)* Except while PyroWave is active: its per-session rate
controller then acts in place of the ABR (`codec-pyrowave.md` §Rate).
An agent may ignore `feedback`, and a client may omit it; the stream then follows QUIC congestion
control alone.

### Input on the control stream (client → agent)

```json
{ "type": "input", "event": { "t": "k", "code": 30, "pressed": false } }
```

`event` is one `input.md` event, verbatim. WebRTC input is reliable and ordered today
(`input.md`), so a client **SHOULD** send every press or release on this stream: `k`, `mb`, a `gp`
whose buttons changed, and any touch start or end event. Pointer motion (`mm`, `ma`), scroll
(`ms`), axis-only `gp` changes and other continuous input MAY go as datagrams (§Input), where a
lost sample is replaced by the next. The agent applies both paths to the same input stream, in
arrival order.

### Ping and pong (both directions)

Liveness, round-trip time and the clock offset. Either side sends `ping`; the other answers `pong`
at once.

```json
{ "type": "ping", "id": 17, "tc": 123456.7 }                                   // client → agent
{ "type": "pong", "id": 17, "tc": 123456.7, "ts": 998877.1, "tw": 1762400000123.4 } // agent → client
{ "type": "ping", "id": 4, "ts": 998880.0, "tw": 1762400003001.2 }              // agent → client
{ "type": "pong", "id": 4, "ts": 998880.0, "tw": 1762400003001.2, "tc": 123459.9 } // client → agent
```

`tc` is the client's monotonic clock in ms, `ts` the agent's monotonic clock in ms (both as in
`input.md` §Clock sync), and `tw` the agent's wall clock in Unix ms at the instant of `ts`, which
maps `capture_us`. A `pong` echoes the `ping`'s fields and adds its own clock. The client pings once
a second; the round trip is the RTT it shows. **`ping` and `pong` never travel on the input channel
on this transport** (unlike the WebRTC DataChannel), so `pong` has a return path.

### Bye and error (either direction)

```json
{ "type": "bye", "reason": "client_closed" | "taken_over" | "session_ended" }
{ "type": "error", "code": "auth_failed", "message": "…" }
```

`bye` then closing the WebTransport session is a clean teardown. `reason` and `code` are open
strings; `message` is for a person. WebTransport close codes mirror the signaling close codes:

| code | meaning |
|---|---|
| `0` | normal close |
| `4400` | protocol violation: no `hello` in 5 s, a control message over 64 KiB, malformed framing |
| `4401` | token refused |
| `4404` | session ended or stopping |
| `4410` | taken over by another client (terminal for this client: do not reconnect) |

The session's lifecycle is the control plane's and the agent's, as for WebRTC. The agent treats a
closed connection, or 10 s with nothing received on the control stream, as the client gone, and
applies the same rule as a lost WebRTC peer (the session ends when the host's idle window passes
with no client).

## Video (`0x01`, agent → client)

An encoded access unit, split into fragments that each fit one datagram.

```
offset size field
0      u8   channel     0x01
1      u32  pkt_seq
5      u8   flags       bit0 keyframe, bit1 discardable, bit2 retransmit; others 0
6      u8   config_id   the hello_ack configuration this frame decodes under, wraps
7      u16  frame_seq   per-session access-unit counter, +1 per unit in encode order, wraps
9      u16  frag_index  0-based
11     u16  frag_count  ≥ 1
13     u32  ts_90khz
17     u64  capture_us
25     ...  payload
```

The header is 25 bytes; at the 1200-byte UDP budget a fragment carries at least about 1123 bytes
(§Channels, "Datagram size").

- *(Amendment 55)* A PyroWave frame's payload is defined by `codec-pyrowave.md` §Video
  datagrams, not by the next rule.
- The payload is the access unit in the form `EncodedVideoChunk` takes: 4-byte length-prefixed NAL
  units for H.264 and HEVC (no Annex-B start codes), the low-overhead OBU format for AV1 (each OBU
  with its size field, no temporal delimiter). Fragments are byte ranges of it, in order.
- Every fragment of a frame carries the same `frame_seq`, `config_id`, `flags` (apart from
  `retransmit`), `frag_count`, `ts_90khz` and `capture_us`.
- `discardable`: nothing later references this frame, so losing it never needs a keyframe.

## Audio (`0x02`, agent → client)

One Opus packet per datagram, 10 ms (`hello_ack.audio`).

```
0      u8   channel     0x02
1      u32  pkt_seq
5      u16  packet_seq  +1 per audio packet, wraps
7      u32  ts_48khz
11     u64  capture_us
19     ...  Opus packet
```

Audio is never retransmitted; a lost packet is concealed (Opus PLC or silence). `capture_us` on
both channels is what keeps audio and video in sync.

## Audio redundancy (`0x06`, agent → client)

*(Amendment 67, accreleus/quasar#1346.)* Sent only when the client's `hello.audio_red` was `true`
(§Hello) and `hello_ack.audio.redundancy` names it (§Hello ack). The same layout as `0x02` with
channel byte `0x06`, carrying one audio packet a second time, so a lost or late packet is already
there with the next one:

```
0      u8   channel     0x06
1      u32  pkt_seq
5      u16  packet_seq
7      u32  ts_48khz
11     u64  capture_us
19     ...  Opus packet
```

`packet_seq`, `ts_48khz`, `capture_us` and the Opus packet are the repeated packet's, unchanged;
`pkt_seq` is the datagram's own, as for every agent → client datagram (§Channels). For each audio
packet N the agent sends, it sends one `0x06` datagram repeating packet N − 1 **immediately before**
N's `0x02` datagram. It sends none before the first packet, and none across a pause: nothing when
N − 1 began more than 100 ms before N did (`capture_us`).

The client treats a `0x06` datagram exactly as the audio packet it repeats, and drops a copy of a
packet it has already taken — so a copy that arrives after the datagram it repeats, or after the
packet has played, is dropped. A packet whose original was lost is played from its copy, with no
concealment. Audio is still never retransmitted on request.

**Cost.** Audio bitrate roughly doubles (a few hundred kbit/s at most). The copies are not counted
against the video bitrate ceiling: they are not in the estimate the ABR is fed (§Feedback).

## Input (`0x03`, client → agent)

```
0      u8   channel     0x03
1      ...  one input.md event: UTF-8 JSON, verbatim
```

One event per datagram, every `input.md` event except `ping` and `pong` (§Ping and pong). A lost
datagram is lost, so presses and releases SHOULD go on the control stream instead (§Input on the
control stream); datagrams are for continuous input.

## Microphone (`0x05`, client → agent)

One Opus packet per datagram, 10 ms, mono, 48 kHz (`hello_ack.mic`), only when the session was
granted a microphone.

```
0      u8   channel     0x05
1      u16  packet_seq  +1 per packet, wraps
3      u32  ts_48khz
7      ...  Opus packet
```

The agent plays it into the session's microphone sink exactly as it plays the WebRTC microphone
track (`agent-api.md` `session_assign.stream.mic`). Never retransmitted.

## Loss recovery

**Agent.** It keeps every video datagram it sent for at least `retransmit_window_ms` (at least
250 ms, and at least twice the smoothed RTT, within a memory bound it chooses) and answers `nack`
from that buffer.

**Client.**

1. **Detect.** A fragment is missing when a later fragment of the same frame, or a fragment of a
   later frame, has arrived and it has not.
2. **NACK.** Send one `nack` per frame as soon as fragments are missing (ranges may be batched),
   and at most one repeat per fragment, after one RTT without it.
3. **Late.** A frame is late when it cannot complete before the client needs it: a `nack_miss`
   covers one of its fragments, its repeat NACK went unanswered for one more RTT, or it is still
   incomplete when the client would present it. The client drops a late frame. If it was not
   `discardable` the client sends `keyframe_request {reason: "late"}` and drops every frame until
   the next keyframe.
4. **FEC first.** When the frame's FEC (§FEC) can rebuild the missing fragment, the client does
   that and sends no NACK for it. A repair datagram follows the fragments it protects, so a gap
   shows before the repair that covers it can arrive. *(Amendment 65, accreleus/quasar#1046,
   signed off 2026-11-10.)* While repair datagrams are arriving (at least one within the last
   second), a client MAY hold a fragment's first NACK (rule 2) until the earliest of: a repair
   rebuilds it; a fragment of a later frame arrives; or 8 ms after the fragment was found missing.
   The repeat NACK and the late rule (3) count from when the first NACK is sent. A fragment
   rebuilt before its NACK is never NACKed, and so is not in the client's count of fragments
   found missing (`control-api.md` stats, `packets_lost`).
5. **Reordering.** *(Amendment 73, accreleus/quasar#1330, proposed 2026-11-14.)* A client MAY
   measure how late reordered datagrams arrive (from a later `pkt_seq` arriving to an earlier one
   arriving) and, while recent measurements show the path reordering (not one late datagram),
   hold a gap that long before it reports the gap in feedback or sends its first NACK (rule 2),
   whether or not a fragment of a later frame has arrived. The hold is at most 150 ms, and no
   longer than `retransmit_window_ms` less one frame interval and half the round trip, so a held
   NACK still finds the fragment in the agent's buffer. The repeat NACK and the late rule (3) then
   each wait that much longer. A gap wider than reordering could open (more datagrams than arrive
   in the longest hold allowed, at the current rate) is an outage, and feedback reports it without
   this hold. A client that sees no reordering holds nothing beyond rule 4. Feedback still reports
   every `pkt_seq` before a held gap on the cadence of §Feedback; a message whose range would
   start at the held gap waits for it, so one may come up to the hold later than that cadence.

`keyframe_request {reason: "loss"}` remains for a client that does no NACK at all.

## FEC (`0x04`, agent → client)

Specified now so a client and agent can be built against it; the agent need not send it, and in
0.6 it is the last, cuttable unit. The agent sends FEC only when `hello_ack.fec` names a scheme.

**Scheme `xor`.** One repair datagram protects up to 48 fragments of one frame: fragments
`first_frag + i × stride` for `i` in `0 … count − 1`. Its payload is the XOR of those fragments'
payloads, each padded with zeros to the longest; it rebuilds any **one** of them that is missing.
`stride` 1 protects a run of fragments, a larger `stride` interleaves groups against a burst. The
agent chooses `count`, `stride` and how many repair datagrams to send from the loss it sees in
feedback. The agent SHOULD send a frame's repair datagrams right after that frame's video
datagrams, before any datagram of a later frame *(amendment 65)*.

```
0      u8   channel     0x04
1      u32  pkt_seq
5      u8   scheme      0x01 = xor
6      u16  frame_seq
8      u16  first_frag
10     u8   count       2 … 48
11     u8   stride      ≥ 1
12     u16  length_xor  XOR of the protected payloads' lengths
14     ...  repair payload
```

To rebuild fragment `k` when it alone of its group is missing: its payload is the repair payload
XOR every other protected payload (zero-padded), truncated to `length_xor` XOR every other
protected length; its header is a sibling fragment's with `frag_index = k`. A group with two or
more missing is left to NACK. Repair datagrams are never retransmitted.

## Versioning

- `wt_proto` is an increasing integer; this document is version 1.
- Inside a version, growth is additive only: new control-stream `type`s, new fields, new datagram
  channels and new FEC schemes, each ignored by a receiver that does not know them. A change to a
  binary header above needs a new `wt_proto`.
