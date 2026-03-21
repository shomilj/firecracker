# Virtio Balloon, Guest Entropy, and Generated Protocol Constants

This document describes three related concerns inside the virtual machine monitor’s virtio device layer: dynamic memory reclamation through the virtio memory balloon, cryptographic-quality random byte delivery through the virtio entropy device, and mechanically produced protocol constants that keep multiple virtio implementations aligned with the same wire-level definitions. The balloon and entropy devices are full virtio peripherals with queues, interrupts, activation, persistence, metrics, and event-driven processing. The generated constants module is not a device; it is shared vocabulary that other virtio devices import when negotiating features, interpreting descriptors, or matching layout to the specification.

The following sections apply the same eight-part structure required for architecture notes in this repository. Within each section, subsections separate the balloon concern, the entropy concern, and the generated constants concern so that responsibilities do not blur.

---

## Purpose & Boundaries

### Memory ballooning

The memory balloon implements the standard virtio balloon model for Firecracker-style micro-virtual machines. Its job is to let the guest operating system voluntarily surrender physical pages back to the host by inflating the balloon, and to acknowledge requests to return pages to the guest by processing deflate traffic. When statistics are enabled, it also coordinates periodic guest memory statistics through a dedicated queue and a timer so the control plane can observe pressure, swap behavior, and cache-related quantities without probing guest memory directly from the host.

What the balloon does not do is silently change guest memory layout without cooperation from the guest driver. It does not allocate new backing store for the virtual machine; it removes or relinquishes host mappings for ranges the guest has identified. It does not implement general balloon policy for the orchestrator; it exposes mechanisms and obeys configuration. If this component fails during steady operation, memory overcommit and host-level reclaim strategies lose a primary lever: the virtual machine may retain more resident memory than desired, and live migration or density targets become harder to meet.

### Guest entropy

The entropy device exists to supply unpredictable bytes to the guest through virtio queues, using the host’s cryptographic random number generator as the source. Its purpose is operational: guests need entropy for keys, session identifiers, and various protocol stacks. The device is deliberately narrow. It does not interpret what the guest does with the bytes, does not guarantee a particular statistical test profile beyond what the host generator provides, and does not replace guest-side mixing or user-space entropy daemons.

If entropy delivery fails or is throttled excessively, guests may block in boot or runtime paths that wait for randomness, or they may fall back to weaker sources if misconfigured. The device’s responsibility is bounded to faithful virtio handling, rate limiting, and safe memory writes.

### Generated protocol constants

The generated module exists to centralize numbers and layout artifacts that originate from the same upstream virtio definitions used across the ecosystem. It is not responsible for device behavior, queue processing, or guest-visible policy. Its failure mode is subtle: if constants drift from the specification or from what other components assume, feature negotiation can diverge, leading to guests enabling features the monitor does not implement correctly, or rejecting combinations that would have worked. The module’s purpose is consistency and auditability, not runtime logic.

---

## Interfaces & Contracts

### Memory ballooning

The balloon presents itself as a virtio device with a small configuration space that tracks target and actual balloon sizes in page units, multiple virtio queues for inflate, deflate, and optionally statistics, and feature bits that enable deflate-on-memory-pressure behavior and statistics reporting. Toward the rest of the monitor it exposes configuration updates for target size and statistics polling interval, accessors for summarized statistics, and hooks for snapshot save and restore that capture virtio state together with balloon-specific fields such as outstanding statistics descriptors.

Callers must activate the device with a valid guest memory map before queue processing is meaningful. The balloon assumes the guest driver follows descriptor semantics: inflate descriptors supply read-only page frame lists, deflate descriptors are acknowledged even though the heavy lifting of returning memory may be guest-side, and statistics buffers follow a tagged layout. The balloon guarantees that obviously malformed descriptors are skipped or rejected without panicking in normal paths, that used-ring updates and interrupts follow virtio expectations when work completes, and that configuration change notifications use the appropriate interrupt type when the target size changes on an active device.

### Guest entropy

The entropy device exposes a single virtio queue for requests, empty configuration read and write behavior, and feature negotiation limited essentially to modern virtio versioning. It depends on a pluggable rate limiter that constrains both operation count and byte volume so that a guest cannot exhaust host entropy generation or CPU by flooding requests. The device consumes guest memory only through descriptor chains that describe writable buffers where random bytes will be written.

Callers must supply a rate limiter configuration consistent with product policy. The device guarantees that write-only buffers receive bytes up to the negotiated limit, that zero-length requests complete without touching random generation, and that read-only or malformed chains fail in a controlled way with used lengths reflecting failure where appropriate.

### Generated protocol constants

The generated layer exposes numeric constants grouped by concern: transport-wide feature bits, block-device opcodes and feature bits, network header structures and feature bits, and ring-related flags. Consumers depend on generated output remaining aligned with the project’s header-import tooling; the contract is that these values track the virtio specification sources the tool ingests.

The module explicitly does not provide behavioral implementations. Downstream virtio devices are responsible for only advertising features they implement and for using structural types when marshaling headers on the wire.

---

## Data Flow

The overall pattern is that guest drivers place work on virtio queues, the monitor observes readiness through event notifications, and devices read or write guest memory through validated mappings. Balloon and entropy differ in what crosses the boundary: the balloon moves ownership intent about physical pages and statistics records, while entropy moves random bytes into guest buffers.

The diagram below situates the three concerns relative to the guest, the virtio transport, and host services.

```
                    +-------------------+
                    |   Guest driver    |
                    +---------+---------+
                              |
                    virtio queues / config
                              |
                    +---------v---------+
                    |  Virtio device    |
                    |  implementation   |
        +-----------+---------+---------+-----------+
        |                     |                   |
        | inflate/deflate/    | entropy queue     | feature bits &
        | stats (balloon)     |                   | wire constants
        |                     |                   |
+-------v-------+     +-------v-------+   +-------v-------+
| Guest memory  |     | Guest buffers |   | Other virtio  |
| page lists &  |     | (writable)    |   | devices and   |
| stat records  |     +-------+-------+   | negotiation   |
+-------+-------+             |           +---------------+
        |                     |
        v                     v
+---------------+     +---------------+
| Host mapping  |     | Host crypto   |
| reclaim and   |     | random bytes  |
| memory advice |     | from kernel   |
+---------------+     +---------------+
```

Before the diagram, note that all three paths ultimately originate in guest-issued virtio work items or configuration reads. The balloon path translates page frame numbers into host actions on guest RAM regions. The entropy path fills guest buffers from the host generator. The constants path is not a runtime data plane; values flow at compile time into feature masks and structure layouts.

After the diagram, consider directionality. For the balloon, data enters as lists of page frame numbers and tagged statistics structures in guest memory. The implementation transforms page frame lists into contiguous ranges, applies host operations to release backing, and updates local summaries of statistics. Data exits as used-ring entries, interrupts, and updated configuration visible to the guest. For entropy, data enters as descriptor chains pointing at destinations; data exits as random bytes in those destinations plus used lengths and interrupts. For generated constants, there is no runtime I/O; the “flow” is import-time propagation into feature negotiation and packet header construction elsewhere.

---

## Control Flow

Execution is event-driven. Both balloon and entropy integrate with the monitor’s event loop: an activation handshake registers either a one-shot activation channel or the full runtime set, after which queue notifications and device-specific timers or rate limiters drive work.

The next diagram sketches the balloon’s critical path branches.

```
                    [Event dispatch]
                           |
              +------------+------------+
              |            |            |
        [Activate]   [Inflate q]   [Deflate q]
              |            |            |
              |            v            v
              |     pop descriptors   pop & ack
              |            |            |
              |            v            |
              |    gather page lists   |
              |            |            |
              |            v            |
              |    compact & reclaim   |
              |            |            |
              +------------+------------+
                           |
              +------------+------------+
              |            |
        [Stats q]    [Stats timer]
              |            |
              v            v
        read stats    complete buffer
        from guest    & interrupt
```

Before the diagram, activation is the gate: until guest memory is bound and queues are initialized, queue events are ignored or treated as spurious. Inflate processing loops until the available ring is exhausted, with an inner pattern that may pause when internal accumulation buffers fill, then compact and reclaim, then resume. Statistics processing reads structured records from guest memory into a cached aggregate and defers completion until a periodic timer fires, at which point the pending descriptor is returned on the used ring.

After the diagram, note that deflate handling is intentionally lightweight in this implementation: descriptors are acknowledged to unblock the guest even though the semantic “return pages” work is largely guest-side policy. Control flow for entropy is simpler: queue notifications drain available descriptors in order, optionally stopping early when rate limits block, and rate-limiter timer events revisit queued work when budget returns.

---

## State & Lifecycle

### Memory ballooning

The balloon maintains virtio state including negotiated features, queue snapshots, interrupt status, and activation status. It tracks a compact configuration space mirror, optional statistics polling interval, a monotonic cache of the latest statistics snapshot, and bookkeeping for a deferred statistics descriptor index when the timer has not yet completed the second phase of the protocol. A reusable buffer holds page frame numbers during inflate processing to amortize sorting and range detection.

Lifecycle begins with construction from initial sizing and policy flags. Queues are sized according to product limits, and the statistics queue may be absent entirely when polling is disabled, matching the specification’s expectation that optional virtqueues are omitted rather than left idle. Activation wires guest memory into queues and signals readiness through an activation event so the event loop can switch from registration of the activation channel to registration of runtime sources. If statistics are enabled, a periodic timer is armed. Snapshot save serializes virtio transport state together with balloon-specific fields. Restore reconstructs queues from saved memory metadata, reapplies configuration, and restarts the statistics timer when appropriate.

### Guest entropy

The entropy device holds virtio transport state, a rate limiter instance, and a scratch area for gathering guest buffer contents into a form suitable for vectorized write-back. Lifecycle parallels other virtio devices: inactive until activation writes the activation event and binds memory. Persistence saves virtio state and rate limiter token state so that throttling remains coherent across snapshot boundaries.

### Generated protocol constants

There is no runtime lifecycle. Constants are compiled into dependents. When headers are regenerated, downstream code may need to adjust if new definitions appear, but the generated constants themselves do not participate in activation or teardown.

The following state view contrasts activation for balloon versus entropy at a high level.

```
  Inactive ----activation----> Active (guest memory bound)
      |                              |
      |                              +-- balloon: optional stats timer armed
      |                              +-- entropy: rate limiter tied to events
      v
  Snapshot restore may re-enter Active without repeating guest boot
```

Before the figure, both devices start inactive with queues not bound to guest memory. Activation transitions are idempotent from the perspective of virtio rules: once active, queue processing assumes valid indices and memory.

After the figure, snapshot restore can leave a device active immediately, which is why event registration distinguishes initial attach versus resume: the event subscriber may register runtime events immediately when memory is already active during restore.

---

## Failure Modes

### Memory ballooning

Descriptor errors, malformed statistics payloads, and invalid available indices are handled with logged errors and metric increments. Certain invalid available index conditions are treated as fatal to the virtual machine because they indicate unrecoverable queue corruption. Memory removal failures during inflation log and skip but do not necessarily stop the queue walk: the design favors forward progress on best effort reclaim. Statistics enablement cannot be toggled across arbitrary intervals after activation without error, preventing half-enabled states that the virtio layout cannot represent.

Silent corruption risks center on queue index misuse or guest memory aliasing; the implementation relies on virtio queue invariants and guest memory APIs that validate accesses. If those assumptions break, pages might be reclaimed incorrectly or statistics might be wrong without raising an error.

### Guest entropy

Failures include host random generator errors, guest memory mapping errors, descriptor parsing errors, interrupt signaling errors, and rate limiter internal errors. The device generally degrades by completing chains with zero bytes written when generation fails, incrementing failure metrics, and logging. Throttling is not a failure: it intentionally leaves descriptors pending until budget exists.

Silent corruption would require incorrect used lengths or writing beyond guest buffers; the implementation uses bounded buffers and descriptor loading helpers intended to prevent out-of-range writes when used as directed.

### Generated protocol constants

The primary failure is organizational: duplicated or divergent definitions if someone bypasses the generated headers. There is no runtime recovery because there is no runtime. The mitigation is process: regenerate from the canonical script and review diffs when updating vendor headers.

---

## Operational Characteristics

### Memory ballooning

CPU cost spikes during inflate when many descriptors arrive in one burst because page frame numbers are sorted and compacted into ranges before reclaim. Memory overhead includes fixed buffers for accumulation and statistics caching. I/O to the host kernel occurs during reclaim via memory advice and, for snapshots restored from file-backed mappings, an anonymous remapping workaround before advice. Observability is through structured metrics counting activations, inflations, deflations, statistics updates, and failure categories, plus logs on recoverable errors.

### Guest entropy

Cost scales with requested bytes and the number of descriptor chains processed. The rate limiter introduces timer-driven wakeups when throttled. Observability includes counters for requests, bytes delivered, throttling episodes, host generator failures, and rate limiter events. This device can become a bottleneck if policy sets byte budgets very low relative to guest demand.

### Generated protocol constants

Compile-time size and build time grow slightly with header breadth, but runtime impact is nil. Observability is not applicable beyond code review of generated diffs when updating.

---

## Design Rationale

### Memory ballooning

The balloon follows virtio’s separation between transport and device-specific queues so that Linux guests can use upstream drivers. Compacting page frame numbers before reclaim reduces system call overhead and makes memory advice more effective. Splitting statistics into a guest-provided buffer plus a device-driven completion timer matches the specification’s pattern of guest-filled records followed by device acknowledgment. Omitting the statistics queue entirely when disabled avoids exposing a virtqueue that should not exist, reducing attack surface and guest confusion.

The snapshot path reconstructs queues with explicit checking against the guest memory layout to avoid desynchronizing transport state from memory after migration. Special handling for memory restored from files acknowledges that certain memory advice operations behave differently for anonymous versus file-backed mappings.

### Guest entropy

Using the host cryptographic generator aligns guest randomness with host guarantees appropriate for cloud environments. Combining byte and operation rate limiting protects the host from abusive guests while still allowing bursts within policy. Keeping configuration space empty reflects that virtio entropy in this stack does not expose tunables through config bytes, simplifying the guest-visible surface.

Persistence includes rate limiter state so that a resumed virtual machine does not instantly gain a burst that policy would have denied if the machine had stayed continuous.

### Generated protocol constants

Mechanical generation from the same headers used elsewhere reduces transcription errors and endless debates about numeric values. Splitting output into transport, block, network, and ring groupings mirrors how different areas of the codebase evolve, while still allowing shared transport-level versioning expectations to stay uniform across devices. The balloon and entropy devices rely on transport-wide versioning constants, while block and network devices consume richer sets; keeping everything in one generated area avoids siloed copies drifting apart.

Together, the three concerns form a coherent slice of the virtio layer: two devices that deliver host services into the guest through standard queues, plus a shared specification backbone that keeps those interactions honest relative to the virtio standards they implement.
