# Shared VirtIO Infrastructure Architecture

This document describes the cross-cutting VirtIO plumbing that sits underneath concrete device implementations. It covers the memory-mapped transport, virtqueue mechanics, guest memory vectorization helpers, snapshot-oriented persistence hooks, the external process integration path, and the abstract contract that every emulated VirtIO device must satisfy. Device-specific transports, protocol details for particular device classes, and generated protocol constants are intentionally out of scope here; those concerns live with the individual device modules and their companion artifacts.

## Purpose & Boundaries

The shared layer exists to give the virtual machine monitor a single, coherent implementation of VirtIO 1.0 semantics for Firecracker-style guests. Its responsibility is to translate between three worlds: the guest’s MMIO-driven programming model, the host’s memory and threading constraints, and—when configured—the expectations of an out-of-process backend that accelerates data plane work.

What this layer owns is the shape of the virtqueues, the rules for feature negotiation and configuration space access, the lifecycle gates that prevent illegal queue mutation, the memory barriers and dirty-page tracking needed for correctness and migration, and the scaffolding that lets higher layers plug in either pure emulation or delegation to a userspace companion process.

What it does not own is the meaning of individual requests. Block read and write semantics, network frame formats, entropy generation policies, and similar behaviors belong to the device modules that build on top of this foundation. The shared layer also does not define how the broader VMM schedules work onto CPUs or how the guest interrupt controller is wired; it exposes interrupt signaling primitives, but wiring is an integration concern.

If this infrastructure failed wholesale, guest I/O would stall or the VMM would become unable to honor VirtIO invariants. Incorrect virtqueue handling can corrupt guest memory or allow a malicious driver to pin the emulator in a tight loop. Incorrect MMIO state transitions can leave devices half-initialized, causing drivers to wedge or to assume capabilities that were never actually enabled. The persistence hooks are similarly sensitive: restoring inconsistent queue indices or transport registers can desynchronize the guest’s view of device progress relative to host-side buffers.

The following diagram situates the shared pieces relative to a typical VirtIO device stack. The arrows indicate primary data and control dependencies, not every callback edge.

```
                    +------------------+
                    |  Guest VirtIO    |
                    |  driver (in VM)  |
                    +--------+---------+
                             |
                             | MMIO loads/stores
                             v
                    +------------------+
                    | MMIO transport   |<------+
                    | (registers +     |       |
                    |  status machine) |       |
                    +--------+---------+       |
                             |                 |
              feature/config |                 | interrupt status
              reads/writes   |                 | and event signaling
                             v                 |
                    +------------------+       |
                    | Abstract VirtIO  |-------+
                    | device surface   |
                    +--------+---------+
                             |
              queues, kicks, | memory views
              buffer chains  v
                    +------------------+       +------------------+
                    | Virtqueue state  |       | Optional external |
                    | and iterators    |       | backend session   |
                    +------------------+       +------------------+
                             |                         ^
                             |                         |
                             v                         |
                    +------------------+       +------------------+
                    | I/O vector       |       | Companion process |
                    | helpers          |       | protocol setup    |
                    +------------------+       +------------------+
```

Before the diagram, the important idea is that the MMIO transport is the narrow waist through which the guest configures everything. After the diagram, the takeaway is that virtqueues and vector helpers are shared machinery that concrete devices reuse, while the external backend path is an alternate sink for the same queue configuration when the data plane is outsourced.

## Interfaces & Contracts

The abstract device surface is a behavioral contract rather than a wire format. Each implementation must advertise which VirtIO feature bits it supports, merge driver acknowledgements into a negotiated mask without accepting bits the device never offered, expose one or more virtqueues together with per-queue kick notifications, and provide a stable interrupt signaling object that the MMIO layer can combine with transport-level interrupt status bits.

The MMIO transport consumes that contract by performing only the operations the VirtIO specification allows at each driver readiness stage. Feature selection uses paged registers so that 64-bit feature masks are visible to 32-bit guest code without special cases in the device implementations themselves. Queue setup is mediated through a queue selector: the driver chooses an index, writes sizes and guest physical addresses for the descriptor table and rings, and marks the queue ready. The transport refuses to mutate queue state when the device status bits indicate that configuration is complete or that the driver has declared failure.

The virtqueue implementation exposes operations to initialize host-side pointers from guest physical addresses, to consume available descriptor chains, to publish used elements, and to participate in interrupt efficiency features when negotiated. Initialization validates power-of-two sizing, alignment of ring components, and the ability to map contiguous guest memory for each ring structure. Consumers receive iterator-like views of descriptor chains with explicit limits on chain length to avoid infinite loops on malformed tables.

The guest memory vectorization layer turns descriptor chains into scatter-gather lists suitable for host system calls and for byte-granular copying. One family of helpers aggregates read-only guest buffers for outbound data movement; another aggregates write-only guest regions for inbound data movement, including proactive dirty marking so that migration observes guest-visible writes.

The external process integration presents a session-oriented interface: connect over a local socket, negotiate overlapping feature sets between the VMM and the backend, publish the guest memory layout as a set of mmap-backed regions the backend can translate, and program each queue’s location in guest memory together with kick and call file descriptors. A companion metrics registry allows each backend instance to record activation failures, configuration failures, and timing without hard-wiring device-specific identifiers into the shared module.

Callers must maintain VirtIO ordering expectations: used-ring updates must become visible only after descriptor contents are written, and available-index advancement on the guest side must be observed with acquire semantics on the device side. The MMIO layer assumes the encapsulated device respects activation failure signaling. Snapshot restore paths assume saved queue state is internally consistent with saved feature bits and device type expectations enforced by the higher-level restorer.

## Data Flow

Guest traffic enters as 32-bit MMIO transactions against a compact register file. Identity and capability reads return magic values, version, device type, and paged feature words. Interrupt status reads reflect a shared atomic bitfield that accumulates reasons for interrupting the guest, with special handling when the transport knows interrupts are proxied from an external backend that cannot mirror arbitrary bit combinations.

Configuration data reads and writes pass through to the device implementation once the driver has reached an appropriate readiness state. Writes to device-specific configuration space are rejected until the driver has acknowledged device presence and declared itself; this prevents early scribbling while the guest is still discovering topology.

Queue data movement does not flow through the MMIO registers. Instead, the driver places descriptor chains in guest memory and advances the available ring; the device side pops chains, interprets buffer direction flags, and completes work by writing used elements and optionally signaling an interrupt. The vector helpers translate chains into lists of guest memory slices, validating that each span lies within the guest memory map.

The external backend path repackages the same guest memory addresses into a form the companion understands. Host virtual addresses for ring components are derived through the guest memory abstraction, and a unified call channel is registered so that completion notifications from any queue collapse into the same interrupt pathway the purely emulated devices use.

The following diagram emphasizes how MMIO is control-plane-centric while the virtqueues are data-plane-centric.

```
  Guest physical memory
  +----------------------------------------------+
  | descriptor table | avail ring | used ring  |
  +---------+--------+------------+--------------+
            ^                       ^
            |                       |
            | pops/pushes           | MMIO never reads
            | chains                | these directly
            |                       |
     +------+-------+         +------+--------+
     | Virtqueue    |         | MMIO transport |
     | engine       |         | (setup + IRQ   |
     |              |         |  bookkeeping)  |
     +------+-------+         +----------------+
            |
            v
     +--------------+        +------------------+
     | Vector       |        | External backend |
     | helpers      |        | (optional)       |
     +--------------+        +------------------+

  Interrupt path: used completions -> shared status bits -> guest IRQ
```

Before this diagram, the key point is that MMIO configures addresses and sizes, but actual payload bytes live in guest RAM. After the diagram, the reader should note that vector helpers and optional external backends are alternative consumers of the same queue-visible memory layout.

## Control Flow

Execution is driven by guest MMIO stores and loads, by queue kick events that indicate new work, and by activity on interrupt signaling descriptors. The critical initialization path walks the VirtIO 1.0 status progression: the driver acknowledges the device, declares itself, negotiates features, transitions to a features-ready state, prepares queues, and finally sets a running state that triggers activation on the device implementation. Activation binds the current guest memory map to queue initialization so that host pointers reflect the guest’s chosen layout.

Branching occurs around illegal transitions. Writes that would change queue parameters after the device is fully configured are ignored with warnings rather than silently mutating live state. Feature acknowledgements that include bits not previously advertised are masked out. Activation failures set a reset-needed indication and raise a configuration interrupt so the guest can recover according to the specification.

Queue processing paths branch on whether notification suppression is active. When suppression is enabled, the engine may defer popping until it can safely re-enable notifications, mirroring the Linux kernel’s “needs event” style decision for whether to inject an interrupt after batching used elements.

External backend setup runs after negotiation and memory publication. It walks queues in index order, pushing ring geometry and addresses to the companion, then enables rings explicitly. Failure at any step aborts the setup and surfaces as an activation error class that the MMIO layer maps into the same reset-needed pathway as local activation failures.

## State & Lifecycle

The MMIO transport owns transport-visible registers: feature page selectors, queue selectors, a monotonic configuration generation counter, the rolling device status bitfield, and a mirror of interrupt status visible to the guest. It also holds a snapshot of guest memory for activation, distinct from the queues themselves, which belong to the device implementation.

Each virtqueue owns sizing metadata, readiness, guest physical addresses, host pointers into mapped guest memory, cursor state for available and used indices, and optional suppression-related counters. Descriptor chain views are ephemeral objects built atop the mapped descriptor table.

The interrupt helper pairs an atomic status word with a non-blocking event descriptor so the rest of the VMM can inject guest interrupts when either configuration changes or buffer completions require attention.

At startup, transports and queues begin uninitialized except for static maxima. During steady operation, queues advance indices and may batch used notifications. On reset, the transport clears selectors and status bits, zeroes interrupt bits, and rebuilds queue shells with preserved maximum sizes while intentionally not discarding event descriptors tied to kicks, accepting occasional spurious wakeups as harmless.

Snapshot persistence captures queue parameters and cursors, negotiated features, device type metadata, interrupt status, and activation flagging. Restoration reconstructs queues as inert structures first, then re-initializes host pointers when the saved snapshot indicates the device was active, re-applying notification suppression if the saved feature mask implies it. Transport restoration reapplies saved selectors and status verbatim on top of a freshly constructed transport shell.

## Failure Modes

MMIO misuse by the guest is largely contained: unknown registers are ignored or warned, and illegal status transitions do not advance the state machine. Activation failures are explicit: the transport marks the device as needing reset and raises a configuration interrupt. If a device implementation cannot reset cleanly, the transport may leave a failed bit set, reflecting a partially recovered system.

Virtqueue handling distinguishes benign errors from security-relevant ones. Misaligned ring addresses or unmapped memory produce structured errors during initialization, which snapshot restore surfaces as persistence failures rather than silent success. A corrupted available index that implies more work than the queue can hold is treated as a potential denial-of-service signal; runtime code is expected to fail fast rather than loop infinitely, while snapshot loading reports the condition without panicking.

Vector helpers enforce directionality: attempts to treat write-only buffers as sources or read-only buffers as sinks fail early. Overflow when summing chain lengths is treated as an error rather than wrapping silently.

External backend setup can fail at connection time, during feature negotiation, while publishing memory tables, or while programming individual rings. Each failure maps to a typed error that activation logic can translate into user-visible errors. Memory table publication requires file-backed guest regions; anonymous-only layouts cannot be shared with the companion process through this mechanism.

Silent corruption risks center on violating VirtIO memory ordering rules, double-completing buffers, or restoring mismatched queue indices relative to guest memory contents. The design leans on barriers around index updates, explicit dirty marking before handing raw pointers to scatter-gather utilities, and conservative validation when rebuilding from snapshots.

## Operational Characteristics

The MMIO register file is intentionally small; most traffic is a handful of 32-bit accesses during boot followed by queue kicks during workload execution. Virtqueue operations are pointer-heavy and cache-sensitive because they touch guest memory through mapped regions. The maximum queue depth exposed by this stack is capped at a moderate power of two to bound work per kick and simplify fixed-size data structures in helpers.

The scatter-gather ring buffer used for mutable I/O vectors avoids copying when exposing contiguous slices to system calls by mirroring virtual mappings, trading extra virtual address space and setup cost for steady-state efficiency during bursty I/O.

External backends move CPU work out of the VMM but add latency for session setup and dependency on a cooperating process and socket availability. All queues sharing one call channel simplifies interrupt merging but requires the MMIO read path to disambiguate interrupt reasons when the backend cannot represent arbitrary combined status bits.

Observability is uneven by design: queue internals rely on general logging for exceptional paths, while the external integration path includes structured counters and timing stores per device identifier, flushed with the broader metrics pipeline rather than spamming logs during normal operation.

## Design Rationale

This stack optimizes for a microVM monitor: small surface area, explicit failure modes, and paths that fail closed when the guest misbehaves. Using MMIO rather than legacy port I/O matches modern VirtIO expectations and keeps the register model aligned with widely available documentation.

Centralizing queue logic avoids duplicating subtle ring arithmetic across devices, and it concentrates the security-sensitive bounds checks where they can be reviewed once. The vector helpers exist because VirtIO chains are inherently scattered, yet host syscalls and copiers want linear or iov-style views; pushing this translation down prevents each device from reimplementing half of the same traversal logic.

The external process integration acknowledges that high-performance storage and networking often want kernel or specialized userspace workers outside the VMM. The session setup sequence mirrors upstream vhost-user expectations so that compatible backends can be reused without bespoke per-device wiring in the transport.

Persistence separates transport registers from device state because snapshots must resume mid-flight without replaying guest MMIO. Reconstructing host pointers lazily from saved guest addresses keeps snapshots smaller and avoids serializing raw pointers tied to a specific host mapping generation.

Notification suppression support trades additional branching for fewer interrupts under bursty I/O, which matters when the guest and host share a constrained interrupt budget.

The following ASCII state machine captures the MMIO-level device status progression that gates activation. Prose before the figure explains that this is a simplified view of the VirtIO 1.0 initialization story; prose after reinforces that skipped or reordered steps are rejected unless the specification explicitly allows them.

```
                    +------+
                    | init |
                    +---+--+
                        |
                        | guest sets "acknowledge"
                        v
                 +--------------+
                 | acknowledged |
                 +------+-------+
                        |
                        | guest sets "driver"
                        v
                 +--------------+
                 | driver live  |
                 +------+-------+
                        |
                        | guest sets "features ok"
                        v
                 +--------------+
                 | features ok  |
                 +------+-------+
                        |
                        | queues configured in memory
                        | guest sets "driver ok"
                        v
                 +--------------+
                 | running      |-----> activate device with
                 +--------------+       current guest memory

        failure paths:
        any invalid step -> ignored or logged; severe activation
        errors -> mark needs-reset and signal configuration IRQ
```

Taken together, these choices produce a VirtIO foundation that is strict about initialization, careful about guest memory, efficient for common I/O patterns, and extensible enough to delegate the heavy lifting to an external worker when that is the right tradeoff for a particular deployment.
