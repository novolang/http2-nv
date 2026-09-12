# http2-nv

HTTP/2 for novo-lang, as a value: the frames, HPACK, the stream state
machine and the flow-control windows, with a connection the host pumps
and nothing that touches a socket.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A port of Rust's [`h2`](https://docs.rs/h2) — hyper's HTTP/2 half — cut
to the layer design.  RFC 9113 is the protocol and RFC 7541 is HPACK,
and this package is both of them with the network taken out: bytes and
values go in, events and requested actions come out, and the host is the
only party that performs anything.

| module | holds | rows |
| --- | --- | --- |
| `h2err` | the thirteen error codes, the decode and encode faults, and § 5.4's connection-or-stream verdict | `[]` |
| `h2frame` | the nine-byte header and the ten frame types, to and from a caller's bytes | `[]` |
| `h2hpack` | the static table, the Huffman code, the prefix integer, and a dynamic table the caller sizes | `[]` |
| `h2settings` | the six parameters, what they refuse, and § 6.9.2's retroactive window delta | `[]` |
| `h2stream` | the seven states, the nine moves, and the identifier rules | `[]` |
| `h2flow` | two windows, a negative one, and the credit a receiver owes | `[]` |
| `h2prelude` | ALPN, prior knowledge and the h2c upgrade, as values | `[]` |
| `h2conn` | **`H2Action`** — the connection the host pumps | `[]` |

## The load-bearing interface

```novo norun:pseudo
pub enum H2Action
    H2DoSend(data: [u8])
    H2DoCloseConnection(code: H2ErrorCode, last_stream: Int)
    H2DoDropStream(stream: Int)
    H2DoExpectSettingsAck(within_ms: Int)

pub struct H2Step
    conn: H2Conn
    events: [H2Event]
    actions: [H2Action]
```

**`H2Action` is the package**, and its smallness is the claim.  A `core`
package's budget is `[]` and there is no escape from it, so every call
here answers what the caller must perform and the caller is the only
party that performs: four things, and everything else HTTP/2 might have
wanted to do is arithmetic the connection does itself.

Events and actions are **two lists and not one**, because they go to two
different places — the events up to the application, the actions down to
the socket — and a single interleaved list would have every host writing
the same partition.

**One feed produces many events**, which is where this differs from
[`http-codec-nv`](https://registry.novo-lang.org/http-codec-nv)'s
one-event-per-call reader.  A single TCP segment routinely carries
frames for six streams; HTTP/1.1 has one message in flight and can
afford the simpler shape, and HTTP/2 cannot.

## The one example that will work

```novo
use h2conn
use h2hpack
use h2prelude
use h2settings

fn main() [io]
    // A client that will speak HTTP/2 because ALPN said so.
    let c = h2conn.client(H2ViaAlpn, h2conn.default_limits(),
                          h2settings.default_settings())

    // The preface and this endpoint's SETTINGS, as bytes to write.
    let opening = h2conn.start(c)

    // A stream, and a request head on it.
    match h2conn.open_stream(opening.conn)
        Err(e) => println(e.message())
        Ok(o)  =>
            match h2conn.send_headers(o.conn, o.stream,
                                      [h2hpack.header(":method", "POST"),
                                       h2hpack.header(":scheme", "https"),
                                       h2hpack.header(":path", "/greeter/hello"),
                                       h2hpack.header(":authority", "example.test")],
                                      false)
                Err(e)   => println(e.message())
                Ok(step) => println("${h2conn.send_window(step.conn, o.stream)}")
```

## Adding it, and checking it

```console
$ novo pkg add http2-nv
$ novo pkg build
$ novo test tests/h2conn_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: http2-nv.<module>.<fn>`.  That is
what an interface release looks like from the outside, and it is how the
first implementation will know it is finished.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The layer, and why

`core`, and every row in the package is `[]` — there is not one
effect-polymorphic function either, which is unusual enough on this grid
to be worth a sentence.  HTTP/2's whole difficulty is state, and state
is a value: a connection takes the bytes a host read and answers what it
learned and what the host now owes.  There is no stream to take through
a trait, because the caller's stream never enters this package.  It
enters as `[u8]`.

### The device claim, and where it stops

`tests/embedded_probe.nv` builds for `--target=nrf52-qemu`, and what it
builds is the frame layer's arithmetic, HPACK's prefix integer and
static-table lookups, and the whole of flow control.  The consumer is
real: a sensor reporting over HTTP/2 to a gateway holds one connection
and one stream, reads a nine-byte header, decides whether the frame is
for it, subtracts a length from a window, and decides whether it may
send.  None of that needs a heap, and those functions carry
`@tier(embedded)`.

**`h2conn` does not carry it, and neither does HPACK's decoder.**  A
connection holds a list of streams, a pending buffer and two dynamic
tables; a dynamic table means eviction, which means storage the caller
did not size.  A device drives the frame layer directly and keeps its
one stream's window in two integers — which is what the probe is — and
this package does not claim more than that.

## What grpc-nv must change

The point of this row is that gRPC runs, and it was named as the missing
one by
[`grpc-codec-nv`](https://registry.novo-lang.org/grpc-codec-nv)'s README
and by grpc-nv's `grpctrans` module.  **This package cannot implement
`GrpcTransport[e]` itself**: that trait is declared in grpc-nv, which is
`host`, and a `core` package may not depend on a `host` one.  So the
impl belongs in grpc-nv, over this package's connection value, and here
is the whole of it — six methods, one call each.

| `GrpcTransport[e]` method | over http2-nv |
| --- | --- |
| `grpc_open` | `h2conn.open_stream` → `H2Opened.stream` |
| `grpc_send_headers` | `h2conn.send_headers`, with `GrpcMetadata` mapped to `[H2Header]` |
| `grpc_send_data` | `h2conn.send_data` → `H2Sent.taken`, which is already the "how much it took" that method answers |
| `grpc_send_trailers` | `h2conn.send_trailers` |
| `grpc_receive` | the next `H2Event` from the last `h2conn.feed`, or `GrpcWireIdle` when there is none |
| `grpc_reset` | `h2conn.reset_stream` |
| `grpc_is_open` | `h2conn.is_open` |

Four changes, and none of them is to the trait's shape:

1. **grpc-nv adds `http2-nv` as a dependency** and writes one impl over
   `H2Conn`.  The effects in `e` are the host's — `[io, net]` over a
   socket — and nothing in this package contributes any.
2. **Four constants move.**  grpc-nv declares `H2_NO_ERROR`,
   `H2_CANCEL`, `H2_INTERNAL_ERROR`, `H2_REFUSED_STREAM` and `ALPN_H2`
   because it had no package to take them from; they are
   `h2err.H2ErrorCode` and `h2prelude.H2_ALPN_ID` here.  Two
   declarations of the same name in one assembly is the hazard
   [`docs/publishing.md` § Public type names are globally
   unique](https://novo-lang.org/docs/publishing) describes, so the
   duplicate is grpc-nv's to drop at its next version.
3. **`never_processed` gets an answer instead of an assumption.**
   grpc-nv answers it today from a fault type it invents;
   `h2conn.never_processed` answers it from the GOAWAY the connection
   actually received and the RST_STREAM code the peer actually sent, so
   the safe-to-retry cases and the retryable status agree by
   construction rather than by a table somebody keeps in step.
4. **`GrpcWireWindowStalled(waited_ms)` has no counterpart here and
   should not gain one.**  A stalled stream is `h2conn.send_window` of
   0, which is an ordinary condition and not a fault; "for longer than
   the caller allows" needs a clock, and the clock is grpc-nv's.  The
   variant stays, and what fills it is grpc-nv's own timer rather than
   anything this package reports.

`grpc-codec-nv` needs no change at all.  Its README lists six things a
host must provide and this package provides all six; HPACK never appears
in its API, and it never appears in the table above either — the codec
hands over `GrpcMetadata` and takes it back, and how those pairs are
compressed is this package's business entirely.

## What is NOT here, and why

- **No scheduler.**  Which stream gets the next byte of a scarce
  connection window is a policy question with no protocol answer:
  `h2flow.sendable` says how much may go and the host decides for whom.
  A `core` package that chose would be choosing for a proxy, a browser
  and a device at once.
- **No priority.**  RFC 9113 § 5.3.1 deprecates RFC 7540's dependency
  tree, and § 5.3 lets an endpoint ignore it.  The five bytes are parsed
  so a relay can forward them and a capture can be read, and there is no
  scheduler to feed them to.
- **No `http-codec-nv` dependency, in either direction.**  That package
  is HTTP/1.1.  Its `H1Header` is a field line — a name whose case is
  insignificant, a value, and no indexing decision — and an HTTP/2
  header name with a capital in it is a malformed message (§ 8.2.1)
  while every header here carries HPACK's never-indexed bit.  Sharing
  the type would mean either an HTTP/1.1 type growing an HPACK field or
  this package losing the one it needs; the conversion between them is
  three lines in whichever host bridges the two protocols.
- **No TLS, no ALPN negotiation, no socket.**  `h2prelude` publishes the
  identifier to offer and the preface to send; offering it is
  `std.tls`'s and the host's.
- **No server push beyond the frames.**  PUSH_PROMISE is parsed,
  written and state-machined, because a client that receives one after
  announcing `SETTINGS_ENABLE_PUSH` of 0 is entitled to a connection
  error and has to be able to tell.  Deciding what to push is an
  application's.

## What widened, and what did not

- **Nothing widened.**  Every row in the package is `[]` and the
  `effect-budget` audit row measures 140 public functions against the
  `core` budget with none over it.  That is the useful result of writing
  the design as a value rather than as a callback.
- **`H2Conn` could not claim `@tier(embedded)` and the README says so
  rather than the claim quietly shrinking.**  A connection allocates by
  construction; the probe builds what a device can hold and nothing
  more.
- **`feed`'s error arm carries one value and the answer to a fault is
  two**, so `fault_actions` is published beside it.  A `Result` whose
  error arm could carry the frames to write would have made it one call,
  and a host that skipped it and closed the socket leaves the peer to
  time out.
- **A tuple in a `Result` was avoided four times.**  `H2VarInt`,
  `H2VarStr`, `H2Encoded` and `H2Decoded` each exist because a decoder
  answers a value AND a new offset or a new table, and the two halves go
  to different places in the caller.  Named structs read better at the
  call site, and they keep a `@value` struct out of a `Result` payload,
  which the language refuses.

## The reference implementation

[`h2`](https://docs.rs/h2) is the port's reference, with
[`nghttp2`](https://nghttp2.org/) as the second opinion on the frame
layer.  The specifications are
[RFC 9113](https://www.rfc-editor.org/rfc/rfc9113) (HTTP/2),
[RFC 7541](https://www.rfc-editor.org/rfc/rfc7541) (HPACK) and
[RFC 7540 § 3.2](https://www.rfc-editor.org/rfc/rfc7540#section-3.2)
(the h2c upgrade, which RFC 9113 removed and which is still deployed
everywhere).  The test vectors are RFC 7541's appendix C for HPACK and
appendices A and B for the static table and the Huffman code.

## Licence

Apache-2.0.
