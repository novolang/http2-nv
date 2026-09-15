# http2-nv

HTTP/2 is a binary framing of the same semantics HTTP/1.1 carries as
text, with many requests in flight on one connection at once. It is
specified in [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113), and its
header compression, HPACK, in
[RFC 7541](https://www.rfc-editor.org/rfc/rfc7541). This package brings
both to novo-lang as a connection value a host drives, with no socket
underneath. Two other packages on the registry run on it:
[grpc-codec-nv](https://novo-lang.org/packages/grpc-codec-nv) and
[grpc-nv](https://novo-lang.org/packages/grpc-nv), because gRPC is
defined over HTTP/2 and over nothing else. It depends on
[base64-nv](https://novo-lang.org/packages/base64-nv) for one header
value. [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) is
the same semantics over the HTTP/1.1 wire format.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What HTTP/2 is

One HTTP/2 connection carries many **streams**, and a stream carries one
request and its response. Everything on the connection is a **frame**: a
nine-byte header of a 24-bit length, an 8-bit type, an 8-bit flag set
and a 31-bit stream identifier, then a payload the type decides the
shape of (RFC 9113 section 4.1). Frames for different streams are
interleaved freely, which is what lets one connection carry a hundred
requests without a queue behind the first slow one.

| Type | Name | What it carries |
| --- | --- | --- |
| 0x00 | DATA | A message body, in pieces |
| 0x01 | HEADERS | The start of a message, as a compressed header block |
| 0x02 | PRIORITY | A stream dependency. Deprecated by section 5.3.1 |
| 0x03 | RST_STREAM | One stream, ended |
| 0x04 | SETTINGS | Connection parameters, and the acknowledgement of a set |
| 0x05 | PUSH_PROMISE | A server announcing a stream it is about to open |
| 0x06 | PING | Eight opaque bytes the peer echoes back |
| 0x07 | GOAWAY | The connection ending, with the last stream the sender processed |
| 0x08 | WINDOW_UPDATE | Flow-control credit, for one stream or for the connection |
| 0x09 | CONTINUATION | More of the header block the frame before it began |
| other | — | Ignored and discarded, which is the protocol's extension point |

A **header block** is a header list compressed with **HPACK**. HPACK
holds two tables. The **static table** is 61 entries both peers know
before the connection opens, and the **dynamic table** is entries the
peers have added since. A header is put on the wire as an index into
those tables, or as a literal that may be added to the dynamic table.
The tables therefore make a connection stateful: a decoder that skips a
block, or decodes half of one, has a table that no longer matches the
encoder's, and every later header on the connection decodes to something
nobody sent.

**Flow control** is a credit scheme. A **window** is a signed count of
bytes a sender may put on the wire, a WINDOW_UPDATE frame adds to it,
and a DATA frame subtracts from it. There are two windows on every byte,
one for the connection and one for the stream, and a DATA frame counts
against both (RFC 9113 section 5.2.1). Only DATA is flow controlled.

A stream moves through seven states as frames are sent and received
(section 5.1). The machine is asymmetric: the two half-closed states are
not one state.

| State | Meaning |
| --- | --- |
| idle | Nothing has happened on this identifier yet |
| reserved (local) | This endpoint promised the stream with a PUSH_PROMISE |
| reserved (remote) | The peer promised it |
| open | Both directions are live |
| half-closed (local) | This endpoint has finished sending and may still receive |
| half-closed (remote) | The peer has finished sending and this endpoint may still send |
| closed | Over in both directions |

**SETTINGS are not a handshake.** Each peer announces what it will
accept and the other obeys, so there are always two values in play for
every parameter and they are not equal.

| Parameter | Default | What it governs |
| --- | --- | --- |
| SETTINGS_HEADER_TABLE_SIZE | 4096 | The largest HPACK dynamic table the sender will accept |
| SETTINGS_ENABLE_PUSH | 1 | Whether the sender will accept PUSH_PROMISE |
| SETTINGS_MAX_CONCURRENT_STREAMS | no limit | How many streams the sender will have open at once |
| SETTINGS_INITIAL_WINDOW_SIZE | 65535 | The flow-control window the sender gives each new stream |
| SETTINGS_MAX_FRAME_SIZE | 16384 | The largest frame payload the sender will accept |
| SETTINGS_MAX_HEADER_LIST_SIZE | no limit | The largest header list the sender will accept |

A connection becomes HTTP/2 in one of three ways. Over TLS, **ALPN**
negotiates the identifier `h2`. Without TLS, a client that already knows
the server speaks HTTP/2 uses **prior knowledge** and simply sends the
24-byte connection preface. A client that does not know asks over
HTTP/1.1 with an `Upgrade: h2c` header. RFC 9113 section 3.1 removed
that third mechanism, and every service mesh and reverse proxy still
runs it.

These are the fixed numbers the protocol gives.

| Quantity | Value | Reference |
| --- | --- | --- |
| Frame header length | 9 bytes | Section 4.1 |
| Smallest SETTINGS_MAX_FRAME_SIZE a peer may announce | 16384 | Section 6.5.2 |
| Largest SETTINGS_MAX_FRAME_SIZE a peer may announce | 16777215 | Section 6.5.2 |
| Initial flow-control window | 65535 | Section 6.9.2 |
| Largest flow-control window | 2147483647 | Section 6.9.1 |
| Largest stream identifier | 2147483647 | Section 5.1.1 |
| Connection preface | 24 bytes | Section 3.4 |
| HPACK static table | 61 entries | RFC 7541 appendix A |
| HPACK per-entry overhead | 32 bytes | RFC 7541 section 4.1 |

This package performs no input or output. It opens nothing, writes
nothing and consults no clock. A connection takes the bytes a host read
and answers what it learned and what the host now owes, so the same code
runs in a server, in a client, in a proxy and in a tool reading a packet
capture.

## Install

```
novo pkg add http2-nv
```

## Example

```novo
use h2conn
use h2err
use h2hpack
use h2prelude
use h2settings

fn main() [io]
    // A client on a connection whose TLS layer negotiated `h2`.
    let c = h2conn.client(H2ViaAlpn, h2conn.default_limits(),
                          h2settings.default_settings())

    // The preface and this endpoint's SETTINGS, as actions to perform.
    // Nothing here writes them: the host does, in this order.
    let opening = h2conn.start(c)
    for a in opening.actions
        match a
            H2DoSend(data)               => println("write ${list.len(data)} byte(s)")
            H2DoExpectSettingsAck(ms)    => println("close if no ACK in ${ms} ms")
            H2DoCloseConnection(code, _) => println(h2err.error_code_name(code))
            H2DoDropStream(s)            => println("stop waiting on stream ${s}")

    // A stream, then a request head on it.  The connection picks the
    // identifier, because only it knows which numbers are still free.
    match h2conn.open_stream(opening.conn)
        Err(e) => println(e.message())
        Ok(o)  =>
            match h2conn.send_headers(o.conn, o.stream,
                                      [h2hpack.header(":method", "GET"),
                                       h2hpack.header(":scheme", "https"),
                                       h2hpack.header(":path", "/"),
                                       h2hpack.header(":authority", "example.test")],
                                      true)
                Err(e)   => println(e.message())
                // How many body bytes the two windows would let through.
                Ok(step) => println("${h2conn.send_window(step.conn, o.stream)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: http2-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `h2conn` | A connection as a value. Bytes from the peer go in, and the events the application wants and the actions the host owes come out. Opening a stream, sending headers, data and trailers, resetting a stream, and shutting the connection down. |
| `h2frame` | The nine-byte header and the ten frame types, to and from a caller's bytes. The flag predicates, the padding rule and the priority field. |
| `h2hpack` | HPACK. The static table, the Huffman code, the prefix integer, a dynamic table the caller sizes, and the encode and decode of a whole header block. |
| `h2stream` | The seven stream states, the nine moves between them, and the identifier parity and ordering rules. |
| `h2flow` | Flow control as arithmetic on two integers: what may be sent, what a send costs, and the credit a receiver owes. |
| `h2settings` | The six parameters, the values that are refused, and the retroactive window change a new initial window size causes. |
| `h2prelude` | ALPN, prior knowledge and the h2c upgrade, as values. The 24-byte preface and the `HTTP2-Settings` header. |
| `h2err` | The thirteen error codes, the decode and encode faults, and whether a fault ends one stream or the whole connection. |

## How to choose an entry point

**`h2conn` is the whole protocol.** It joins CONTINUATION frames, keeps
both HPACK tables, runs the stream state machine and does the
flow-control arithmetic. A client, a server or a proxy holds one
`H2Conn` per connection and pumps it. Use it unless you have a reason
not to.

**The four lower modules are usable on their own.** `h2frame` reads and
writes frames, `h2hpack` encodes and decodes header blocks, `h2stream`
answers where a stream is, and `h2flow` answers how much may be sent.
A capture tool that only reads frames needs `h2frame` alone. A device
that holds one stream keeps its window in two integers and never builds
a connection value. See "Running on a microcontroller".

Both entry points use the same values. A frame `h2frame` decoded is the
frame `h2conn` would have decoded, and the windows on an `H2Stream` are
`h2flow`'s.

## The rules a user needs

1. **Call `start` before the first `feed`.** It answers the preface, for
   a client, and this endpoint's SETTINGS, for both. A client sends both
   together and does not wait for the server's, because waiting adds a
   round trip to every connection (RFC 9113 section 3.4).
2. **Perform the actions in the order they came out.** A SETTINGS
   acknowledgement that overtakes the frame it acknowledges, or a
   WINDOW_UPDATE that arrives before the RST_STREAM it credits, puts the
   peer's accounting out of step with yours.
3. **One `feed` produces many events.** A single TCP segment routinely
   carries frames for six streams. A chunk that completes no frame is
   not a fault: the step comes back with two empty lists.
4. **A fault needs two answers.** `feed` refuses with one
   `H2DecodeError`, and the frames the peer is owed come from
   `fault_actions`. A host that closed the socket instead leaves the peer
   to time out.
5. **`h2err.is_connection_fault` decides the recovery** (section 5.4).
   True means every stream is over and the peer is owed a GOAWAY. False
   means one stream is over and the peer is owed a RST_STREAM. Tearing
   down a connection for a stream error loses ninety-nine calls because
   the hundredth had a bad header. Recovering a connection error as a
   stream error leaves the HPACK table desynchronised for the rest of
   the connection.
6. **Anything HPACK is always the connection's fault.** The dynamic table
   is shared, so a decoder that lost its place cannot read any later
   block. The code is COMPRESSION_ERROR.
7. **A DATA frame is counted against two windows**, the connection's and
   the stream's, and a send may exceed neither (section 5.2.1).
   `h2flow.sendable` is the minimum of those two and the peer's maximum
   frame size. `h2conn.send_window` asks the same question of a whole
   connection.
8. **Only DATA is flow controlled.** HEADERS, SETTINGS, PING,
   RST_STREAM and WINDOW_UPDATE are not, and the last one is why: if
   WINDOW_UPDATE were flow controlled, a connection whose window had
   closed could never be reopened.
9. **A window may be negative, and must not be clamped.** Section
   6.9.2's retroactive change can push an open stream below zero. The
   sender then sends nothing until credit lifts it back. Clamping at
   zero sends bytes the peer counts as a violation.
10. **Credit is owed on delivery, not on arrival.** The window shrinks
    when bytes arrive and reopens only once the application has read
    them. Call `h2conn.delivered` when it has. A host that never calls it
    transfers 65535 bytes and stops, which is the most common "HTTP/2 is
    slow" report there is.
11. **`send_data` takes what it can and says how much.** A send of 100
    KiB against a 16 KiB window answers `taken` of 16384, and the caller
    sends the rest when `H2EvWindowOpened` arrives. `taken` of 0 is a
    stalled stream, which is ordinary and not a fault. `end_stream` is
    applied only if the whole of the data was taken.
12. **There are two settings copies and they are never equal.** Writing
    is bounded by the remote SETTINGS_MAX_FRAME_SIZE, what the peer said
    it would accept. Reading is bounded by the local one. A connection
    that kept one copy sends frames the peer refuses, and the symptom
    arrives as a GOAWAY minutes later.
13. **New local settings take effect when the peer acknowledges, not
    when they are sent** (section 6.5.3). `announce` puts the SETTINGS
    frame in the actions along with `H2DoExpectSettingsAck`, and a host
    that does not hear the acknowledgement in time closes the connection
    with SETTINGS_TIMEOUT. The timer is the host's, because this package
    has no clock.
14. **A change to SETTINGS_INITIAL_WINDOW_SIZE is retroactive** (section
    6.9.2). Every already-open stream's send window moves by the
    difference, which `h2settings.window_delta` computes and
    `h2flow.retune` applies. Applying the new value only to streams
    opened afterwards deadlocks against a peer that shrank its window.
15. **Unknown things are ignored, not refused.** An unknown frame type
    is discarded (section 4.1), an unknown SETTINGS identifier is
    ignored (section 6.5.2), and an unknown error code is kept as its
    number and handled as INTERNAL_ERROR (section 7). Each of those is an
    extension point, and refusing one breaks against every peer that
    implements something newer.
16. **half-closed (local) and half-closed (remote) are two states**
    (section 5.1). One means this endpoint has finished talking, the
    other means the peer has, and they have opposite consequences for
    every later frame. A half-closed stream still admits WINDOW_UPDATE,
    PRIORITY and RST_STREAM.
17. **A closed stream still receives frames, and that is normal.** After
    a RST_STREAM there is a flight of frames the peer sent before it
    heard. `h2stream.tolerates_after_close` says which to tolerate.
    Answering each one with another RST_STREAM is how two
    implementations reset each other until one gives up.
18. **Stream identifiers only go up and are never reused** (section
    5.1.1). Clients use odd numbers and servers even. An identifier at or
    below the highest one already seen is a connection error. When the
    space is exhausted, `next_client_id` answers 0 and the endpoint opens
    a new connection.
19. **A header block is decoded whole or not at all.** `h2conn` joins
    CONTINUATION frames before it decodes, and `h2hpack.decode` is never
    called with a fragment. `H2Limits.max_continuation` bounds the join,
    which is the defence against an unbounded CONTINUATION sequence.
20. **Thread the HPACK table that comes back.** `encode_into` and
    `decode` each answer a new table, because an incrementally-indexed
    header changed it. A caller that keeps the old one desynchronises
    from its peer on the very next block.
21. **An HPACK entry larger than the whole table empties the table and
    is not added** (RFC 7541 section 4.4). It is not an error, and the
    table does not grow past its maximum.
22. **HPACK indices are one space and they are 1-based.** Index 1 to 61
    are the static table, 62 is the newest dynamic entry. Index 0 is not
    an entry: it is the marker that a literal name follows.
23. **An entry costs its name plus its value plus 32** (RFC 7541 section
    4.1). The same 32 per entry counts against
    SETTINGS_MAX_HEADER_LIST_SIZE, which is why the limit is not simply
    the bytes.
24. **`never_indexed` is a property of the value and it travels** (RFC
    7541 section 7.1.3). It is what an `authorization` header or a
    session cookie gets, because a compression table shared across
    requests is what CRIME and BREACH read secrets out of. Use
    `h2hpack.header_never_indexed`.
25. **Huffman-code a string only when it is shorter** (RFC 7541 section
    5.2). The code is optimised for the characters headers contain, so a
    base64 blob or a non-Latin path gets longer.
    `h2hpack.huffman_shorter` is the test.
26. **HTTP/2 header names are lowercase** (section 8.2.1). A name with a
    capital in it is a malformed message rather than something to
    normalise. `connection`, `keep-alive`, `proxy-connection`,
    `transfer-encoding` and `upgrade` are forbidden outright. Every
    pseudo-header comes before every regular header (section 8.3).
27. **`never_processed` is what makes a non-idempotent retry safe.**
    A stream above a received GOAWAY's `last_stream`, or one the peer
    reset with REFUSED_STREAM, was certainly never looked at. Nothing
    else is.
28. **An h2c upgrade starts with stream 1 already open and half-closed
    (remote).** The HTTP/1.1 request that asked for the upgrade is that
    stream, and its body has already arrived. A server that opens stream
    1 fresh loses the request.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers the frame layer's arithmetic, HPACK's
prefix integer and its static-table lookups, and the whole of `h2flow`.
Those functions are integer arithmetic over bytes the caller supplies.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds, producing a Cortex-M4 executable. The consumer
it is written for is a sensor reporting to a gateway: one connection,
one or two streams, a nine-byte header to read, a length to subtract
from a window, and a decision about whether it may send.

**`h2conn` does not carry the claim, and neither does HPACK's decoder.**
A connection holds a list of streams, a pending buffer and two dynamic
tables. A dynamic table means eviction, which means storage the caller
did not size. Nor do `static_table`, `huffman_codes` and
`huffman_lengths`, because a function that answers a list of 61 or 257
entries allocates it. A device uses the lookups, which answer an `Int`
from a name it already holds.

`h2settings.embedded_settings` and `h2conn.embedded_limits` are the
numbers a device announces: a 512-byte HPACK table, push off, four
concurrent streams, an 8 KiB window, the minimum frame size and a 2 KiB
header-list ceiling. A peer that respects them needs no allocator on the
device side.

## What is not included

- **A scheduler.** Which stream gets the next byte of a scarce
  connection window is a policy question with no protocol answer.
  `h2flow.sendable` says how much may go and the host decides for whom.
- **Priority.** Section 5.3.1 deprecates RFC 7540's dependency tree and
  section 5.3 lets an endpoint ignore it. The five bytes are parsed so a
  relay can forward them and a capture can be read. There is no
  scheduler to feed them to.
- **Deciding what to push.** PUSH_PROMISE is parsed, written and
  state-machined, because a client that announced SETTINGS_ENABLE_PUSH
  of 0 and then receives one is entitled to a connection error and has
  to be able to tell. What to push is an application's decision.
- **TLS, ALPN negotiation and the socket.** `h2prelude` publishes the
  identifier to offer and the preface to send. Offering it is `std.tls`'s
  work and the host's.
- **A clock.** Nothing here times out a SETTINGS acknowledgement, a
  request or a stalled stream. `H2DoExpectSettingsAck` hands the host a
  number and the host arms the timer.
- **HTTP/1.1.** The two wire formats share a semantics and nothing else.
  An `H1Header` is a field line whose case is insignificant, and an
  HTTP/2 header name with a capital in it is a malformed message. A host
  that bridges the two converts between them in a few lines.

## Related packages

- [grpc-codec-nv](https://novo-lang.org/packages/grpc-codec-nv) is
  gRPC's five-byte length-prefixed message framing, which sits inside
  the DATA frames this package carries. It needs no change to run here.
- [grpc-nv](https://novo-lang.org/packages/grpc-nv) is the host half of
  gRPC. Its transport is six calls onto a connection from this package:
  `open_stream`, `send_headers`, `send_data`, `send_trailers`,
  `reset_stream` and the next event from `feed`. Its retry decision is
  `h2conn.never_processed`.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) is
  HTTP/1.1: the same methods, statuses and header fields as text, with
  one message in flight at a time. A server that speaks both holds one
  codec of each.
- [websocket-codec-nv](https://novo-lang.org/packages/websocket-codec-nv)
  is the WebSocket frame format. Its handshake is an HTTP/1.1 upgrade,
  so it sits beside http-codec-nv rather than beside this package.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is this
  package's one dependency. The `HTTP2-Settings` header of an h2c
  upgrade is base64url without padding, and standard base64's `+` and
  `/` are not safe in a header field.
- `std.tls` in the standard library is where ALPN offers `h2`.
  `std.net` is where the bytes come from and go.

## Tests

```bash
novo test tests/h2frame_tests.nv      # 9 tests on the frame layer
novo test tests/h2hpack_tests.nv      # 11 tests on HPACK
novo test tests/h2stream_tests.nv     # 8 tests on the state machine
novo test tests/h2flow_tests.nv       # 6 tests on flow control
novo test tests/h2settings_tests.nv   # 8 tests on the parameters
novo test tests/h2conn_tests.nv       # 10 tests on the connection
novo test tests/h2write_tests.nv      # 12 tests on encoding
```

The reference data is the specifications' own. RFC 7541 appendix C
supplies the HPACK vectors, appendix A the 61 static-table entries and
appendix B the 257 Huffman codes. RFC 9113 supplies the frame layouts,
the error code numbers, the SETTINGS defaults and the state machine. The
port follows Rust's [`h2`](https://docs.rs/h2), with
[`nghttp2`](https://nghttp2.org/) as the second opinion on the frame
layer.

Five of the tests exist because the mistake they catch looks correct.
The same flag bit means two things on two frame types. Only DATA is flow
controlled, and WINDOW_UPDATE must not be, or nothing could unstall a
closed window. A flow-control window may be negative and must not be
clamped. Credit is owed on delivery and not on arrival. An HPACK entry
larger than the whole table empties it rather than growing it.

`tests/embedded_probe.nv` is not one of the suites. It is built for
`--target=nrf52-qemu`, and what it proves is that the frame arithmetic,
the HPACK lookups and the whole of flow control link on a Cortex-M4.

The tests compile today and fail at run, each on the
`not implemented: http2-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `H2_*` constants in `h2frame`, `h2flow`, `h2hpack` and `h2prelude` | yes (they are constants) |
| `h2conn.client`, `.server`, `.start`, `.default_limits`, `.embedded_limits` | no |
| `h2conn.feed`, `.fault_actions` | no |
| `h2conn.open_stream`, `.send_headers`, `.send_trailers`, `.send_data`, `.reset_stream` | no |
| `h2conn.go_away`, `.ping`, `.announce`, `.delivered` | no |
| `h2conn.send_window`, `.stream_state`, `.open_count`, `.is_open`, `.goaway_last_stream`, `.never_processed`, `.local_settings`, `.remote_settings`, `.pending_bytes` | no |
| `h2frame.header_of_bytes`, `.header_into`, `.frame_of_bytes`, `.frame_into` | no |
| `h2frame.frame_kind_num`, `.frame_kind_of_num`, `.frame_kind_name`, `.frame_kind`, `.frame_stream`, `.frame_length` | no |
| `h2frame.needs_stream`, `.is_connection_scoped`, `.is_flow_controlled`, `.stream_id_ok` | no |
| `h2frame.flag_end_stream`, `.flag_end_headers`, `.flag_padded`, `.flag_priority`, `.flag_ack`, `.unpad` | no |
| `h2frame.priority_of_bytes`, `.default_priority`, `.priority_is_self_dependent` | no |
| `h2hpack.static_table`, `.static_at`, `.static_index_of`, `.static_index_of_name` | no |
| `h2hpack.table`, `.table_insert`, `.table_resize`, `.table_at`, `.table_len`, `.table_index_of`, `.entry_size` | no |
| `h2hpack.integer_into`, `.integer_of_bytes`, `.string_into`, `.string_of_bytes` | no |
| `h2hpack.huffman_codes`, `.huffman_lengths`, `.huffman_encoded_len`, `.huffman_shorter`, `.huffman_into`, `.huffman_of_bytes` | no |
| `h2hpack.header`, `.header_never_indexed`, `.indexing_of`, `.is_pseudo`, `.name_ok`, `.value_ok`, `.pseudo_order_fault`, `.list_size` | no |
| `h2hpack.encode_into`, `.decode`, `.table_update_into` | no |
| `h2stream.stream`, `.state_name`, `.move_name`, `.state_after`, `.advance` | no |
| `h2stream.may_send`, `.may_receive`, `.tolerates_after_close`, `.is_closed`, `.send_finished`, `.recv_finished` | no |
| `h2stream.is_client_initiated`, `.next_client_id`, `.next_server_id`, `.new_id_ok` | no |
| `h2stream.fault_is_connection`, `.fault_code`, and `H2StreamFault.message` | no |
| `h2flow.window`, `.sendable`, `.spend`, `.receive`, `.deliver`, `.credit` | no |
| `h2flow.update_due`, `.granted`, `.retune`, `.is_stalled`, `.fault_code`, and `H2FlowFault.message` | no |
| `h2settings.setting_id_num`, `.setting_id_of_num`, `.setting_name`, `.default_settings`, `.embedded_settings`, `.unlimited` | no |
| `h2settings.value_ok`, `.apply`, `.apply_all`, `.to_list`, `.diff`, `.window_delta`, `.frame_fits` | no |
| `h2prelude.connection_preface`, `.preface_ok`, `.preface_fault`, `.alpn_offer`, `.alpn_chose_h2` | no |
| `h2prelude.settings_payload`, `.upgrade_settings_value`, `.settings_of_upgrade_value`, `.upgrade_request_headers`, `.upgrade_accepted_status`, `.upgraded_stream_id`, `.prelude_is_current` | no |
| `h2err.error_code_num`, `.error_code_of_num`, `.error_code_name`, `.effective_code` | no |
| `h2err.is_connection_fault`, `.code_for`, `.stream_of`, and both `message` impls | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
