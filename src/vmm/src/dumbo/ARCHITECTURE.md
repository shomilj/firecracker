# In-process TCP/IP stack for metadata networking

This document explains how the lightweight, in-process protocol stack models Ethernet, Internet Protocol version 4, Address Resolution Protocol, and Transmission Control Protocol for the microVM metadata service. The stack runs entirely inside the virtual machine monitor. Guest network frames that target the metadata address are parsed, answered, and re-encapsulated without relying on a host kernel network stack for that path. The design favors determinism, bounded memory, and simplicity over full standards compliance.

---

## Purpose & Boundaries

The system’s responsibility is to interpret incoming link-layer frames as structured protocol units, validate and extract nested layers, drive a minimal passive TCP implementation suitable for serving HTTP-style requests over IPv4, and emit well-formed replies on the same path. It provides parsing and serialization helpers for common PDUs, a connection lifecycle tuned to passive opens only, and integration hooks so a higher layer can turn a parsed HTTP request into a response body.

What lies outside this responsibility is equally important. The stack does not implement a general-purpose router, firewall, or full TCP peer suitable for arbitrary internet workloads. It does not perform congestion control, selective acknowledgements, or window scaling. It does not reassemble IPv4 fragments, and it does not maintain arbitrary out-of-order TCP data queues. Active connection initiation toward guests is not supported. Non-TCP IPv4 traffic toward the metadata address is recognized as unusual but not served at the application layer. If this component fails or rejects traffic, guests lose in-band access to instance metadata over the emulated network path; management-plane APIs that do not depend on that path may still work, but anything that expects link-local HTTP to the metadata address will stall or reset.

---

## Interfaces & Contracts

**Exposure.** Callers receive a way to classify whether a frame is intended for the metadata service, to hand matching frames into the stack for processing, and to drain the next outbound frame when one is ready. Inbound processing updates connection tables, may invoke an application callback with a fully framed HTTP request, and records metrics through the surrounding crate. Outbound processing prioritizes pending address-resolution replies, then TCP segments including control segments and application payloads wrapped in IPv4 and Ethernet.

**Consumption.** The stack consumes raw octets representing Ethernet frames. For IPv4 payloads it expects a coherent datagram whose stated length matches the buffer; checksum verification is often skipped on ingress to accommodate possible offload or validation elsewhere in the device model. The application layer supplies a callback that maps a parsed request to a response; the stack does not embed business rules for metadata keys or tokens.

**Caller invariants.** Frames passed to the handler after classification must actually be destined for the metadata IPv4 address and, for TCP, the configured server port. Timestamps used for retransmission timing must be monotonically non-decreasing. Buffers provided for transmission must be large enough for the written headers and payloads the stack is configured to produce.

**Guarantees.** Within its simplifying assumptions, the stack provides predictable behavior: bounded connection counts, bounded pending reset traffic, explicit events when connections are created, replaced, torn down, or when unexpected segments arrive. Writing operations return length information so the device model can emit exactly one Ethernet frame when appropriate.

---

## Data Flow

Before any TCP bytes move, the guest may need to resolve the metadata IPv4 address to a MAC address. An ARP request frame arrives with an Ethernet header and an ARP payload asking who owns the metadata address. The stack records the requester’s link-layer and network-layer addresses and schedules a reply. On the transmit side, an Ethernet frame is built with the ARP reply mapping the stable server MAC and IPv4 address to the requester’s addresses.

The following diagram shows the high-level ingress path from the wire to the application callback.

```
  Guest NIC / tap
        |
        v
  Ethernet frame  ----->  Ethertype?
        |                    |
        |                    +-- ARP  -----> record peer, queue ARP reply
        |
        +-- IPv4  ----->  protocol field
                              |
                              +-- TCP  ----->  strip to segment
                                                    |
                                                    v
                                            listener demux by
                                            (remote IP, remote port)
                                                    |
                                                    v
                                            connection + HTTP
                                            assembly buffer
                                                    |
                                                    v
                                            request -> callback -> response bytes
```

Once TCP is established, IPv4 datagrams carry TCP segments as their payload. The segment supplies ports, sequence and acknowledgement fields, flags, window, and optional options. The listener extracts the segment, checks the local port, and routes to an existing conversation or treats the segment as the start of a new one if it matches the passive-open pattern. Payload octets that pass validation are copied into a fixed receive buffer managed above the pure transport state machine. When the application has produced a response, that response is held in a dynamic buffer while the transport layer slices it into segments constrained by the maximum segment size and the peer’s advertised window.

Egress reverses the nesting: TCP segment bytes are placed in an IPv4 packet buffer, addresses and length fields are set, header checksums are completed, then the IPv4 packet becomes the Ethernet payload. Address learning updates the destination MAC from the last seen frame on that attachment point.

The egress composition can be visualized as nested envelopes.

```
  +----------------------------------------------------------+
  | Ethernet: dst MAC, src MAC, ethertype IPv4               |
  |  +----------------------------------------------------+  |
  |  | IPv4: src/dst addresses, length, protocol TCP     |  |
  |  |  +--------------------------------------------+    |  |
  |  |  | TCP: ports, seq/ack, flags, window, payload|    |  |
  |  |  +--------------------------------------------+    |  |
  |  +----------------------------------------------------+  |
  +----------------------------------------------------------+
```

---

## Control Flow

Execution is driven by the virtual device model on each received frame and on each opportunity to transmit. There is no independent kernel thread; the stack advances when the surrounding code invokes receive and transmit helpers.

The critical receive path begins with frame classification. If the frame is ARP and targets the metadata IPv4 address, the stack stores state for a reply and returns. If the frame is IPv4 and the protocol is TCP, the stack parses the segment and either attaches it to an existing conversation, begins a new conversation if the opening segment is valid, or enqueues a connection reset for unrecognized traffic. The application callback runs synchronously when a complete HTTP request has been assembled in the receive buffer.

The transmit path first flushes any queued ARP reply because address resolution must complete before the guest will accept IPv4 unicast. Otherwise it consults whether immediate transmission is possible or whether a retransmission deadline must elapse first. When allowed, it may emit a pending reset segment, or ask an active conversation for the next TCP segment. After writing, the stack may remove conversations that have finished their teardown sequence.

Branching hinges on segment flags and connection tables. A valid opening segment creates state; a segment for an unknown tuple that is not an opening segment triggers a reset response unless the segment itself is already a reset. When the table is full, new openings may displace an idle conversation if one exceeds an eviction age, or the new opening may be refused with a reset.

---

## State & Lifecycle

**PDU authoring model.** Many structures can exist in an incomplete form while header fields that depend on the full packet—such as checksums after the payload length is known—remain unset. Completion fills in those fields and fixes outer lengths so the byte array is a valid on-wire representation.

**Transport connection state.** Conversations created only through passive open progress through early handshake phases: having accepted an opening segment, having sent the combined acknowledgement and opening reply, and then being established for data. Additional boolean-style markers record whether the graceful close flag was sent and acknowledged, and whether the connection was aborted by reset. Sequence state tracks the next sequence number to send, the highest acknowledgement seen from the peer, receive and send window edges, and whether an acknowledgement or retransmission is pending. Timestamps are opaque counters; they back off retransmission when data is unacknowledged for too long, up to a maximum number of attempts before the connection self-aborts.

**Listener state.** The listener holds a map from remote network identity to active conversation objects, a set of conversations that can send immediately, the nearest future retransmission deadline among conversations, and a bounded queue of reset segments for peers that sent unexpected traffic. Limits apply to concurrent conversations and to how many resets may wait when the system is overloaded.

**Application-level buffering.** A fixed upper bound applies to how many octets of HTTP request line and headers are accumulated while searching for the end of the request. The receive window advertised to the peer is tied to this bound so the peer cannot overrun it. Responses are buffered in a growable store until fully acknowledged, then cleared.

**Lifecycle.** Startup initializes addresses, ports, and empty tables. Steady operation alternates receive-driven mutation and transmit-driven draining. Shutdown of a conversation occurs after both sides have completed the graceful close handshake or after a reset. Eviction removes stale conversations when new ones need space.

A simplified state progression for the handshake and steady data phase appears below. This is conceptual; the implementation uses multiple coordinated fields rather than a single enumerated state.

```
        [heard valid open]
                |
                v
        [sent opening reply]
                |
                v
        [established] <------------------+
                |                       |
                | data / acks           | retransmission /
                v                       | duplicate ack path
        [closing both halves]           |
                |                       |
                v                       |
             [done] ---------------------+
                or [reset]
```

---

## Failure Modes

Malformed link or internet headers cause parse failures; those frames may be dropped with error accounting rather than partially processed. TCP checksum validation may be omitted on ingress; a corrupted segment could theoretically be accepted if surrounding layers do not catch it, which is a deliberate tradeoff noted in the implementation context.

At the transport layer, invalid acknowledgements during the handshake cause a reset. In the established phase, some invalid segments are ignored instead of resetting, which can delay progress but avoids tearing down the conversation on single anomalies. Out-of-order data is not buffered: the implementation expects the next in-sequence segment; anything else is reported as an unusual condition and may trigger an acknowledgement prompting retransmission from the peer.

Receive buffer exhaustion while still unable to find a complete HTTP message causes an active reset because the service cannot bound work otherwise. If the callback fails to serialize a response, higher layers surface errors through metrics.

Silent misbehavior is possible if monotonic time assumptions break, if remote windows move backward without being caught, or if IPv4 length fields lie while checksums are skipped—the design assumes trusted, local, non-Internet traffic.

---

## Operational Characteristics

Resource use is dominated by fixed per-connection footprints, the bounded reset queue, and the maximum request buffer. There is no dynamic growth of out-of-order TCP buffers because that path does not exist. Transmit slicing respects maximum segment size derived from options on the opening segment, with a conservative default when options are absent.

Scaling bottlenecks include the maximum concurrent conversations, the cost of linear scans for eviction candidates, and the single-threaded polling model—throughput is adequate for metadata queries, not for bulk transfer.

Observability is delegated outward: the stack returns rich events and relies on the embedding crate for counters such as accepted frames, errors, and connection churn. The minimalist transport layer itself avoids logging to stay reusable.

---

## Design Rationale

The metadata service only needs short HTTP transactions on a trusted, non-egress path inside the microVM. That justifies omitting congestion control, delayed acknowledgements, window scaling, and out-of-order assembly. Passive-only opens match the server role. Using a compact in-process stack avoids syscall overhead and keeps failure modes visible to the VMM, at the cost of maintaining protocol details that kernels normally hide.

The incomplete PDU pattern separates layout from checksum completion, which keeps parsing and writing symmetric and safe when payloads vary. Identifying conversations by remote address and port while fixing local address and port simplifies the table and matches the single-listener design.

Reset queues and eviction policies acknowledge that guests may misbehave or open many connections; bounding these paths prevents unbounded memory use in pathological cases. Skipping some checksum verification on ingress acknowledges real hardware offload behavior but documents the residual risk.

Taken together, the architecture trades generality for a small, reviewable surface that is sufficient for link-local metadata access and related testing scenarios, while making the assumptions and limitations explicit for maintainers.
