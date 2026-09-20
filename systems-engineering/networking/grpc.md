# gRPC

## 30-Second Intuition

gRPC is a high-performance RPC framework — you define a service contract once in a `.proto` file, and `protoc` generates typed client and server stubs in whichever language you need, so a "remote call" reads like a local function call instead of a hand-built JSON-over-HTTP integration. It runs on two things doing their jobs together: HTTP/2 for the wire transport (multiplexed streams, framed and binary at the transport layer) and Protocol Buffers for the message encoding (compact, schema-defined binary instead of text). The one fact that matters operationally: **one gRPC channel is one (or a small pool of) long-lived TCP connection(s), and every concurrent RPC on that channel — no matter how many are in flight — is its own independent HTTP/2 stream multiplexed over it**, which is why gRPC avoids both HTTP/1.1's connection-per-request overhead and its head-of-line blocking, at the cost of needing HTTP/2-aware infrastructure (proxies, load balancers) everywhere in the path, a requirement that trips up more production deployments than any other part of the design.

Current status (grounded September 2026): the core `grpc/grpc` repo is at **v1.83.x** (v1.83.1, Aug 27, 2026); Protobuf has moved from the proto2/proto3 syntax split to an **Editions** system (edition `2023` was the initial baseline, edition `2024` is the latest GA edition, edition `2026` exists but is parse-tolerated only / not GA upstream yet) — proto3 and proto2 files still compile and interoperate, Editions is additive, not a breaking replacement. gRPC-Web remains necessary because browsers cannot originate raw HTTP/2 trailers-based gRPC calls; HTTP/3 support for gRPC/gRPC-Web/Connect reached a real milestone in 2026 once `quic-go` v0.47.0 added HTTP trailer support over QUIC, though gRPC-over-HTTP/3 is still primarily a same-org/service-mesh play rather than a public-internet default.

---

## Resource-Layer Map

| Layer | Role gRPC plays | What it's optimizing for |
|---|---|---|
| CPU | Protobuf's binary wire format is decoded via **generated code that reads fields directly by position/tag**, not via a general-purpose reflective parser — no string tokenizing, no runtime type dispatch the way a JSON parser building a generic map/object graph requires. This is real, measurable CPU savings per message, not just smaller bytes | Minimizing serialization/deserialization CPU cost per RPC, which matters most at high call volume (internal service-to-service traffic, not occasional API calls) |
| Memory | One multiplexed HTTP/2 connection per channel means far fewer open sockets, TCP send/receive buffers, and TLS session contexts than a model that opens a connection per outstanding request (classic HTTP/1.1 client pools). A server handling thousands of concurrent RPCs from one client does it over a handful of connections, not thousands | Bounding per-connection memory overhead (socket buffers, TLS state) as concurrent RPC count grows, by decoupling "concurrent logical calls" from "concurrent TCP connections" |
| Disk | Not applicable — gRPC is a wire protocol and RPC framework with no persistence layer of its own; whatever a service does with the request after decoding it (write to a database, append to a log) is that service's own concern, not gRPC's | N/A |
| Network | **This is the layer gRPC is actually optimized for.** Two independent wins stack: (1) HTTP/2 stream multiplexing means N concurrent RPCs share one TCP connection as N independent streams — no per-request TCP/TLS handshake, and no application-level request blocking another the way HTTP/1.1 pipelining does; (2) Protobuf's binary encoding is smaller on the wire than the equivalent JSON for the same logical message (no field-name repetition, no textual number encoding) | Minimizing both the *number of round trips/connections* needed for a given concurrency level and the *number of bytes* per message — the two levers that dominate network-bound RPC-heavy architectures (microservices calling microservices) |
| GPU | Not applicable — gRPC has no computation of its own; it is a transport/serialization layer, not a workload | N/A |

Two systems that look similar on paper diverge here first: gRPC vs. REST/JSON-over-HTTP/1.1 differ primarily in the network row (connection-per-request + text serialization vs. multiplexed-stream + binary serialization), and that single difference is responsible for nearly every practical reason teams pick one over the other for internal service meshes.

---

## The Signature Mechanism: HTTP/2 Stream Multiplexing + Protobuf Binary Encoding

Neither half of this mechanism is gRPC-specific in isolation — HTTP/2 multiplexing is a transport-layer feature any HTTP/2 client gets, and Protobuf is a serialization format usable without gRPC at all (plenty of systems use `.proto`-defined messages over Kafka, for instance). gRPC's actual signature mechanism is **binding the two together as the default, so that "call a method" and "get a multiplexed, binary-framed RPC" are the same action** — you don't opt into either piece separately.

### Multiplexing: one connection, many concurrent streams

```
                     ONE TCP connection (one gRPC "channel")
                     │
        ┌────────────┼────────────┬────────────────┐
        │            │            │                │
   stream 1      stream 3     stream 5         stream 7
  (unary RPC    (streaming   (unary RPC       (unary RPC
   in flight)    RPC, long-   in flight)        in flight)
                 lived)
        │            │            │                │
   HEADERS      HEADERS       HEADERS          HEADERS
   DATA         DATA          DATA             DATA
   (trailers)   DATA...       (trailers)       (trailers)
                DATA...
                (trailers)
```

Each stream is independent: a large DATA frame on stream 3 does not block a small unary call's HEADERS/DATA/trailers on stream 5 from completing — HTTP/2 interleaves frames from different streams on the same connection, so one slow or large RPC cannot head-of-line-block an unrelated fast one *at the connection level* (this is exactly the failure mode HTTP/1.1 pipelining has, and exactly why HTTP/1.1 clients instead open a connection pool to fake concurrency). gRPC stream IDs are odd for client-initiated streams and increase monotonically per channel; a channel can carry a very large number of concurrent RPCs before it needs a second connection (the practical ceiling is governed by `SETTINGS_MAX_CONCURRENT_STREAMS`, commonly defaulted around 100–250 depending on server implementation and tunable).

### Protobuf: schema-defined binary encoding, worked by hand

Take a small message:

```protobuf
message Point {
  int32 x = 1;
  int32 y = 2;
}
```

Encode `Point{x: 150, y: 2}`. Protobuf's wire format is a sequence of `(tag, value)` pairs, no field names, no delimiters:

- **Tag byte** = `(field_number << 3) | wire_type`. `x` is field 1, an `int32`, which uses wire type 0 (varint): tag = `(1 << 3) | 0` = `0x08`.
- **Value**: 150 as a varint. Varints encode 7 bits per byte, MSB set on all but the last byte. 150 = `0b10010110`. Split into 7-bit groups (LSB-first): `0010110` and `0000001`. Byte 1 gets its continuation bit set (more bytes follow): `10010110` = `0x96`. Byte 2 has no continuation bit: `00000001` = `0x01`.
- So field `x` encodes as: `0x08 0x96 0x01` (3 bytes).
- **Tag byte for `y`** (field 2, varint): `(2 << 3) | 0` = `0x10`.
- **Value for `y` = 2**: fits in one byte, no continuation bit: `0x02`.
- So field `y` encodes as: `0x10 0x02` (2 bytes).

**Total wire bytes: `08 96 01 10 02` — 5 bytes.**

The equivalent JSON: `{"x":150,"y":2}` — 16 bytes as UTF-8, more than 3x larger, and that gap widens with every additional field name and every level of nesting (JSON repeats every field name in every message instance; Protobuf's tag is 1-2 bytes regardless of how long the field name is in the `.proto` source, because the name never goes on the wire). This is also why Protobuf decoding is cheaper on CPU, not just smaller on the wire: the generated parser reads a tag, does a switch/jump on the field number to know exactly which struct field and type to expect, and reads a fixed-shape varint/length-delimited value — no string comparison against field names, no dynamic type inference the way a JSON parser building a generic value tree needs.

Verifying this by hand with `protoc`'s own text/binary tools:

```bash
$ cat point.proto
syntax = "proto3";
message Point { int32 x = 1; int32 y = 2; }

$ echo 'x: 150 y: 2' | protoc --encode=Point point.proto | xxd
00000000: 0896 0110 02                             ......

$ echo 'x: 150 y: 2' | protoc --encode=Point point.proto | wc -c
5
```

Matches the hand-derived encoding exactly: `08 96 01 10 02`.

---

## High-to-Low Walkthrough

Trace a single unary RPC from `.proto` file to HTTP/2 frames on the wire.

**1. Define the service contract:**

```protobuf
syntax = "proto3";
package inventory.v1;

service InventoryService {
  rpc GetItem (GetItemRequest) returns (GetItemResponse);
}

message GetItemRequest {
  string item_id = 1;
}

message GetItemResponse {
  string item_id = 1;
  string name = 2;
  int32 quantity = 3;
}
```

**2. Generate code with `protoc`:**

```bash
$ protoc --go_out=. --go-grpc_out=. inventory.proto
# produces inventory.pb.go (message types + Protobuf marshal/unmarshal)
# and inventory_grpc.pb.go (client stub + server interface)
```

This generates, in outline (Go-flavored pseudocode, but the shape is the same in Java/Python/C++):

```go
// Client stub — generated
type InventoryServiceClient interface {
    GetItem(ctx context.Context, in *GetItemRequest, opts ...CallOption) (*GetItemResponse, error)
}

// Server interface the application implements
type InventoryServiceServer interface {
    GetItem(context.Context, *GetItemRequest) (*GetItemResponse, error)
}
```

**3. The application calls it like a local function:**

```go
conn, _ := grpc.NewClient("inventory.svc.internal:443",
    grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)))
client := inventory.NewInventoryServiceClient(conn)

resp, err := client.GetItem(ctx, &inventory.GetItemRequest{ItemId: "sku-123"})
```

`conn` here is the channel — the one (pooled) HTTP/2 connection every subsequent `GetItem` call, and every call to any other method on this stub, reuses as a new stream.

**4. What actually goes over the wire — one new HTTP/2 stream, three frame groups:**

```
Client                                                    Server
  │                                                          │
  │──── HEADERS (stream 5, END_HEADERS) ───────────────────▶│
  │      :method: POST                                       │
  │      :scheme: https                                      │
  │      :path: /inventory.v1.InventoryService/GetItem       │
  │      :authority: inventory.svc.internal                  │
  │      content-type: application/grpc+proto                │
  │      grpc-timeout: 500m                                   │
  │      te: trailers                                          │
  │                                                            │
  │──── DATA (stream 5) ─────────────────────────────────▶  │
  │      [1-byte compressed flag][4-byte length][protobuf bytes]│
  │      (this 5-byte prefix + payload is the gRPC "Length-    │
  │       Prefixed Message" framing, sitting *inside* the       │
  │       HTTP/2 DATA frame payload)                            │
  │                                                            │
  │                                    server decodes GetItemRequest,
  │                                    runs InventoryServiceServer.GetItem()
  │                                                            │
  │◀──── HEADERS (stream 5) ─────────────────────────────────│
  │      :status: 200                                          │
  │      content-type: application/grpc+proto                  │
  │                                                            │
  │◀──── DATA (stream 5) ────────────────────────────────────│
  │      [flag][length][serialized GetItemResponse bytes]       │
  │                                                            │
  │◀──── HEADERS (stream 5, END_STREAM, trailers) ───────────│
  │      grpc-status: 0                                        │
  │      grpc-message: (empty on success)                       │
  │                                                            │
```

Two things worth naming explicitly: `grpc-status` (the actual RPC outcome) rides in the **trailing HEADERS frame**, not the leading `:status` pseudo-header — `:status: 200` only means "the HTTP/2 exchange itself succeeded," it says nothing about whether the RPC succeeded, which is precisely the field browsers can't read directly (see gRPC-Web below). The 5-byte length-prefix inside each DATA frame is gRPC's own message framing, independent of HTTP/2's own frame length — it exists because a single logical gRPC message can span multiple HTTP/2 DATA frames (or a single DATA frame can carry multiple small gRPC messages back-to-back on a streaming RPC).

**5. Verifying the same call against a real, reflection-enabled server with `grpcurl`** (no `.proto` file needed client-side — the server exposes its own schema via gRPC Server Reflection):

```bash
$ grpcurl -plaintext localhost:50051 list
inventory.v1.InventoryService
grpc.reflection.v1.ServerReflection

$ grpcurl -plaintext localhost:50051 list inventory.v1.InventoryService
inventory.v1.InventoryService.GetItem

$ grpcurl -plaintext localhost:50051 describe inventory.v1.InventoryService.GetItem
inventory.v1.InventoryService.GetItem is a method:
rpc GetItem ( .inventory.v1.GetItemRequest ) returns ( .inventory.v1.GetItemResponse )

$ grpcurl -plaintext -d '{"item_id": "sku-123"}' \
    localhost:50051 inventory.v1.InventoryService/GetItem
{
  "itemId": "sku-123",
  "name": "Widget",
  "quantity": 42
}
```

`grpcurl`'s `-v` flag surfaces exactly the trailer this doc keeps pointing at:

```bash
$ grpcurl -v -plaintext -d '{"item_id": "sku-123"}' \
    localhost:50051 inventory.v1.InventoryService/GetItem
Resolved method descriptor:
rpc GetItem ( .inventory.v1.GetItemRequest ) returns ( .inventory.v1.GetItemResponse )

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc

Response contents:
{
  "itemId": "sku-123",
  "name": "Widget",
  "quantity": 42
}

Response trailers received:
grpc-status: 0
grpc-message:
```

---

## Deep Internals

### The 4 RPC Types

**Unary** — one request, one response, already shown above. One HTTP/2 stream, opened and closed within a single round trip's worth of frames.

**Server streaming** — one request, a stream of responses:

```protobuf
service InventoryService {
  rpc WatchStockLevel (WatchRequest) returns (stream StockUpdate);
}
```

```bash
$ grpcurl -plaintext -d '{"item_id": "sku-123"}' \
    localhost:50051 inventory.v1.InventoryService/WatchStockLevel
{
  "quantity": 42
}
{
  "quantity": 41
}
{
  "quantity": 39
}
```

HTTP/2 shape: one HEADERS + one DATA frame from the client (the single request), then the server keeps the same stream open and sends **multiple DATA frames over time**, closing with a trailing HEADERS (`grpc-status`) only when it's done (or the client cancels). The stream stays open as long as updates keep coming — this is a long-lived stream with one-directional flow.

**Client streaming** — a stream of requests, one response:

```protobuf
service InventoryService {
  rpc BulkUpdateStock (stream StockUpdate) returns (UpdateSummary);
}
```

HTTP/2 shape: the client sends multiple DATA frames on the same stream over time (each one a length-prefixed message), then a final `END_STREAM` marks it's done sending; the server processes all of them and responds with a single HEADERS+DATA+trailer once, at the end.

**Bidirectional streaming** — both sides stream independently on the same stream:

```protobuf
service InventoryService {
  rpc SyncInventory (stream StockUpdate) returns (stream StockUpdate);
}
```

```
Client                                              Server
  │── HEADERS (stream 7) ─────────────────────────▶│
  │── DATA (update 1) ────────────────────────────▶│
  │◀───────────────────────────── DATA (ack/merge) │
  │── DATA (update 2) ────────────────────────────▶│
  │◀───────────────────────────── DATA (ack/merge) │
  │◀───────────────────────────── DATA (ack/merge) │
  │── DATA (update 3), END_STREAM ────────────────▶│
  │◀── HEADERS (trailers, grpc-status: 0) ─────────│
```

This is the case that most clearly shows off HTTP/2 multiplexing at the single-stream level: DATA frames flow in both directions, interleaved, on one long-lived stream, with no requirement that a client message be immediately followed by a server message (it's not request/response ping-pong — either side can send whenever it has something, independently).

### Protobuf wire format basics

Every field on the wire is a `(tag, value)` pair. The tag byte is `(field_number << 3) | wire_type`. The wire types that matter in practice:

| Wire type | Value | Used for |
|---|---|---|
| 0 | Varint | int32, int64, uint32, uint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, repeated packed fields |
| 5 | 32-bit | fixed32, sfixed32, float |

Varint encoding (worked above for `x = 150`) is why small numbers are cheap (1 byte for 0-127) and why Protobuf recommends field numbers 1-15 for frequently-set fields — those get a 1-byte tag, field numbers 16+ need a 2-byte tag (the field number shifts left by 3 bits, so it overflows a single byte sooner).

### Deadline/cancellation propagation

This is a genuinely distinctive gRPC feature relative to typical REST clients, where a per-request timeout is usually local to one hop and nothing tells the downstream service to stop working once the caller has given up. gRPC deadlines are carried **as request metadata** (`grpc-timeout` header, seen in the HEADERS frame in the walkthrough above) and are meant to propagate through an entire call chain:

```
Client sets a 500ms deadline
  │
  ▼
Service A receives the call with grpc-timeout: 500m
  │  (100ms of local work)
  ▼
Service A calls Service B, propagating the REMAINING deadline (~400ms),
not a fresh 500ms — this is the propagation contract, and it is opt-in:
the calling code must explicitly derive the outbound call's context/deadline
from the inbound one; nothing does this automatically unless the framework
or an interceptor is configured to
  │
  ▼
Service B calls Service C with the further-reduced remaining deadline
  │
  ▼
If the client cancels or the original deadline expires, gRPC's cancellation
signal propagates down the same chain: Service A's call to B gets cancelled,
which cancels B's call to C — bounding the total wasted work across every
hop, not just the first one
```

Concretely, in Go, this looks like passing the same `context.Context` (or a `context.WithTimeout` derived from it) into every downstream call:

```go
func (s *serviceA) GetItem(ctx context.Context, req *GetItemRequest) (*GetItemResponse, error) {
    // ctx already carries the deadline gRPC decoded from grpc-timeout
    return s.serviceBClient.GetItem(ctx, &pb.GetItemRequest{ItemId: req.ItemId})
    // passing ctx forward, not context.Background(), is what makes
    // propagation actually happen — this is the easy-to-forget step
}
```

### gRPC status codes / error model

gRPC uses a small, fixed, structured set of status codes (carried in the `grpc-status` trailer, an integer, with an optional `grpc-message` string and optional binary `google.rpc.Status` details blob) rather than HTTP's much larger and more loosely-applied status code space:

```bash
$ grpcurl -plaintext -d '{"item_id": "does-not-exist"}' \
    localhost:50051 inventory.v1.InventoryService/GetItem
ERROR:
  Code: NotFound
  Message: item does-not-exist not found
```

Common codes: `OK` (0), `CANCELLED` (1), `INVALID_ARGUMENT` (3), `DEADLINE_EXCEEDED` (4), `NOT_FOUND` (5), `ALREADY_EXISTS` (6), `PERMISSION_DENIED` (7), `RESOURCE_EXHAUSTED` (8), `FAILED_PRECONDITION` (9), `UNAVAILABLE` (14), `UNAUTHENTICATED` (16). This is a much smaller, RPC-shaped vocabulary than HTTP's status codes (which conflate transport-level outcomes like 301/304 with application outcomes like 404/409) — a gRPC client checks one field for the definitive outcome, rather than inferring meaning from an HTTP status code that was designed for documents and caches, not RPC semantics.

---

## Comparative: vs. REST/JSON-over-HTTP/1.1, vs. GraphQL

**vs. REST/JSON-over-HTTP/1.1**: the difference is exactly the resource-layer table above, stated as a comparison. HTTP/1.1 REST clients get concurrency by opening a pool of TCP connections (commonly 6 per host in browsers, tunable elsewhere) because HTTP/1.1 pipelining's head-of-line blocking makes true single-connection concurrency unworkable — gRPC gets the same concurrency over one connection via HTTP/2 streams, at lower memory and connection-setup cost. JSON payloads carry field names as text on every message; Protobuf's tag-based encoding never puts a field name on the wire at all (shown numerically above: 5 bytes vs. 16 for a 2-field message, and the gap only grows with schema size). The tradeoff: REST/JSON is human-readable without tooling (curl a URL, read the response) and has universal HTTP/1.1 client support everywhere including every browser natively; gRPC needs a `.proto` schema shared between client and server (or reflection, as `grpcurl` uses) and HTTP/2-aware infrastructure end-to-end.

**vs. GraphQL**: a different axis of problem entirely, not a strict alternative. GraphQL solves over-fetching/under-fetching — a client shapes exactly the fields it wants from a flexible schema graph, valuable when many different frontends need different slices of the same data. gRPC solves fixed-contract, high-throughput RPC between services that agree on an exact message shape in advance and want the cheapest possible encode/decode/transport cost for that fixed shape — there's no per-request query parsing or resolver graph to execute, which is precisely why gRPC is the more common choice for internal service-to-service traffic while GraphQL shows up more at the client-facing API-aggregation layer.

---

## Gotchas

- **HTTP/2-unaware infrastructure breaks gRPC in ways that are hard to diagnose**: some older proxies, load balancers, and corporate middleboxes only partially support HTTP/2 (or silently downgrade/terminate it to HTTP/1.1), which breaks gRPC outright since it depends on HTTP/2 framing and trailers, not just "HTTP/2 as a faster HTTP/1.1." Symptoms tend to look like mysterious `UNAVAILABLE` or connection-reset errors rather than a clear "HTTP/2 not supported" message — verify any intermediary in the path (including cloud load balancers and API gateways) is explicitly documented as gRPC-capable, not merely "HTTP/2 compatible" in a generic sense.
- **A single long-lived multiplexed connection load-balances unevenly at L4**: because one gRPC channel is (by design) a small number of persistent TCP connections, an L4 (connection-level) load balancer distributes *connections*, not *requests* — once a client has established its connections to a handful of backend replicas, all its RPC traffic (potentially thousands of concurrent calls) keeps landing on those same replicas until the connection is torn down and re-established, producing visibly uneven load across a replica set even though the LB's connection-count view looks balanced. The fix is an L7/HTTP2-aware load balancer or proxy (e.g. Envoy, or client-side/lookaside load balancing via xDS) that can distribute individual RPCs (streams) across backends rather than treating each TCP connection as a single unit of load.
- **gRPC-Web exists because browsers can't do this natively — and it's a real protocol translation, not just a JS wrapper**: browser JavaScript cannot read HTTP/2 trailers (where `grpc-status` lives) or freely control HTTP/2 framing, so a direct browser-to-gRPC-server call isn't possible without an intermediary. gRPC-Web moves the trailer information into the response body (or uses a proxy like Envoy's gRPC-Web filter to translate) so the browser can access the real RPC outcome — meaning a browser client is never talking "real" gRPC end-to-end, it's talking gRPC-Web to a proxy that speaks real gRPC to the backend, an extra hop and a protocol difference worth knowing about when debugging.
- **Deadline propagation is opt-in per language implementation and easy to forget**: gRPC decodes an inbound deadline into the language's native context/cancellation primitive (e.g. Go's `context.Context`, or a deadline object in Java/Python), but nothing forces a service to pass that same context into its own downstream calls — a developer who calls a downstream service with a fresh, unrelated context (or no deadline at all) silently breaks the propagation chain, and the result is exactly the failure mode deadlines exist to prevent: a cancelled or timed-out parent call whose downstream work keeps running to completion anyway, wasting resources with no one still waiting on the answer.
- **Protobuf field-number reuse is a silent, delayed-onset schema bug, not a compile error**: once a field number has ever been used and shipped, reusing that number for a different field later causes old data (or old binaries still using the previous meaning) to be misinterpreted as the new field's type — Protobuf will not stop you from doing this, it only tells you when the wire bytes decode into garbage or the wrong field downstream. The standard mitigation is to mark retired field numbers `reserved` so `protoc` refuses to let anyone reuse them by accident:

  ```protobuf
  message GetItemResponse {
    reserved 2;           // old `name` field, retired — never reuse
    reserved "old_name";  // reserve the name too, for editions/proto2 interop
    string item_id = 1;
    int32 quantity = 3;
  }
  ```

  This is conceptually the same problem Iceberg's schema evolution discipline addresses with catalog-assigned field IDs (see `data-formats/apache-iceberg.md`'s "Column rename... Safe (field IDs)" row) — both systems decouple the stable on-disk/on-wire identity of a field from its human-readable name so renames are safe — but the mechanisms differ: Iceberg's catalog *assigns* field IDs centrally and tracks the ID-to-name mapping in table metadata that evolves with every schema change, while Protobuf's field numbers are assigned by whoever edits the `.proto` file, with no central authority preventing reuse other than the `reserved` keyword and code review discipline.
- **Adding required-like semantics to proto3/Editions messages is discouraged for the same evolution reasons `required` was removed from proto3 in the first place**: proto2's `required` field validation caused exactly the kind of hard, cross-version breakage schema evolution is supposed to avoid — a server that starts requiring a field breaks every existing client binary that predates that field's existence, with no graceful degradation. proto3 (and Editions) default every field to optional/presence-tracked instead, pushing "this field must be set" into application-level validation rather than wire-format-level enforcement, so that adding a new field is always a backward-compatible change, never a forced upgrade.

---

*Grounded via grpc.io, the `grpc/grpc` GitHub releases page, protobuf.dev's Editions documentation, and gRPC-Web/HTTP-3 status writeups as of September 2026. Core gRPC: v1.83.x (v1.83.1, Aug 27 2026). Protobuf: Editions 2023 (baseline) and 2024 (latest GA) supersede the proto2/proto3 syntax distinction going forward, though proto3/proto2 files remain fully supported and interoperable — edition 2026 exists but is parse-tolerated only, not GA upstream, so treat it as forward-looking rather than something to adopt yet. gRPC-over-HTTP/3 gained real trailer support in `quic-go` v0.47.0 in 2026 but is still mostly a controlled-environment (service mesh, mobile/desktop client you ship yourself) technology rather than a public-internet default — re-verify exact version numbers and edition-2026 GA status before citing either in a deliverable, since both are moving targets.*
