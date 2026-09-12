# Changelog

All notable changes to http2-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
- **What grpc-nv changes to sit on top** is a table in the README: six
  methods, one call each, four constants that move here, and a
  `never_processed` that is answered rather than assumed.
- **No `http-codec-nv` dependency.** HTTP/1.1's header type has none of
  HPACK's shapes, and the README argues it.
- **Five quiet mistakes are tests**: the same flag bit means two things
  on two frame types; only DATA is flow controlled, and WINDOW_UPDATE
  must not be or nothing could unstall; a flow-control window may be
  negative and must not be clamped; credit is owed on delivery and not
  on arrival; and an HPACK entry larger than the whole table empties it
  rather than growing it.
