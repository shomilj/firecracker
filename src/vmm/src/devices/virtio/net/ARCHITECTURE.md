# Paravirtual Network Device Architecture

This document describes the architecture of the virtio network implementation that bridges a guest operating system to the host using a Linux TAP interface, optional metadata-service detours, token-bucket rate limiting, and structured per-device metrics.

## Purpose & Boundaries

The subsystem’s responsibility is to emulate a virtio-compliant network adapter whose data plane moves Ethernet frames between guest memory and a host-side TAP device. It negotiates virtio-net features with the guest driver, maintains two virtqueues for receive and transmit, translates descriptor chains into scatter-gather views of guest memory, prepends or interprets virtio-net framing headers, and drives the TAP file descriptor with non-blocking bulk reads and writes. When configured, it can also intercept a narrow class of guest-originated frames intended for an in-VM metadata service and inject responses without sending those frames onto the physical path.

What this component explicitly does not do is own routing policy for the host, configure bridges or firewalls, implement TCP/IP for the host, or manage DHCP. Those concerns live outside the virtual machine monitor. The component also does not implement arbitrary packet filtering beyond metadata-service recognition and a best-effort Ethernet source address consistency check when a guest MAC is known.

If this component failed catastrophically, the guest would lose network connectivity for that adapter, virtio queue progress could stall, and interrupts might not fire correctly—breaking workloads that depend on the network. A partial failure on the TAP path surfaces as dropped or stuck traffic rather than silent guest memory corruption, because frame boundaries and descriptor completion are tied to explicit success or error handling at the device boundary.

## Interfaces & Contracts

The emulated device exposes the standard virtio network device contract to the guest: a small configuration space for the MAC when that feature is enabled, feature bits for checksum and segmentation offload, merged receive buffers, and event index notification suppression when negotiated. It consumes mapped guest memory from the hypervisor, an opened TAP handle with negotiated virtio header size and offload flags, two independent rate limiters for receive and transmit, and optionally a metadata network stack plus shared metadata storage.

Callers that construct the device must supply a valid TAP identity string within host naming limits, ensure the TAP can be opened exclusively, and provide rate limiters whose semantics align with the token model used elsewhere in the monitor. After activation, the guest driver must supply receive buffers that meet minimum length requirements derived from negotiated features—large enough for jumbo-class frames when segmentation offload is in use, or a smaller conventional MTU-oriented size otherwise, unless merged receive buffers are enabled in which case much smaller pieces can be chained.

The device guarantees non-blocking interaction with the TAP where the implementation relies on kernel non-blocking mode, best-effort completion of virtqueue operations for well-formed requests, and deterministic handling of malformed guest descriptors by completing them with zero length where appropriate rather than panicking. It also guarantees that rate limiting applies symmetrically in terms of token accounting: a logical operation is charged both an operations token and a byte token, and if byte accounting fails after operations were consumed, the operations token is rolled back for that attempt.

## Data Flow

Ingress from the host toward the guest begins with raw bytes arriving at the TAP file descriptor, optionally preceded by competition from the metadata service path which can synthesize a frame directly into a staging buffer when it has data to deliver. For TAP ingress, the implementation performs a vector read into guest-provided memory that has been aggregated from one or more descriptor chains depending on whether merged receive buffers were negotiated. Successful reads produce a length, populate the virtio-net header area, update merged-buffer counts in the header when multiple chains participate, and mark used descriptors so the guest driver can reclaim buffers.

Egress from the guest toward the host starts with the guest enqueueing transmit descriptor chains. The device maps those chains into a contiguous logical view, validates size limits, applies rate limiting, then inspects Ethernet headers after skipping the virtio-net prefix. Eligible frames may be absorbed by the metadata path; otherwise bytes are written to the TAP with a vector write that preserves guest layout. Metrics record successful and failed tap writes, malformed frames, and suspected MAC mismatches when a configured guest MAC is present.

The following diagram situates the major buffers and directions without naming implementation artifacts. Arrows show the dominant byte movement; the metadata branch is optional.

```
                    +------------------+
                    | Guest virtio-net |
                    | driver / queues  |
                    +--------+---------+
                             |
              RX buffers     |     TX chains
              (writeable)    |     (readable)
                             v
                    +--------+---------+
                    |  Descriptor map  |
                    |  to guest RAM    |
                    +--+-------------+--+
                       |             |
           +-----------+             +------------+
           | readv from TAP         | writev to TAP
           v                        v
    +------+------+           +-----+------+
    | Host TAP fd |           | Host TAP fd |
    +-------------+           +-------------+
           ^                        |
           |                        |
    optional injection        optional detour
           |                        |
    +------+------+            +----+-----+
    | Metadata    |            | Metadata |
    | response    |            | request  |
    | synthesizer |            | handler  |
    +-------------+            +----------+
```

Before the diagram, note that guest memory is always the source of truth for transmit contents and the sink for receive contents; the TAP is simply the host networking attachment. After the diagram, observe that metadata handling short-circuits the TAP for a subset of guest-originated frames and can inject frames toward the guest without an external packet arriving first, which is why the receive path checks for metadata output before attempting another TAP read when buffers exist.

## Control Flow

Execution is event-driven. Until the virtio device is activated, only a dedicated activation notification is registered with the runtime poller; queue notifications and TAP readability are ignored to avoid operating on uninitialized queues. Upon activation, the poller subscribes to receive-queue and transmit-queue kick events, both rate-limiter timer endpoints, and the TAP descriptor with edge-triggered input as appropriate. A one-time transition swaps the activation subscription for the runtime set.

When the receive queue signals, the handler drains the notification, pulls newly available descriptor chains into an internal receive staging structure, and if rate limiting permits, resumes reception. When the TAP signals readability, the handler likewise resumes reception while skipping work if the receive limiter reports blocked state. Transmit queue notifications drain the kick, and if the transmit limiter is not blocked, the handler processes as many transmit chains as available, stopping early if rate limiting refuses a frame and restoring the queue state so the frame can be retried later.

Rate limiter notifications exist to restart work when token buckets refill after a prior throttle. The receive-side path first attempts to complete any deferred receive completion that was held back solely due to rate limiting, then continues ingesting from TAP or metadata. The transmit-side path retries the transmit pump.

A simplified view of the critical path for a single incoming frame is: ensure sufficient staged receive capacity for a maximum-sized packet, preferentially service metadata if it has output, else read from TAP into guest staging, then account rate limits before committing used descriptors and signaling the guest if notification rules allow.

```
  [RX queue kick] -----> parse guest buffers
                              |
                              v
                      enough capacity?
                       /            \
                     no              yes
                     |                |
                     v                v
              (wait / count)     read metadata or TAP
                                        |
                                        v
                                 rate limit OK?
                                  /         \
                                no           yes
                                |             |
                                v             v
                          defer finish   complete used,
                          completion     signal if needed
```

The diagram above highlights branching on buffer capacity and on rate limiting. The transmit path is analogous but ordered differently: parse chain, enforce size, consume rate tokens, choose metadata detour or TAP write, then complete the transmit used entry.

## State & Lifecycle

The device holds virtio queue state including availability and used indices, negotiated feature bits, interrupt signaling state, and configuration such as the guest MAC when exposed. It owns the TAP handle, two independent rate limiters, a staging area for assembling receive-side descriptor coverage, a staging buffer for large ingress frames when pulling from TAP before scatter-writing into guest memory, a small header scratch region for transmit inspection, and a single transmit reassembly buffer view over guest memory for the duration of each transmit batch.

Lifecycle phases are: construction and TAP setup including virtio header size programming; inactive virtio state where only activation is wired; activation that initializes queues against guest memory, enables optional notification suppression, programs TAP offload flags consistent with guest capabilities, and records minimum receive buffer sizes; steady-state event processing; and optional snapshot restore paths that reconstruct rate limiters, queue state, and partial receive staging metadata.

Receive-side staging tracks how many descriptor chains are pinned, how many have been fully consumed for completion, and whether a frame is waiting to complete used descriptors because rate limiting deferred the final step. Transmit-side staging clears between batches so guest memory views do not alias across concurrent operations.

## Failure Modes

TAP open failures surface early and prevent device creation. IOCTL failures when setting header size or offload modes fail activation. Non-blocking TAP reads return a would-block style result when no packet is available; that condition is ordinary and stops a batch without being treated as fatal. Unexpected read errors escalate to a device-level error path because they indicate a broken host attachment.

Malformed guest descriptors on receive increment failure metrics and complete descriptors with zero length where possible so the guest can observe the failure mode. Oversized transmit frames are rejected as malformed. Missing virtio-net headers in transmit buffers are treated as malformed. If the guest exhausts internal staging capacity for descriptor chains, processing stops conservatively.

Rate limiting introduces a soft failure mode: work is not lost, but completion is deferred until tokens arrive. On transmit, the queue pop may be undone so the request remains available. On receive, partially processed frames can remain staged until the limiter allows completion. Metadata detours replenish transmit rate tokens because those frames never hit the TAP, preserving fairness between logical transmit operations and actual wire operations.

Silent corruption is unlikely if guest memory mapping and descriptor chaining invariants hold; violations in descriptor graph structure are expected to be caught by lower-level memory accessors. A violated assumption about buffer sizes relative to negotiated features could theoretically truncate or split frames incorrectly; the design mitigates this by enforcing minimum receive sizes unless merged buffers allow fragmentation.

## Operational Characteristics

Resource consumption centers on guest memory bandwidth for vectorized copies, syscall volume for TAP reads and writes, and event-loop wakeups from queue kicks, TAP readiness, and timer-based limiter refills. The largest single receive size is bounded by a fixed upper buffer size constant, which also drives the requirement that staged guest capacity meet that threshold before TAP reads proceed, preventing fragmentation attacks from causing undersized reads.

Scaling bottlenecks include the single-TAP throughput of the host, the cost of mapping and walking descriptor chains, and the sequential processing of transmit requests within a batch. Notification suppression reduces interrupt storms when the guest supports event indices.

Observability is implemented as a per-interface identifier keyed map of counters and latency aggregates serialized to JSON when the metrics writer flushes. Each device contributes a prefixed block named after its interface identifier, and an aggregate block rolls up differentials across devices for backward compatibility. Counters cover activation and configuration errors, queue and TAP event counts, rate-limiter-related events, throttle counts, byte and packet totals, malformed frame counts, descriptor processing failures, TAP read and write failures, aggregated TAP write latency, spoofed source MAC observations, and queue backlog hints. Some counters exist for partial memory operations that may be exercised by alternative call paths. The aggregate path does not currently participate in every store-wide metrics aggregation hook, which is a documented limitation.

## Design Rationale

Paravirtual networking trades hardware fidelity for efficiency: the guest cooperates via virtqueues instead of emulating PCI register churn per packet. TAP was chosen as the host attachment because it presents an Ethernet-oriented datagram interface that aligns with L2 framing expected by virtio-net, and because the Linux tun/tap driver supports virtio header metadata and offload negotiation that mirror guest capabilities.

Splitting receive and transmit rate limiters allows operators to shape guest traffic asymmetrically, reflecting typical WAN versus LAN bandwidth asymmetry and mitigating denial-of-service from one direction without starving the other. Pairing operations and byte tokens reduces both packet-per-second floods and bandwidth floods.

Optional metadata-service integration acknowledges that cloud instances often need a local control plane channel that should not traverse the external network. Prioritizing metadata output on receive avoids starvation when both TAP and metadata have work, at the cost of delaying pure host ingress briefly; the tradeoff favors control-plane responsiveness.

Merged receive buffers complicate descriptor bookkeeping but reduce guest memory pressure for small MTUs and enable chaining without allocating a single giant buffer. The implementation pays complexity in header bookkeeping and used-ring updates to gain flexibility for drivers that negotiate the feature.

Centralizing metrics in a registry keyed by interface identity avoids ambiguous ordering when multiple NICs are instantiated; the map preserves human-interpretable attribution in telemetry even if devices are created in varying orders.

---

**Diagram note:** The state-oriented sketch below summarizes receive completion deferral without using implementation-specific labels. Solid lines represent forward progress; dashed lines represent waiting on tokens.

```
       +----------------+
       | Frame ready in |
       | staging        |
       +--------+-------+
                |
                v
         rate limit allows
          completion?
           /         \
         no           yes
         |             |
         v             v
  +------+------+  +---+-----------+
  | Hold used     | Publish used  |
  | ring advance  | ring updates    |
  | until refill  | and kick guest  |
  +---------------+----------------+
```

Before this diagram, recall that receive ingress can read a frame successfully yet still delay guest-visible completion if the rate limiter lacks budget; the frame remains logically owned by the device until completion. After the diagram, note that timer-driven limiter events are the intended way to exit the held state without dropping the frame, preserving correctness under throttling.
