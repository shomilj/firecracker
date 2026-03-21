# Paravirtual Block I/O Architecture

## Purpose & Boundaries

This area implements paravirtual block storage for a virtual machine: the guest operating system sees a standard block device, while the hypervisor translates guest read, write, flush, and identity requests into host-side storage operations. The implementation’s core responsibility is to honor the virtio block protocol on the virtual machine side, move data between guest memory and backing storage with correct alignment and bounds checking, and surface completion back to the guest through the virtio transport.

Two deployment shapes exist at the outer edge. One path keeps the entire data plane inside the virtual machine monitor: a seekable host file is opened, its size drives exposed capacity, and an in-process engine performs the actual host I/O. The other path delegates work to an external process over a socket-based protocol; in that mode the monitor primarily negotiates features, wires up queues and interrupts, and forwards activity rather than issuing file operations itself. The present document focuses on the in-process path where host file images and optional asynchronous submission are first-class, because that is where synchronous versus asynchronous backends, file-backed images, kernel asynchronous I/O integration, flush semantics, and local failure behavior are defined end-to-end.

What is explicitly out of scope here is the guest’s filesystem or volume manager, any network or distributed storage protocol, and the policies of the separate backend when the external process handles disks. The component also does not attempt to hide host storage volatility beyond what the configured cache and flush behavior promise; guests must still use their own journaling and ordering rules for crash consistency at the filesystem layer.

The diagram below situates the guest, the monitor-side paravirtual implementation, host file storage, and the optional external backend path. Reading left to right, the guest issues virtio traffic into the device layer; the in-process variant issues file input and output directly to a disk image, while the optional path crosses a socket boundary to a separate process that owns its own storage policy.

```
  Guest                    VMM / device layer                 Host storage
+---------+            +------------------------+            +------------+
| driver  |  virtio    |  Paravirtual block     |   file   | disk image |
|         +----------->|  implementation        +----------->| (regular   |
|         |  queues    |  + optional async I/O  |   I/O     |  file)     |
+---------+            +------------------------+            +------------+
                              ^
                              | socket protocol
                              v
                       +------------------+
                       | External backend |
                       | (optional path)  |
                       +------------------+
```

Both shapes expose virtio block semantics toward the guest, but only the in-process path couples durability and error semantics directly to a host file descriptor inside the monitor. When another process serves the disk, feature negotiation and interrupts may look the same at the virtio edge while caching, ordering, and failure behavior are determined outside this module.

If this subsystem failed catastrophically, the guest would lose access to root or data volumes, boot would stall or panic, and snapshot or migration workflows that depend on a consistent device model would be unreliable. A partial failure, such as a single bad request, is usually isolated to that request’s completion status rather than tearing down the whole device, which matches guest expectations for block devices.

## Interfaces & Contracts

Toward the guest, the device advertises virtio block capabilities: modern virtio versioning, interrupt suppression via event indices where applicable, optional read-only operation, and optionally the ability to flush. The guest supplies requests as chains of descriptors: a small header describing operation type and sector address, an optional data buffer for transfer, and a writable status byte. The implementation validates descriptor layout, direction flags, sector alignment, and that transfers do not extend past the exposed disk end.

Toward the host, the in-process backend depends on a regular file opened with read or read-write access as configured. The file must be seekable and have a well-defined length; that length is translated into a sector count for the guest configuration space. An identifier string derived from host file metadata may be returned to the guest on request so clients can tell devices apart without embedding host paths in the guest.

The internal contract between the virtio request layer and the storage engine is deliberately narrow. Each logical request carries enough information to write a completion status and to know which queue entry to complete. The engine either completes work immediately and returns a byte count, or admits work into a queue and signals completion later. Rate limiting, when enabled, consumes abstract tokens for operations and bytes before execution, and may defer work until a timer replenishes budget.

Callers must supply valid guest memory for data and status. The implementation guarantees that it will not complete a descriptor chain unless it can report a status byte when possible; if the status location is unusable, the chain may be discarded with no used-ring entry, which is a deliberate trade to avoid inventing success where none can be communicated.

## Data Flow

Guest-initiated traffic begins when the driver places one or more descriptor chains on the available ring and signals the monitor. The event path wakes the device, which pops chains from the virtio queue. Each chain is parsed into a structured request: operation type, sector index, buffer extent, and status location. Read and write operations map the sector index to a byte offset in the host file by fixed scaling. Device identity reads copy a bounded identifier into the guest buffer without touching the file beyond metadata already collected at open. Unsupported operation codes still receive a defined negative completion on the status byte when descriptor layout permits.

The next figure summarizes how data moves for a read or write when the in-process engine is selected. The vertical separation emphasizes that guest memory is the source or sink of payload bytes, while the host file is the persistence layer. Every payload byte crosses a narrow bridge: guest RAM is not handed to arbitrary host APIs without the guest memory abstraction, and the synchronous engine relies on volatile transfer helpers so safety and dirty tracking remain coherent.

```
                    +-------------------+
                    |   Guest memory    |
                    | (descriptor data) |
                    +---------+---------+
                              |
              parse / validate |
                              v
                    +-------------------+
                    |  Request dispatch |
                    +---------+---------+
                              |
            +-----------------+-----------------+
            |                                   |
            v                                   v
   +----------------+                 +----------------+
   | Sync engine    |                 | Async engine |
   | (blocking I/O) |                 | (queued I/O) |
   +--------+-------+                 +--------+-------+
            |                                   |
            |                                   |
            v                                   v
                    +-------------------+
                    | Host backing file |
                    +-------------------+
```

The asynchronous path registers guest buffer pointers with the kernel submission interface; when reads complete, guest pages touched by inbound data must be marked dirty so migration and memory tracking remain correct.

For the asynchronous engine, operations are not finished inside the queue pump. They are converted into kernel submission entries referencing preregistered file descriptors and guest buffer pointers, then submitted in a batch. Completion data returns through a separate completion channel monitored by another event source. Only after a completion entry arrives does the implementation write the virtio status byte and recycle the used ring slot. Conceptually, two cooperating loops meet in pending state: one consumes virtio chains and pushes host submissions, the other reacts to completion readiness. A batched submission step ensures work queued during a pump pass actually reaches the kernel. The figure below shows that separation; the guest may enqueue faster than completions return, so each outstanding operation carries an opaque handle through the host layer until a completion maps back to the correct descriptor chain.

```
  Virtio pump                         Host rings
 +--------+    submit batch     +------------------+
 | pop /  +-------------------->| submission queue |
 | parse  |                     +---------+--------+
 +--------+                               |
                                          v
                                   +---------------+
                                   | host storage |
                                   +---------------+
                                          |
 +--------+    completions        +---------v--------+
 | finish |<----------------------+ completion queue |
 | chains |     event signal      +------------------+
```

Completion entries are consumed in order from the host side; each maps to a single terminal virtio completion, which is why one interrupt can follow several host completions drained together.

## Control Flow

Execution is driven by a small set of asynchronous stimuli. Queue notifications pull work from the available ring until suppression rules say to stop or until a deferral occurs. A rate limiter timer may wake the device to retry work that was paused for token shortage. When the asynchronous engine is active, a completion notification indicates that one or more host operations finished and results can be applied to virtio state.

The critical path for a synchronous request runs entirely inside the queue handler: pop, parse, optionally enforce rate limits, execute read, write, or flush against the file, then publish the used descriptor and optionally raise an interrupt. The critical path for an asynchronous request forks: after a successful enqueue the chain remains incomplete until the completion handler runs, which drains completion entries, maps results to pending virtio state, updates the used ring, and signals the guest.

Branching appears in several places. Invalid descriptor chains are logged and completed with zero progress where the protocol allows, avoiding partial updates that would confuse the driver. Rate limiting can return a chain to the available side without executing I/O so it can be retried later. The asynchronous engine can signal that its submission or completion queues are full; in that case the current chain is put back and a flag remembers that the device must resume when space appears, which happens after completions are consumed.

A separate activation handshake transitions the device from configuration to live operation: guest memory is bound to queues, optional notification modes are enabled, and runtime event sources replace the one-shot activation hook. Until activation completes, queue traffic is ignored as spurious.

Admission happens when a chain is parsed and accepted into the asynchronous engine; completion is always asynchronous relative to that admission. The state machine for a single asynchronous operation can be thought of as moving from admitted to submitted to completed, with an error path that still reaches completed but carries an error result encoded in the guest-visible status.

```
       +-----------+
       |  Admitted |
       +----+------+
            |
            | enqueue to host ring
            v
       +-----------+
       | Submitted |
       +----+------+
            |
     +------+------+
     |             |
     v             v
+---------+   +---------+
| Success |   | Failure |
| result  |   | result  |
+----+----+   +----+----+
     |             |
     +------+------+
            |
            v
       +-----------+
       | Completed |
       | on virtio |
       +-----------+
```

Both success and failure converge on the same completion stage so the guest always observes a single terminal status per chain whenever the status byte can be written.

## State & Lifecycle

The device owns virtio transport state, including negotiated feature bits, queue memory, interrupt signaling handles, and the small configuration space that exposes capacity. It also owns host-side state: the open backing file, the computed sector count, any cached image identifier, and the choice of synchronous or asynchronous engine. When rate limiting is configured, token buckets live with the device and survive across individual requests.

Initialization opens the backing file, probes size, builds the identifier, and constructs the appropriate engine. Activation attaches guest memory to queues and registers for ongoing events. During steady operation, the device alternates between draining guest work and, if applicable, draining host completions. When the backing path changes for a live update, the file is reopened, the engine is rebound, configuration space is refreshed, and a configuration-change interrupt informs the guest that capacity or identity may have changed.

Shutdown and snapshot paths must reconcile in-flight asynchronous work. Before a snapshot, the implementation drains outstanding submissions and completions so no half-finished operation remains ambiguous, and flushes are synchronized according to cache policy. On final drop, cache policy again determines whether a best-effort flush to media is attempted or only asynchronous queues are drained without a full sync.

## Failure Modes

Parse failures cover malformed descriptor chains, wrong read versus write direction, missing segments where data is required, sector ranges beyond the disk, and nonconformant buffer lengths. These are treated as request-level failures: the device logs, increments error-oriented metrics, and typically completes the chain with no data movement and no used length, reflecting a hard parse failure rather than a misleading success.

Execution failures include host read or write errors, partial transfers, seek problems, and asynchronous submission errors that are not the special queue-full condition. Those map to a negative status byte for the guest when the status byte can be written. Guest memory errors while accessing data buffers produce a similar outcome. If the status byte cannot be written, the completion may be dropped entirely, which is worse for the guest but avoids asserting success.

The asynchronous engine introduces additional failure surfaces: submission rings may be full, completion rings may need draining before more work enters, and completion entries may report kernel-level I/O errors. Queue-full conditions are treated as backpressure rather than fatal error: the device stops taking new chains temporarily and retries after completions reduce pressure. A completion that reports failure still follows the normal completion path so virtio state stays consistent.

Cache policy interacts with failure semantics. In the mode where flush is not advertised, the guest is not offered a virtio-level flush capability, so ordering expectations are weaker by design. In the mode where flush is advertised, guest flush requests map to a host flush followed by a full sync of the file’s state to storage where the synchronous engine is used, or to an asynchronous fsync-equivalent operation followed by an additional full sync when draining for durability at snapshot or teardown in the asynchronous case.

Silent corruption is not a stated goal of the block layer: if the host returns short reads or write errors, the guest sees I/O error status. The main assumption that could bite operators is a mismatch between the file length on disk and the sector count cached in configuration space after an external truncation; requests that still fit parsing rules but hit a shorter file may see errors at execution time rather than at parse time.

## Operational Characteristics

Resource usage centers on the open file descriptor, per-device metrics, optional rate limiter state, and for the asynchronous engine, fixed-size submission and completion rings plus a completion notification handle. The asynchronous path can overlap host I/O with guest execution but is bounded by ring capacities; when those fill, throughput temporarily matches completion rate rather than growing without bound.

Scaling bottlenecks include single-queue virtio processing in the in-process implementation, the cost of copying or mapping guest memory for I/O, and host storage latency. The asynchronous engine reduces syscall overhead and enables batching, but still respects the same guest-visible ordering and completion rules.

Observability is primarily through counters: queue events, rate limiter stalls, asynchronous engine stalls, execution failures, configuration access problems, activation issues, and per-operation class counts and byte totals. Logging accompanies exceptional conditions rather than per-request chatter.

## Design Rationale

The dual-engine design exists to trade simplicity against performance. The synchronous engine is easy to reason about: every pop from the virtio queue finishes a request before the next begins, which minimizes outstanding state and makes snapshots straightforward. The asynchronous engine targets higher throughput and lower per-operation syscall overhead by pushing work into the kernel’s asynchronous I/O facility, at the cost of a second event source, explicit submission flushing, completion draining, and backpressure handling when rings are full.

File-backed images are the default because they map directly to portable disk artifacts, align with snapshot and migration workflows, and expose capacity through length alone. Sector alignment is enforced for data operations to match guest and protocol assumptions; trailing partial sectors at the end of a file are hidden by truncating the exposed sector count, with a warning when the file is not an exact multiple of the sector size.

The kernel asynchronous integration is deliberately restricted: only a small allowlist of operation types is permitted on preregistered file descriptors, which reduces the attack surface and keeps behavior predictable. A modest ring size balances burst capacity with memory use and keeps descriptor chains from outrunning submission slots under typical virtio packing. Completion notifications tie into the same event framework as queue notifications so the virtual machine monitor does not busy-wait on the host rings.

Flush and sync semantics are split across two policy knobs. Where flush is not advertised, the implementation does not pretend to provide cache writeback ordering through virtio, and teardown may only drain asynchronous state without forcing media sync unless the other policy is selected. Where writeback semantics are requested, flush is advertised, guest flush operations trigger durable host behavior, and device teardown attempts a stronger sync so closing the virtual machine does not leave host page cache lazily holding guest writes.

Separating the external-process backend from the in-process file backend acknowledges that some deployments want specialized storage daemons or shared storage. That path reuses virtio feature negotiation toward the guest but moves failure modes and performance to another address space; the in-process design remains the reference for how paravirtual requests map to concrete storage behavior when the monitor itself owns the disk. Taken together, the stack ties virtio descriptor semantics to host file operations, pairs a predictable synchronous path with an asynchronous path built on restricted kernel ring submission for throughput, and uses explicit cache and flush policies so guests know what durability guarantees mean in practice.
