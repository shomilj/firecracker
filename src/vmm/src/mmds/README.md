# MicroVM Metadata Service (MMDS)

1. **Purpose and placement in the stack**

   1.1. The MicroVM Metadata Service exposes a compact HTTP surface—conceptually aligned with EC2 Instance Metadata Service (IMDS) usage patterns—so a guest can retrieve structured metadata (JSON) that the host populated. The service is not a general-purpose web server: it supports a narrow verb set, a fixed token-issuance path, and content negotiation between JSON and a plaintext “IMDS listing” form.

   1.2. Architecturally, three layers cooperate:

   - A **metadata datastore** holding arbitrary JSON, with size limits and merge semantics for host-driven updates.
   - An **HTTP decision layer** that maps parsed requests to datastore reads, token issuance, and version-specific authorization behavior.
   - A **minimal IPv4/TCP stack** embedded in the VMM that terminates connections toward a link-local address and bridges guest virtio-net frames into that stack; responses are injected back toward the guest on the same virtio path.

   1.3. High-level data flow (guest → host metadata):

   ```
        Guest stack                    VMM
       +-----------+                  +---------------------------+
       | HTTP      |  virtio TX       | Strip virtio-net header   |
       | client    +----------------->| Classify: MMDS vs TAP      |
       +-----------+                  | If MMDS: minimal TCP+HTTP |
                                      | Else: forward to TAP      |
                                      +-------------+-------------+
                                                    |
                                                    v
                                      +---------------------------+
                                      | Lock shared datastore     |
                                      | Route GET/PUT, version    |
                                      | Build HTTP response       |
                                      +-------------+-------------+
                                                    |
                                                    v
                                      +---------------------------+
                                      | Queue Ethernet reply      |
                                      | virtio RX (MMDS first)    |
                                      +---------------------------+
   ```

2. **Metadata datastore**

   2.1. The datastore is a single JSON value tree (object, array, or scalar depending on what the host installed). Semantically it is **untyped**: the service does not enforce a schema beyond JSON validity and byte-size limits.

   2.2. **Initialization and lifecycle**

   - The store begins uninitialized. A full document **replace** operation succeeds only if the serialized JSON size is at or below the configured byte ceiling; on success the store is marked initialized.
   - Until initialized, incremental **merge** operations fail with a dedicated “not initialized” condition so partial patches cannot create state from nothing.

   2.3. **Host-visible update paths (outside the guest HTTP path)**

   - The control plane can **replace** the entire document in one shot (subject to the same size check).
   - The control plane can **merge** a patch document using JSON Merge Patch semantics (RFC 7396). The merge runs against a clone of the tree; only if the merged result still fits under the byte limit is the clone committed. That pattern prevents a patch from partially applying and then failing size validation after mutating live state.

   2.4. **JSON Merge Patch behavior (deep detail)**

   - If the patch root is a JSON object, the algorithm walks key-by-key. For each key in the patch object:
     - If the patch value is JSON `null`, the key is **removed** from the target if present.
     - Otherwise the merge recurses: if the key is missing in the target, a `null` placeholder is inserted first so nested structures can grow organically.
   - If the patch root is **not** an object (array, string, number, boolean, or null), the entire target document is **replaced** by a deep copy of the patch root—this is a full-document overwrite at the merge entry point, not a field-wise merge.
   - If the patch root is an object but the current target is not, the target is first replaced with an empty object, then the merge proceeds—so a non-object root can be “pulled up” into object form before applying nested keys.
   - Arrays merge **by replacement** when the patch addresses them with a non-object value; there is no array-index merge patch logic beyond what JSON pointer navigation yields on read.

   2.5. **Read model: JSON pointer paths and trailing slashes**

   - Guest requests use the HTTP absolute path as a **JSON Pointer**-style address (leading slash, segments separated by `/`). Repeated slash sequences in the path are collapsed before lookup so clients that accidentally double path separators still resolve consistently.
   - If the path ends with `/` and points to an object, the trailing slash is stripped before pointer resolution—this interacts with how pointers address objects vs. ambiguous “directory” style paths.
   - Successful reads return either:
     - **JSON**: the `serde_json` string for the subtree or scalar at that path, or
     - **IMDS plaintext**: a newline-separated listing when the value is an object (each key on its own line; keys whose values are objects get a trailing `/` to mimic directory listings), or the raw string content when the value is a JSON string. Other scalar types (numbers, booleans, arrays as leaf selections, etc.) cannot be rendered in IMDS mode and surface as “unsupported type” for that format.

3. **Versioning: V1 vs V2 policy**

   3.1. The datastore tracks a **policy version** independent of the JSON contents. The version controls how strictly session tokens gate **GET** requests.

   3.2. **Version 1 (compat / permissive)**

   - GET requests are always evaluated against the datastore after URI normalization.
   - If a session token header is **absent**, a metric counter records “no token” but the request still succeeds if the path exists—guests are not forced to obtain a token.
   - If a token is present but **invalid or expired**, another metric records “invalid token,” yet the GET still proceeds. This matches legacy expectations where tokens were advisory for observability rather than enforcement.

   3.3. **Version 2 (session-token enforced)**

   - GET requires a token supplied in one of two equivalent header names (case-insensitive match against custom header maps).
   - If no token header is present: the response is **401 Unauthorized** with a plain-text body instructing the client to supply a token; a “no token” metric increments.
   - If a token is present but fails validation: **401 Unauthorized** with an invalid-token body; an “invalid token” metric increments.
   - Only after the token validates does the GET proceed to path resolution and content negotiation.

   3.4. **Operational implication**: switching from V1 to V2 is a breaking change for guests that relied on unauthenticated reads; switching the other way reopens reads without valid tokens.

4. **Session tokens and TTL flow**

   4.1. Tokens are **opaque strings** from the guest’s perspective. Internally they bundle a random nonce, an AES-GCM ciphertext of an expiry timestamp, and an authentication tag. They are serialized to a compact binary form, then base64-encoded for transport.

   4.2. **Issuance (HTTP PUT on the token path)**

   - Only the HTTP **PUT** method is accepted for token creation, and only when the sanitized path matches the fixed token endpoint. Any other PUT path yields “not found” for the resource layer.
   - The client must supply a **time-to-live** in seconds using one of two supported header aliases (same numeric semantics). Missing TTL → **400** with guidance text. Non-numeric TTL → **400** with header validation error text.
   - TTL must fall within an inclusive bounded range (minimum one second, maximum six hours). Out-of-range values return **400** with an explicit range hint.
   - On success the response body is **plain text** containing only the new token string.

   4.3. **Cryptographic model**

   - A 256-bit AES key is drawn from the host OS random source at authority construction. Each issued token uses a fresh 12-byte IV.
   - The **expiry instant** is computed in monotonic wall-time milliseconds: current monotonic time plus TTL converted to milliseconds. That value is encrypted as the payload; the tag authenticates both the ciphertext and the **additional authenticated data (AAD)** string.
   - The AAD defaults empty until the control plane binds an instance identifier; afterward AAD becomes a stable prefix plus the instance id. Binding AAD ensures tokens minted for one microVM identity do not decrypt under another configuration if keys were ever reused—operators rotating instance identity should expect old tokens to fail decryption.

   4.4. **Validation**

   - Incoming token strings longer than a small fixed character budget are rejected without decoding work—an anti-DoS guard.
   - Base64 decoding, bounded binary deserialization, and decryption must all succeed. Failure at any step is indistinguishable to the caller: the token is simply invalid.
   - After decryption, expiry (milliseconds) is compared to **current monotonic time in milliseconds**. Tokens are valid only while `expiry > now`; equality or past times mean expired.

   4.5. **Key rotation edge case**

   - The encryptor tracks how many tokens have been issued under the current AES key. If an internal counter would wrap at 32 bits, the implementation generates a **new random key**, resets the counter, and logs a warning. **All previously issued tokens become invalid instantly** because the old key is discarded. This is an extreme edge case; well-behaved workloads should never hit it, but automation should be prepared to re-fetch tokens if issuance suddenly starts failing after such a rotation.

5. **HTTP routing and responses**

   5.1. **Allowed methods**

   - **GET** for metadata reads (subject to version policy).
   - **PUT** exclusively for token issuance on the dedicated path.
   - Any other method receives **405 Method Not Allowed** with `Allow` headers listing GET and PUT only.

   5.2. **URI validation**

   - An empty absolute path yields **400 Bad Request** (“invalid URI”) before any datastore access.

   5.3. **Content negotiation**

   - The HTTP layer inspects the `Accept` header. JSON vs plaintext IMDS listing mode is selected accordingly; defaults follow the embedded HTTP library’s rules when `Accept` is absent or empty (typically plaintext).

   5.4. **GET error mapping**

   - Unknown JSON pointer / missing subtree → **404** with a body referencing the missing resource path.
   - IMDS formatting impossible for the value type → **501 Not Implemented** with an explanatory body.
   - Document would exceed size limits (should not happen on read if limits were enforced on write) → **413 Payload Too Large** with the datastore error text.

   5.5. **PUT safeguards**

   - Token issuance rejects requests carrying `X-Forwarded-For`, including case variants. That header is treated as unsupported on this path to avoid ambiguous client-address semantics in a context where the server is not a full HTTP proxy. The error surfaces as a bad request tied to header validation.

6. **Integration with the minimal TCP stack and virtio-net path**

   6.1. **Frame classification**

   - Incoming Ethernet frames are inspected at L2/L3 without claiming to be a full router: ARP frames targeting the configured MMDS IPv4 address are recognized; IPv4 frames whose destination matches the MMDS address are candidates. Other frames are not MMDS traffic.

   6.2. **ARP handling**

   - ARP requests for the MMDS IPv4 capture the requester’s MAC and IPv4; the stack later emits a reply mapping the MMDS Ethernet address to that IPv4. This establishes L2 reachability from the guest’s perspective toward the link-local service address.

   6.3. **IPv4/TCP**

   - Only TCP segments to the configured TCP port are fed into the user-space TCP handler. IPv4 checksum verification may be skipped on ingress under the assumption that virtio offloading or environment specifics make verification optional—an operational caveat if corrupted packets ever reach the handler.
   - The TCP endpoint accepts connections and parses HTTP requests via the lightweight parser. For each completed request, a callback invokes the HTTP decision layer with a shared **mutex-protected** datastore handle, builds a response, and returns it through the TCP state machine.

   6.4. **Virtio-net detour**

   - On **transmit** from guest to host: after stripping the virtio-net header, if an MMDS network stack is configured for that device and the L2 frame is classified as MMDS-bound, the full frame bytes are copied into the MMDS path. The frame is **not** written to the TAP device. The transmit rate limiter’s budget is **replenished** for that frame as if it were not consumed—MMDS traffic is intentionally excluded from TX byte accounting.
   - If the frame was consumed by MMDS and the receive path has no partially delivered RX data, the device immediately tries to **pull responses** from MMDS before TAP—MMDS replies take priority over host network reads.
   - On **receive** toward the guest: the device first asks the MMDS stack whether it has a pending Ethernet frame (ARP reply or TCP segment). Only if none is available does it read from TAP. Replies are prefixed with the virtio-net header and written into guest RX descriptors.

   6.5. **Concurrency model**

   - The datastore is shared behind a mutex. HTTP handling takes the lock for the duration of request processing; TCP runs on the device’s execution context, so pathological slow clients hold the lock and can delay other readers—acceptable for a metadata plane but worth noting for denial-of-service analysis.

7. **Persistence and migration**

   7.1. **What snapshot layers record**

   - For each virtio network device that had MMDS enabled, persistence captures the **MMDS network stack configuration**: the Ethernet address used as the service MAC, the IPv4 address, and the TCP port. Restoring reconstructs a fresh TCP handler with that configuration bound to the same shared datastore `Arc` supplied by the restoring constructor.
   - The **policy version** (V1/V2) is captured at the device-manager level when older snapshots omit it but MMDS is present, inferring version by reading the live datastore lock at save time.

   7.2. **What is not serialized as part of these structures**

   - The token authority’s random AES key material and encrypted-token counters are **process-local**; a new process loads new entropy. After migration or resume, **all previously issued session tokens become invalid** even if wall-clock time has not reached their nominal TTL.
   - The JSON document itself is managed through host API initialization paths rather than the MMDS network snapshot structs described here; operators should assume metadata content and tokens need to be **re-established** around snapshot resume unless a higher-level orchestration layer reapplies them.

8. **Limits**

   8.1. **Datastore byte ceiling**

   - Configurable upper bound on serialized JSON size for replace and merge operations. Breaches return a dedicated limit error to the control plane; guest GET paths surface that as HTTP 413 if ever encountered on read (unexpected if writes were guarded).

   8.2. **TCP connection fan-in**

   - The embedded TCP handler enforces maximum concurrent connections and maximum pending reset segments; exceeding these manifests as connection failures or dropped work at the TCP layer rather than HTTP-layer errors.

   8.3. **Token string length**

   - Overly long token strings are rejected outright before decode.

9. **Security and operational edge cases**

   9.1. **Lock poisoning**

   - If the mutex protecting the datastore is poisoned (a panic while holding the lock), request handling will panic on the next access—there is no recovery path. This is a catastrophic failure mode for the whole VMM thread handling the device.

   9.2. **Authentication vs. authorization**

   - Tokens prove **freshness** and **integrity** of the session handoff, not fine-grained authorization inside the JSON tree. Any holder of a valid token can read the entire tree visible through GET in V2; there is no per-path ACL.

   9.3. **V1 “soft” enforcement**

   - Metrics may show invalid or missing tokens while the guest still receives data—monitoring should not assume metric spikes imply user-visible failure in V1.

   9.4. **Checksums and trust**

   - Skipping IPv4 checksum verification on inbound MMDS-bound packets trades strict validation for compatibility with offload assumptions; corrupted packets could theoretically desynchronize TCP state until sequence checks fail.

   9.5. **Rate limiting interaction**

   - MMDS TX bypasses TAP byte accounting and replenishes the TX limiter when detouring, so metadata storms do not consume guest→host TAP bandwidth budgets; RX toward the guest still participates in normal virtio RX processing and any configured inbound limiting.

   9.6. **Shared datastore across NICs**

   - Multiple virtio devices can reference the same `Arc` datastore when the control plane wires them that way; all share one token authority and one JSON document. Isolating tenants at the metadata layer requires separate VMs or separate orchestration of datastore handles.

   9.7. **Monotonic time and TTL**

   - Expiry uses monotonic clock milliseconds, not wall clock—NTP step changes do not shorten or extend token life in wall-clock terms. Very long TTLs still cap at six hours by policy.
