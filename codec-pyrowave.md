# PyroWave codec — "Ultra-low latency (PyroWave)" (EXPERIMENTAL, FROZEN)

*(Amendment 55, accreleus/quasar#1050, spec accreleus/quasar#1048, spike accreleus/quasar#1049,
pre-approved by the owner 2026-11-06; independent Opus review before merge. Revised by
amendment 58, accreleus/quasar#1059, pre-approved by the owner 2026-11-07: PyroWave is chosen at
launch only, from the Launch options Codec list or by Auto on a `prefer` profile, and the
mid-session user toggle is withdrawn and deferred to 0.6.1.)*

The wire contract for PyroWave: an intra-only wavelet codec, encoded on the host GPU in Vulkan
compute and decoded by the client in WebGPU. It is the codec behind the user-facing
**Ultra-low latency (PyroWave)** entry in the Launch options Codec list. It rides **only** the WebTransport transport
(`transport-webtransport.md`, `wt_proto` 1) and never WebRTC, because its bitrate (about
200 Mbit/s and up) rules out WebRTC and the internet.

**Frozen** like every document here. **Additive**: a client or agent that does not implement this
amendment never sees PyroWave (§Who gets PyroWave), and nothing in `transport-webtransport.md`
changes for a session that does not use it.

Co-defined with `agent-api.md` (the host codec set, `session_assign.pyrowave`, the
`codec.switched` trace event), `control-api.md` (the launch profile's `pyrowave_mode`, the stream
plan rule with its launch-time Auto rule, the session's `pyrowave` object, the Launch options
entry) and `schema.md` (PyroWave's migration, numbered at landing, 0114 or above).

## The bitstream

PyroWave's bitstream **v1**, as specified in
[`bitstream/bitstream.md`](https://github.com/Themaister/pyrowave/blob/7fe0e2667cd3076b8b2bf04bcb80bc8d691b800b/bitstream/bitstream.md)
at upstream commit **`7fe0e266`** (`Themaister/pyrowave@7fe0e2667cd3076b8b2bf04bcb80bc8d691b800b`,
the bitstream frozen upstream since 2026-10-03). This document refers to it as *the bitstream
spec*. Terms used here:

- a **block**: one 32×32 coefficient block with its 8-byte `BitstreamHeader`, `payload_words × 4`
  bytes in all. A block is decodable on its own: no prediction, no context.
- the **SOF**: the 8-byte `BitstreamSequenceHeader` with `extended = 1` and
  `code = START_OF_FRAME`, one per frame.
- a **packet**: a block or the SOF. Packets are self-delimiting: a reader reads the first u32;
  if its bit 31 (`extended`) is set the packet is 8 bytes, otherwise it is `payload_words × 4`
  bytes (`payload_words`, bits 16–27, is at least 2).
- **`N`**: the number of block indices a frame of this configuration has, from the bitstream
  spec's "Block index ordering" for the session's width, height and chroma. Every frame's blocks
  have indices in `0 … N − 1`.

Quasar uses **YCbCr 4:2:0** (`chroma_resolution = 0`). 4:4:4 is a later amendment. A block whose
`ballot` is 0 is never sent, as the bitstream spec says, so an absent block is not by itself a
lost one (§Coverage).

**A bump of the upstream pin** is a contract change if the bitstream changes, and needs
regenerated golden fixtures in the quasar repository either way.

## Wire vocabulary

`pyrowave` is a **wire** codec string, beside `h264`, `h265` and `av1`. It appears in:

- the host and GPU codec sets (`agent-api.md` `capacity.codecs`, `capacity.gpus[].codecs`),
  only after a passing PyroWave codec probe on that GPU;
- `hello_ack.video.codec` (`transport-webtransport.md`) and the `codec.switched` and
  `caps.negotiated` trace events (`agent-api.md`);
- `Session.negotiated_codec`, when a client reports it is decoding PyroWave (`control-api.md`).

It is **not** a catalog codec: a stream profile (a rung) never names it, and `sessions.codec` /
`stream.codec` / `session_assign.stream.codec` never hold it. Those keep the session's
**standard codec**, the one the rung walk resolved (`h264 | h265 | av1`), which is what a WebRTC
fallback and a step-down use. PyroWave is a separate, per-session overlay
(`session_assign.pyrowave`, `Session.pyrowave`). It has one spelling everywhere, so there is no
rename to bridge: the control plane's one codec bridge (quasar
`control-plane/internal/session/codec.go`) gains the wire constant and never maps a catalog codec
to it, and the rung walk's clamps never select it.

## Who gets PyroWave

PyroWave is sent on a WebTransport connection only when **all** of these hold:

1. the control plane granted it at launch (`session_assign.pyrowave` present): the launch profile
   allows it, the session's transport is `webtransport`, the placed GPU's codec set lists
   `pyrowave`, and the launching device's fresh probe reports WebGPU (`control-api.md`
   §Ultra-low latency (PyroWave));
2. the session wants it: the user picked it at launch, or Auto picked it on a `prefer` profile
   (`session_assign.pyrowave.wanted`), and nothing since has turned it off (§Switching codec);
3. **this connection** can decode it: its `hello` carries `"pyrowave": true` (below);
4. the link qualifies, measured on this connection (§Link).

Otherwise the agent sends the standard codec, exactly as `transport-webtransport.md` describes. A
client that does not implement this amendment never sends `hello.pyrowave`, so it never receives
PyroWave and never sees a codec change.

## Control stream additions

### `hello` (client → agent)

One optional field:

```json
{ "type": "hello", "wt_proto": 1, "...": "...", "pyrowave": true }
```

`pyrowave: true` means this client, on this connection, can decode PyroWave: its WebGPU device is
up and its decoder pipelines are built. Absent or `false`: the agent never sends PyroWave on this
connection. The client decides its decoder path (`subgroups` + `shader-f16`, or the fallback) on
its own; the agent does not need to know it.

### `hello_ack` (agent → client)

For a PyroWave configuration, `video` is:

```json
"video": { "codec": "pyrowave", "description_b64": "<PWV1Header, 24 bytes>",
           "width": 1920, "height": 1080, "fps": 60,
           "color_space": { "primaries": "bt709", "transfer": "bt709",
                            "matrix": "bt709", "full_range": false } }
```

- `description_b64`: base64 of the bitstream spec's 24-byte `PWV1Header` ("Raw disk format
  specification"): `magic` `"PWV1"`, the `BitstreamSequenceHeader` with `sequence`, `extended`,
  `total_blocks` and `code` zero, `frame_rate_num` = `fps`, `frame_rate_den` = 1,
  `reference_bit_depth` 8, `header_version` 1. It lets the client build its decoder before any
  datagram arrives. Its width, height and chroma must equal every SOF of the configuration.
- `color_space` is **required** for PyroWave and agrees with the header's usability fields:

  | `color_space` | PyroWave header |
  |---|---|
  | `primaries: "bt709"` | `color_primaries = SRGB` (0) |
  | `transfer: "bt709"` | `transfer_function = SRGB` (0), the bitstream spec's SDR curve; for this codec `bt709` here means that curve (WebCodecs `iec61966-2-1`) by definition |
  | `matrix: "bt709"` | `ycbcr_transform = BT709` (0) |
  | `full_range: true` / `false` | `ycbcr_range = FULL` (0) / `LIMITED` (1) |

  PyroWave has no BT.601 matrix, so `smpte170m` is never used for it: the agent converts the
  compositor's RGB to YCbCr with BT.709 for every PyroWave configuration, whatever the standard
  codec signals. Only `full_range` may differ between PyroWave configurations. HDR (BT.2020, PQ) is a
  later amendment.
- `retransmit_window_ms` is present, as in every `hello_ack` (a re-sent one repeats every field),
  and keeps the standard codec's value: it applies only to standard-codec frames. PyroWave frames
  are never retransmitted (§Loss).
- `width`, `height`, `fps` as for any codec.

`hello_ack` also gains one optional top-level field, **`codec_switch_reason`**: present on a
`hello_ack` whose `video.codec` differs from the previous configuration's, or on the first
`hello_ack` when a wanted PyroWave was not started; the reason from §Switching codec. A client
shows it to the user ("Ultra-low latency is off: the link is too slow").

### `codec_fallback` (client → agent)

```json
{ "type": "codec_fallback", "reason": "decode_error" }
```

The client cannot go on decoding PyroWave (its WebGPU device was lost, a decode failed). The agent
switches this connection to the standard codec (§Switching codec, reason `client_decode_error`).
It only ever moves a session **off** PyroWave, onto the codec the control plane already chose, so
it grants nothing. `reason` is an open string, `^[a-z0-9_]{1,64}$`, for logs. A client sends it at
most once per configuration.

## Video datagrams

PyroWave frames use the video channel (`0x01`) and its 25-byte header unchanged
(`transport-webtransport.md` §Video). For a PyroWave frame:

| header field | value |
|---|---|
| `flags` | `keyframe` and `discardable` both set on every frame: every frame decodes alone, and none is referenced |
| `config_id` | the PyroWave configuration's |
| `frame_seq` | +1 per frame, as for any codec |
| `frag_index`, `frag_count` | this datagram's index in the frame, and the number of datagrams the frame was sent in |
| `ts_90khz`, `capture_us` | the frame's, the same in every datagram of it |

The **payload** is not a byte range of an access unit (the rule for the other codecs). It is a
12-byte **PyroWave header**, then packets:

```
offset size field
0      u8   kind         0 = packets, 1 = part of one oversized block
1      u8   part_index   kind 1: this part's index, 0-based; kind 0: 0
2      u8   part_count   kind 1: the block's number of parts, ≥ 2; kind 0: 0
3      u8   reserved     0
4      u32  first_block  coverage start (§Coverage)
8      u32  end_block    coverage end, exclusive (§Coverage)
12     ...  kind 0: one or more whole packets; kind 1: a byte range of one block
```

All fields little-endian. The header keeps every packet at a 4-byte offset from the start of the
payload, as the bitstream's u32 words want.

### Packing

The agent sends a frame's packets in bitstream order (the SOF first, then blocks by increasing
`block_index`), filling datagrams greedily, which is upstream's `pyrowave_encoder_packetize` rule
with the boundary set to the datagram's payload room after both headers:

1. A packet that fits in the room left in the current datagram goes there.
2. A packet that does not fit there but fits an empty datagram starts the next datagram. **A packet
   is never split when it fits one datagram.**
3. A block larger than an empty datagram's room is an **oversized block**. It is sent alone, as
   `kind = 1` parts: `part_count` consecutive datagrams of the frame, `frag_index` `j … j +
   part_count − 1`, `part_index = frag_index − j`, each carrying the next byte range of the block;
   every part but the last is filled. Then packing continues with a new datagram.

The SOF is always the first packet of `frag_index` 0. The room comes from the datagram size rule
of `transport-webtransport.md` (1200-byte UDP payload, or more only on a direct path with
path-MTU discovery done), so at the 1200-byte budget a datagram carries at least about 1111 bytes
of packets.

**Why oversized blocks are split** (B00, accreleus/quasar#1049): a single block reached 1432
bytes in the spike's streams, and 0.03–0.4% of blocks exceed 1200 bytes. Even a 1500-byte MTU
path leaves only about 1380 bytes of room after the QUIC, WebTransport, video and PyroWave
headers, and a path through an edge stays at 1200, so "never split a block" cannot hold. The
encoder cannot cap a block's size: at the pin, PyroWave's rate control bounds only the whole
frame (`pyrowave_rate_control.maximum_bitstream_size`), and `payload_words` allows blocks up to
16 KiB. Sending such blocks on a reliable stream would put them behind input and control on the
one control stream (or need a second stream `transport-webtransport.md` does not define), and a
retransmitted block lands after a 16.7 ms frame's deadline anyway. Splitting only oversized blocks,
all-or-nothing, keeps every other block one datagram away from decode, and costs only the rare
block that needs it: losing any one part loses that block, which is concealed like any lost block.

### Coverage

Each datagram states the **range of block indices it is responsible for**, `[first_block,
end_block)`, so a receiver knows exactly which blocks a lost datagram took with it:

- The ranges of a frame's datagrams, in `frag_index` order, are contiguous and partition
  `[0, N)`: datagram 0 starts at 0, each datagram's `first_block` is the previous one's
  `end_block`, and the last one ends at `N`. The one empty range is a datagram 0 that holds only
  the SOF because block 0 is oversized: it carries `[0, 0)`.
- A `kind = 0` datagram's range covers the blocks it carries **and the zero blocks** (never sent)
  between them and up to the next datagram's first block. Every block it carries has an index in
  its range.
- Every part of an oversized block carries the same range, starting at that block's index.

A block index is then, for a decoded frame:

- **received**: its block arrived (an oversized block: every part arrived and they reassembled);
- **zero**: a received datagram (for an oversized block, any one of its parts) covers it and it is
  not a block that datagram carries — it was all zero, as the bitstream spec says for an absent
  block;
- **lost**: no received datagram covers it, or it is an oversized block with a part missing.

### Receiving

- **Validation.** The client drops a datagram:
  - whose `kind` it does not know;
  - whose range is empty or ends past `N`, **except** `frag_index` 0 holding only the SOF, whose
    range is `[0, 0)` when block 0 is itself oversized and goes in the next datagrams;
  - whose packets overrun the payload, or that carries a block whose `payload_words` is below 2;
  - that carries a block with an index outside its range;
  - of `kind = 1` with `part_index` ≥ `part_count`, or whose range does not start at its own
    block's index (in the part with `part_index` 0, the header's `block_index`).

  It drops an oversized block whose parts disagree on `part_count`, range or
  `frag_index − part_index` (the parts of one block are one consecutive run), or whose reassembled
  length is not its header's `payload_words × 4`. A dropped datagram counts as lost: its range is
  concealed (a dropped part: its block).
- **Ignored bitstream fields.** The client orders frames by `frame_seq` and ignores the bitstream's
  3-bit `sequence`. `total_blocks` in the SOF is informational; `frag_count` says when a frame is
  complete. The SOF itself may be lost: the configuration comes from `hello_ack`.
- **When to decode.** The client decodes a frame when all `frag_count` datagrams have arrived, or
  at its presentation deadline with whatever arrived; it SHOULD set that deadline no later than
  one frame interval after the frame's first datagram arrived. A frame whose `frame_seq` is not
  after the last frame decoded is dropped. (B00: the client SHOULD write each block into the GPU
  upload buffer as its datagram arrives, rather than walking a finished frame.)
- **FEC.** `transport-webtransport.md` §FEC applies unchanged: a rebuilt datagram is a datagram.

## Loss: block concealment

A **lost** block is concealed: the client SHOULD use that block index's coefficients from the
most recent frame decoded under the same `config_id` that had it received, and otherwise decodes
it as zero (the bitstream spec's rule for a missing block). A **zero** block is zero, never
concealed. Loss therefore shows as a brief local smear in the lost blocks, never a freeze.

PyroWave uses **no NACK and no keyframe request**:

- the client sends no `nack` and no `keyframe_request` for a PyroWave frame. An agent ignores a
  `nack` for one (no retransmission, no `nack_miss`), and need not keep PyroWave datagrams for
  retransmission;
- every frame is intra, so the next frame repairs everything a loss touched; a retransmission
  would only add a wait before a deadline the codec exists to remove.

`transport-webtransport.md` §Loss recovery is unchanged for the standard codec. Receiver
`feedback` is sent as for any codec: it is what the rate follows.

## Rate

PyroWave's rate control is exact per frame: the agent gives the encoder a byte budget for each
frame (`maximum_bitstream_size`), and the frame lands within it.

- **Target.** `session_assign.pyrowave.target_kbps`, computed once at launch by the control
  plane:

  ```
  target_kbps = min(floor(height_default × fps / 60), pyrowave_kbps ?? 1 000 000)
  height_default = 200 000 if launch height ≤ 1080, 300 000 if ≤ 1440, else 400 000   (kbit/s)
  ```

  *Launch height* is the session's **encoded** stream height at launch (`stream.height` as the
  launch resolved it: the rung's, or a `stream.*` override's), not the client's display or the
  render size; `fps` is the session's launch fps. The default is a per-frame budget taken at a 60 fps reference (B00: 45.9 dB at
  1080p/200, 45.7 dB at 1440p/300, 42.7 dB at 4K/400 on worst-case content), so the per-frame
  quality holds at any frame rate. The launch profile's `pyrowave_kbps` is a **maximum**, not a
  fixed target: it caps the default, and `null` caps it at 1 000 000.
- **Clamp.** Each frame's budget is `min(target, estimate) / fps`, where *estimate* is the
  available-rate estimate of `transport-webtransport.md` §Feedback (from `feedback`, and the QUIC
  connection's congestion state when the client sends none), less the agent's datagram overhead.
  While PyroWave is active the session's ABR governor and its resolution/fps ladder do not act;
  they resume from the estimate after a switch back. A live `stream_width`/`stream_height`
  change (`agent-api.md` `session_display_update`) still applies: it is a new configuration, and
  the target is unchanged.
- **Seeding.** When PyroWave starts on a connection and at every switch on, the agent **seeds its
  estimate at `target_kbps`**, whatever the estimate was under the standard codec (a 20 Mbit/s
  H.264 stream says nothing about a 200 Mbit/s link). From there it drops below the target only
  on loss or delay signals (`feedback`, or QUIC's congestion state), recovers as they clear, and
  never exceeds the target.
- **One controller per codec.** `transport-webtransport.md` §Feedback says the estimate feeds the
  session's ABR and there is no second bitrate controller. For a PyroWave session that rule is
  overridden: while PyroWave is active its per-session rate controller (this section) is the one
  that acts, and the ABR governor is idle. For the standard codec, and on every session without
  PyroWave, amendment 54's rule stands unchanged.
- **Pacing.** The agent SHOULD pace a frame's datagrams over no more than half a frame interval
  (a 1080p/200 frame is about 380 datagrams), not burst them.
- **Step-down.** When the estimate stays below **`floor_kbps`** (`session_assign.pyrowave`, half
  the target) for **1 s**, the agent switches to the standard codec, reason `congestion`
  (§Switching codec). The 1 s timer starts no earlier than the **first `feedback` received after
  the switch on** (for a client that sends no `feedback`: 1 s after the switch on), so a switch
  on is judged on what the link does at PyroWave's rate, never on the estimate it replaced.

## Link

The link that matters is client ↔ host, and it exists only once the WebTransport connection does,
so the agent measures it there:

- **Round trip.** The agent starts PyroWave on a connection only when the QUIC connection's smoothed RTT (from the handshake at `hello`, QUIC's own
  estimate after) is at most **`max_rtt_ms`** (`session_assign.pyrowave`, 5 ms). This is what
  keeps PyroWave on the LAN: over the internet the gate fails, and the session streams the
  standard codec with reason `rtt_too_high`.
- **Rate.** There is no up-front throughput test: the session starts at the target, and the floor
  rule (§Rate) steps it down within a second if the link cannot hold it.

The device probe's `bandwidth_kbps` and `rtt_ms` (`control-api.md` `POST /v1/me/devices`) are
**not** inputs to this gate: they measure the client ↔ control plane path, from a sample of a few
hundred KB, so they say little about a host on the client's LAN or about 200 Mbit/s. *(Amendment
58: they are inputs to one launch-time guess, the control plane's Auto rule on a `prefer`
profile, which only decides whether the session wants PyroWave; this gate and the step-down still
decide whether it gets it, `control-api.md` §Ultra-low latency (PyroWave) ▸ The stream plan.)*
The probe's other input to PyroWave is WebGPU.

## Switching codec

A WebTransport session can now change codec mid-session, between PyroWave and its standard codec.
(`transport-webtransport.md` says the transport never renegotiates the codec; amendment 55 makes
this one exception, and only for a client that sent `hello.pyrowave: true`.)

**Signalling.** A switch is a new configuration: the agent sends `hello_ack` again with the new
`video`, `config_id` + 1 (mod 256) and `codec_switch_reason`, **before** the first frame under it,
exactly as `transport-webtransport.md` §Hello ack does for a resize. The first standard-codec frame
after a switch is a keyframe; every PyroWave frame is. The client holds a frame whose `config_id`
it has no `hello_ack` for (at most 1 s) and drops frames of an older `config_id`, as before, so it
never decodes a frame with the wrong decoder. It keeps both decoders ready while the session is
PyroWave-eligible.

**Who switches, and why** (`codec_switch_reason`, and the `codec.switched` trace event's
`reason`; open strings):

| reason | switch | trigger |
|---|---|---|
| `launch` | on to PyroWave | the first `hello` of a session that wants PyroWave (picked by the user or by Auto) passes §Link |
| `reconnect` | on to PyroWave | a new WebTransport connection (reload, takeover) of a session that still wants PyroWave passes §Link |
| `rtt_too_high` | off, or never on | §Link failed at `hello` or at a switch on |
| `client_no_decoder` | off, or never on | the session wants PyroWave but this connection's `hello` did not offer it (a takeover from another browser) |
| `congestion` | off | §Rate step-down |
| `client_decode_error` | off | the client's `codec_fallback` |
| `encoder_error` | off | the PyroWave encoder failed on the host |
| `webrtc` | off | the media client is now a WebRTC peer (a fallback or a takeover), which cannot carry PyroWave; the agent encodes the standard codec for it |

**Wanting PyroWave.** The agent keeps a per-session *wanted* flag, set from
`session_assign.pyrowave.wanted` at launch. A switch off for `rtt_too_high`, `congestion`,
`client_decode_error` or `encoder_error` **clears** it, so the session does not flap back on, and
nothing sets it again during the session. `webrtc` does not clear it: a client that comes back on
WebTransport gets PyroWave again (reason `reconnect`).

**Mid-session switching on is deferred to 0.6.1** *(amendment 58)*. Amendment 55's `user` reason,
the user's toggle (`PATCH /v1/sessions/{id}/pyrowave` → `session_pyrowave`), is withdrawn: 0.6
decides PyroWave at launch, and a 0.6.1 amendment adds switching in and out of it during a
session, driven by the client → encoder telemetry. The switches above are unchanged.

## Versioning

Everything here lives inside `wt_proto` 1 under that document's additive rule: an optional `hello`
field, an optional `hello_ack` field, a new control message, and a codec-specific payload for a
codec only a client that asked for it receives. The 25-byte video header is unchanged. A change to
the PyroWave header (§Video datagrams) needs a new `kind` value (a receiver drops a `kind` it does
not know) or a new `wt_proto`.
