# MicroVM Metadata Service — Architecture

This document describes the architecture of the in-process metadata service that exposes guest-readable instance metadata through a constrained HTTP surface, backed by a JSON document store, and integrated with the virtual machine’s minimalist TCP and IPv4 implementation. The description focuses on behavior, contracts, and failure characteristics rather than implementation identifiers.

---

## Purpose & Boundaries

The metadata service exists to give workloads running inside a micro virtual machine a controlled way to read structured metadata that the control plane and host have placed into a shared document, without granting those workloads arbitrary network reachability or a full general-purpose HTTP server inside the guest environment. Conceptually it is the guest-facing half of a larger metadata facility: the same logical document can also be inspected and mutated from the trusted host side through the virtual machine monitor’s API, but the code under discussion here is concerned with how the guest interacts with that document when network frames arrive on a configured interface.

Responsibility is limited to interpreting a narrow subset of HTTP semantics on top of a purpose-built transport stack, enforcing optional session-token rules that mirror cloud instance metadata conventions, mapping request paths into a single hierarchical JSON value, and serializing matching fragments back as HTTP responses. It does not implement a generic reverse proxy, TLS termination, full HTTP compliance, routing to arbitrary backends, or any path that bypasses the virtual network device detour. It also does not define how the host populates or patches the document; that is upstream configuration. If this component failed outright, guests that rely on link-local metadata queries would lose access to that document while the rest of the virtual machine might continue to run; host-side API access to the same document might still work depending on how the larger system wires storage and API handlers.

What lies explicitly outside this module’s scope includes the virtio socket device used for unrelated host–guest channels, conventional forwarding of Ethernet frames to the host tap interface, and congestion algorithms or features typical of production TCP stacks on the public internet. The design intentionally accepts those boundaries to keep the attack surface and implementation complexity small.

---

## Interfaces & Contracts

**Guest-visible HTTP contract.** Clients are expected to speak HTTP/1.x at a basic level. Only two methods are accepted for the guest path: retrieval of metadata and a dedicated token-minting operation. Any other method receives a refusal with an indication of allowed methods. Retrieval requests must carry an absolute path that can be normalized into a pointer into the JSON tree; empty paths are rejected. Responses use standard status families: success for found content, “not found” when the path does not resolve, “payload too large” when internal size limits would be exceeded, “not implemented” when a chosen representation cannot be produced for the value’s type, and unauthorized variants when a configured security mode requires a session token that is missing or invalid.

**Content negotiation.** Retrieval honors a simple accept negotiation: one mode returns JSON text for subtree or leaf values, and another returns a plaintext listing style compatible with common instance metadata user experiences. When the stored value cannot be expressed in the plaintext style, the service responds with a not-implemented style error rather than silently coercing types.

**Token minting contract.** A dedicated path accepts a state-changing request that does not alter the metadata document itself but asks the authority to issue a short-lived session secret. That operation requires a second header carrying an integer lifetime in seconds within a fixed allowable range. Two header spellings are accepted for compatibility with different client ecosystems. If the lifetime header is missing, malformed, or out of range, the request fails with a client error. For defensive reasons, a header that could be used to spoof client origin for downstream proxies is explicitly rejected on this operation even though ordinary metadata retrieval ignores it.

**Session token presentation on reads.** Two alternative header names are recognized for supplying the session secret on reads, compared case-insensitively against the set of known names. Depending on a configurable version flag, presence of a token may be optional (with metrics recording absence or invalidity) or mandatory (with unauthorized responses when missing or invalid).

**Internal contract with the transport stack.** The HTTP layer exposes a pure request-to-response transformation given a handle to the shared document and token state. The transport stack is responsible for byte-stream framing, connection lifecycle, and never calling the handler with partial or malformed requests beyond what it can recover from. The handler side assumes well-formed requests from the parser and returns complete responses whose bodies are sized within the limits implied by the document store.

**Upstream contracts.** The document store accepts replacement and merge-patch updates from the trusted configuration path; those operations enforce a serialized size cap. The token machinery relies on the platform monotonic clock for expiry comparison and on a source of cryptographic randomness for keys and nonces. An optional binding to a stable virtual machine identifier mixes into authenticated encryption so that secrets are scoped to the instance context when configured.

The diagram below situates the metadata HTTP semantics relative to the virtual network data path. The important invariant is that metadata traffic is intercepted on the paravirtualized NIC path before frames would otherwise reach the host tap, which is a different conduit from virtio-based socket devices used elsewhere in the stack.

```
                    +-------------------+
                    |   Guest workload  |
                    |  (HTTP client)    |
                    +---------+---------+
                              | Ethernet frames on virtio-net
                              v
                    +---------+---------+
                    | Device model      |
                    |  RX path          |
                    +---------+---------+
                              |
              +---------------+---------------+
              |                               |
              v                               v
    +-------------------+           +-------------------+
    | Metadata stack    |           | TAP / upstream    |
    | (ARP, TCP, HTTP)  |           | L2 path           |
    +---------+---------+           +-------------------+
              |
              v
    +-------------------+
    | Shared JSON store |
    | + token authority |
    +-------------------+
```

After traffic classification, frames destined for the metadata address are consumed by the dedicated stack; all other frames follow the normal tap-backed path. This separation is what allows a minimalist TCP implementation to coexist with ordinary guest networking.

---

## Data Flow

Metadata enters the system from the host side as JSON configured through the control API: the entire tree is replaced or merged according to patch semantics that remove keys when null appears in the patch, recursively merge objects, and replace scalars as needed. That population path is trusted and does not go through the guest HTTP surface. Once loaded, the in-memory tree is the authoritative source for guest reads.

Guest retrieval begins with TCP payload bytes reassembled into an HTTP request. The path component becomes a logical pointer into the JSON tree after normalization that collapses redundant slashes. The handler walks the tree; if a match exists, the selected subtree or leaf is rendered either as JSON or as the plaintext listing format. If the path ends with a slash, the implementation adjusts pointer resolution to match dictionary listing expectations. Nothing is written back to the store on a successful read.

Token minting flow pulls no body from the client for the minting request itself. The handler validates headers, asks the token authority for a new secret with the requested lifetime, and returns the secret as a plaintext body. The secret is an encoded, authenticated ciphertext over an expiry timestamp, not an opaque database row.

Downstream of the handler, response bytes pass back through the same TCP connection object, then IPv4, then Ethernet framing, and finally into the guest receive path. Metrics counters track reception, transmission, connection creation and teardown, and token-related anomalies.

The following diagram summarizes the read path from frame ingress to response egress.

```
  Guest TCP segment
        |
        v
  Reassemble byte stream
        |
        v
  Detect complete HTTP request
        |
        v
  +-----+-----+
  | Optional  |---- missing/invalid ----> 401 (in strict mode)
  | token     |
  | check     |
  +-----+-----+
        | valid / mode allows
        v
  Resolve JSON path --> 404 if missing
        |
        v
  Format body (JSON vs listing) --> 501 if type unsupported
        |
        v
  Build HTTP response --> TCP segments --> IPv4 --> Ethernet --> guest RX
```

Before this diagram, the narrative role of each stage matters: classification happens entirely inside the monitor; the guest only sees ordinary TCP and HTTP behavior. After the diagram, the takeaway is that there is no secondary persistence layer on the read path—the response is always a pure function of the current in-memory tree plus token validity.

---

## Control Flow

Execution is driven by the virtual device model when network frames arrive and when transmit opportunities exist. On receive, a cheap pre-check decides whether a frame might be aimed at the metadata IPv4 address or an ARP query for that address. Non-matching frames are not handled here. ARP handling records the guest’s addressing information so replies can be addressed correctly. IPv4 handling forwards TCP segments into a small connection table with limits on concurrent sessions and pending reset bookkeeping.

When a TCP stream delivers enough bytes to complete one HTTP request, the receive path invokes the metadata response builder with the parsed request. That function branches first on HTTP method, then on configured version behavior for reads, then on path for token minting versus ordinary retrieval. Errors short-circuit to HTTP responses without touching the document. Success paths lock the shared store briefly for consistency during the read or token update.

Transmit control interleaves ARP replies ahead of TCP data when both are pending, preserving predictable neighbor discovery behavior. If the inner TCP state machine has nothing to send and no ARP reply is queued, the device model falls back to reading from the tap interface, which keeps latency predictable for normal traffic.

Branching points worth emphasizing are: method disallowed versus allowed; read path requiring a token versus accepting anonymous reads; token valid versus invalid; JSON path found versus not found; output format feasible versus unsupported value type; minting path hitting header validation versus successful issuance.

---

## State & Lifecycle

Owned state splits into three conceptual areas: the JSON document and initialization flag, the token authority including cipher material and issuance counts, and the network stack object per interface that holds addressing, TCP handler state, and a reference to the shared store.

The document begins empty and uninitialized until the first successful host-side load; some guest operations treat uninitialized storage as missing data rather than an empty object. The size limit applies to serialized length, and both wholesale replacement and patch operations verify the post-update size before committing.

Tokens are generated with bounded lifetimes measured against a monotonic clock. The authority rotates its underlying symmetric key if an extremely large number of tokens have been issued under one key, which invalidates all outstanding tokens at that transition—a rare edge case documented as acceptable because clients are expected to request fresh tokens.

Per-connection state inside the minimalist TCP layer includes receive buffers, send state, and at most one outstanding response per connection. Endpoints are torn down on invalid requests that exceed buffer limits, on clean close signals, or on reset handling.

At snapshot time, the network-facing object can be serialized to capture its Ethernet address, IPv4 configuration, and listening port, while the document version configuration may be captured elsewhere in the device snapshot machinery so restores stay consistent. Restoration reconstructs the network stack and reconnects it to the shared store provided by the resource layer. Startup ordering ensures the store exists before any interface detours traffic to the handler. Shutdown follows general device teardown: connections cease, and no durable flush of the document is implied by the guest stack itself—that remains a host concern.

A small state-oriented view of token validity looks like this:

```
        +-------------+
        | No token    |
        | presented   |
        +------+------+
               |
     +---------+---------+
     | strict read mode  |
     v                     v
 Unauthorized          Continue read
 (v2 behavior)         (v1 may allow)

        +-------------+
        | Token       |
        | presented   |
        +------+------+
               |
     +---------+---------+
     |                     |
     v                     v
 Decrypt + compare      Invalid / expired
 to expiry time         -------------------> Unauthorized
     |
     v
 Valid window
     |
     v
 Continue read
```

This figure is not a full TCP state machine; it only captures the decision layer applied after HTTP parsing. Following the diagram, the key lifecycle fact is that token validity is evaluated entirely inside the monitor on each request, with no server-side session table.

---

## Failure Modes

Malformed Ethernet or IPv4 frames are dropped or ignored with receive error metrics; they do not corrupt the document. TCP segments that overflow receive windows or exceed maximum request size lead to connection reset rather than partial handling, which avoids request smuggling across buffer boundaries.

HTTP layer failures map to explicit status codes: bad request for unusable paths or disallowed headers on the minting endpoint, method not allowed for unsupported verbs, not found for bad JSON pointers, payload too large when limits bite, not implemented for type and format mismatch. Token failures return unauthorized responses in strict mode; in lenient mode, invalid tokens may still be counted for observability while the read proceeds.

Cryptographic failures during token validation fail closed: any decoding, authentication, or length anomaly treats the token as invalid. Extremely long token strings are rejected without expensive processing.

If the mutex guarding the store is poisoned from a panicking thread elsewhere, the current implementation does not recover gracefully—that is an invariant violation of the hosting process rather than an expected runtime condition.

Silent corruption is unlikely if invariants hold: the document is only mutated through validated host paths and merge logic, and reads do not write. The main assumption that could cause surprising behavior is clock movement: expiry uses monotonic time, so host sleep or live migration semantics depend on how the platform exposes that clock across snapshots.

---

## Operational Characteristics

Work is proportional to incoming frame rate and connection count. Each virtio-net interface that enables metadata detour pays classification cost on every received frame before the tap path, so mis-tuned workloads that flood irrelevant traffic still incur the heuristic checks unless frames are clearly not addressed to metadata.

The TCP stack deliberately omits congestion control suited for WANs; it is tuned for a reliable, low-latency virtual link. Connection caps prevent unbounded memory use from parallel clients. Transmit prioritization of ARP over bulk TCP data avoids common stall patterns when guests begin conversations.

Observability is primarily through counters: frames accepted and errored, TCP connection churn, token absence and invalidity on reads, and unusual non-TCP IPv4 packets hitting the metadata address. Logging exists for rare key-rotation events in the token authority.

Scaling limits are single-VM: one logical document, bounded size, bounded connections per interface, and bounded token issuance rate implied by client behavior. There is no horizontal sharding inside this layer.

---

## Design Rationale

The problem is to expose EC2-like instance metadata semantics to unprivileged guests without embedding a full network stack in the guest or forwarding guest traffic to a host service over a socket that would complicate isolation. Implementing a tiny TCP/IPv4/Ethernet path inside the monitor lets the guest use normal link-local addressing and off-the-shelf HTTP clients while Firecracker retains complete control over parsing, authentication policy, and data shaping.

HTTP was chosen as the wire representation because ecosystem tooling already speaks it for metadata services. Restricting methods and headers keeps the parser small and auditable. JSON Merge Patch on the host side matches common configuration practices and avoids custom diff protocols.

Session tokens mirror IMDSv2-style defense in depth: even if a workload misroutes traffic, guessing paths is harder without a fresh secret when strict mode is enabled. Versioned behavior allows older compatibility where tokens were advisory while newer deployments can require them. Authenticated encryption binds expiry into the token so the service remains stateless regarding sessions beyond its keys and counters.

Pairing this layer with virtio-net interception rather than a separate virtio channel avoids duplicating metadata-specific devices in the guest and reuses driver paths guests already have. The dumb stack name reflects a deliberate trade: correctness for the constrained on-link scenario over generality.

Tradeoffs accepted include incomplete HTTP, no TLS on this path (relying on namespace isolation and link-local scope), no IP fragmentation support, and abrupt invalidation of all tokens if the issuance counter ever rolls through its guard—deemed acceptable given the astronomical timescale and recovery pattern of simply minting new tokens.

Together, these choices yield a small, inspectable surface that maps cleanly onto Firecracker’s threat model: the guest receives only what the JSON tree and token rules allow, the host retains authoritative control of content, and the data plane stays inside the VMM process unless explicitly bridged by other configuration.
