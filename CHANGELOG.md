# Changelog

All notable changes to http2-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `h2conn` — `H2Action`, the load-bearing type: a connection is a value
  the host pumps, and every call answers the events it learned and the
  four kinds of action the host must perform.
- `h2frame` — the nine-byte header and the ten frame types, to and from
  a caller's bytes, with an unknown type ignored and counted.
- `h2hpack` — the static table and the Huffman code as data, the prefix
  integer both ways, a dynamic table the caller sizes, and the
  never-indexed bit as a field on the header rather than a flag on the
  encoder.
- `h2stream` — the seven states, the nine moves, the flight-in-progress
  rule after a reset, and the identifier parity and ordering rules.
- `h2flow` — two windows and the minimum of three ceilings, a window
  that may be negative, and the credit a receiver owes on delivery
  rather than on arrival.
- `h2settings` — the six parameters, the values that are refused, the
  identifier that is ignored, and § 6.9.2's retroactive window delta.
- `h2prelude` — ALPN, prior knowledge and the h2c upgrade as values,
  with the upgrade marked as the one RFC 9113 removed.
- `h2err` — the thirteen error codes and § 5.4's connection-or-stream
  verdict as a function rather than as prose.
- `tests/embedded_probe.nv` — the device claim, built for
  `--target=nrf52-qemu`: the frame arithmetic, the HPACK lookups and the
  whole of flow control.

### Notes

- **Every row is `[]`**, with no effect-polymorphic function anywhere.
  The caller's stream never enters this package; it enters as `[u8]`.
- **The device claim stops at the connection.** `h2conn` and HPACK's
  decoder allocate by construction, and the README says so rather than
  letting the claim shrink quietly.
- **What grpc-nv changes to sit on top** is a table in the design notes below: six
  methods, one call each, four constants that move here, and a
  `never_processed` that is answered rather than assumed.
- **No `http-codec-nv` dependency.** HTTP/1.1's header type has none of
  HPACK's shapes; the design notes below say why.
- **Five quiet mistakes are tests**: the same flag bit means two things
  on two frame types; only DATA is flow controlled, and WINDOW_UPDATE
  must not be or nothing could unstall; a flow-control window may be
  negative and must not be clamped; credit is owed on delivery and not
  on arrival; and an HPACK entry larger than the whole table empties it
  rather than growing it.

### Design notes

**What grpc-nv changes to sit on top.** This package cannot implement
`GrpcTransport[e]` itself: that trait is declared in grpc-nv, which is a
`host` package, and a `core` package may not depend on one. The impl
belongs in grpc-nv, over this package's connection value, and it is six
methods with one call each.

| `GrpcTransport[e]` method | over http2-nv |
| --- | --- |
| `grpc_open` | `h2conn.open_stream` → `H2Opened.stream` |
| `grpc_send_headers` | `h2conn.send_headers`, with `GrpcMetadata` mapped to `[H2Header]` |
| `grpc_send_data` | `h2conn.send_data` → `H2Sent.taken` |
| `grpc_send_trailers` | `h2conn.send_trailers` |
| `grpc_receive` | the next `H2Event` from the last `h2conn.feed`, or `GrpcWireIdle` |
| `grpc_reset` | `h2conn.reset_stream` |
| `grpc_is_open` | `h2conn.is_open` |

Four things follow. grpc-nv adds `http2-nv` as a dependency and writes
one impl over `H2Conn`; the effects in `e` are the host's and nothing
here contributes any. Four constants move: grpc-nv declares
`H2_NO_ERROR`, `H2_CANCEL`, `H2_INTERNAL_ERROR`, `H2_REFUSED_STREAM` and
`ALPN_H2` because it had no package to take them from, and they are
`h2err.H2ErrorCode` and `h2prelude.H2_ALPN_ID` here.
`never_processed` gets an answer instead of an assumption, from the
GOAWAY the connection actually received and the RST_STREAM code the peer
actually sent. And `GrpcWireWindowStalled(waited_ms)` has no counterpart
here and should not gain one: a stalled stream is `send_window` of 0,
which is ordinary, and "for longer than the caller allows" needs a clock
that is grpc-nv's. grpc-codec-nv needs no change at all.

**No http-codec-nv dependency, in either direction.** Its `H1Header` is
a field line: a name whose case is insignificant, a value, and no
indexing decision. An HTTP/2 header name with a capital in it is a
malformed message (RFC 9113 § 8.2.1), and every header here carries
HPACK's never-indexed bit, which is a security property of the value
that must survive a relay. Sharing the type would mean either an
HTTP/1.1 type growing an HPACK field or this package losing the one it
needs.

**Four named structs instead of tuples in a `Result`.** `H2VarInt`,
`H2VarStr`, `H2Encoded` and `H2Decoded` each exist because a decoder
answers a value and a new offset or a new table, and the two halves go
to different places in the caller. Named structs read better at the call
site, and they keep a `@value` struct out of a `Result` payload, which
the language refuses.

**`fault_actions` is published beside `feed`** because a `Result`'s
error arm carries one value and the answer to a fault is two: the frames
to write, and a connection that has recorded which.
