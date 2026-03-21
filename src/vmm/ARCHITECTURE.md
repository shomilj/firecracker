# MicroVM monitor spine

This document describes how the top-level pieces of the monitor process fit together: how the binary becomes a running virtual machine, how declarative configuration turns into live guest state, how snapshot and restore are orchestrated above the emulation layers, and how the external control surface relates to device implementations and execution backends. It intentionally avoids naming individual source artifacts or implementation symbols so the architecture can be read on its own.

## Purpose & Boundaries

The monitor’s responsibility is to host exactly one lightweight virtual machine on a Linux host: acquire access to hardware-assisted virtualization, carve out guest memory, attach emulated and paravirtual devices, run virtual processors, and expose a narrow, explicit control plane for lifecycle and observability. It coordinates boot (or resume from a saved image), mediates I/O between the guest and the host, and tears down threads and kernel objects cleanly when the instance stops.

What lives outside this spine is equally important. The monitor does not implement a general-purpose hypervisor control stack for multi-tenant scheduling, migration fabrics, or distributed orchestration; those belong to outer layers that drive the process. It does not own persistent guest disk semantics beyond the files or sockets the operator configures; it maps them into the guest’s view. Networking taps, firewall policy, and storage provisioning are host concerns that surface only through the attachment points the monitor wires up. Logging and metrics sinks are configured early and then treated as ambient services the monitor invokes rather than as part of the guest execution model.

If this spine fails, the surrounding system loses the ability to run or resume the microVM: the process may exit with a classified outcome, virtual processors may hang awaiting coordination, or host resources may leak if teardown ordering is violated. Conversely, failures in individual device backends can often be contained if the control plane can pause execution and surface an error, but a broken coordination path between virtual processors and the main thread tends to be catastrophic because it undermines pause, snapshot, and shutdown guarantees.

## Interfaces & Contracts

The monitor exposes three conceptual faces to the rest of the world: a configuration and action vocabulary used before and after the guest is running, a connection to the host’s virtualization facility and memory, and participation in a host event loop for asynchronous I/O and exit signaling.

The configuration surface accepts machine sizing, boot media, optional initial ramdisk placement, paravirtual devices, rate limits, and metadata services. Callers must respect phase rules: some actions apply only while the machine is being assembled, others only while it is running. Attempting to mix incompatible phases—such as preparing a normal boot after having committed to a restore-only path—is rejected to keep internal invariants simple.

The monitor consumes host capabilities: access to the virtualization kernel interface, memory mapping primitives, optional userfault-based memory backing, timers, and signal disposition. It also consumes a set of sandbox filters keyed by thread role so that the main thread, API servicing threads, and virtual processor threads each receive an appropriate syscall profile. Filters are deserialized from a compact binary form and installed late enough that initialization syscalls remain available, but early enough that steady-state work runs confined.

In return, the monitor guarantees deterministic phase transitions when inputs are valid: configuration accumulates into a resource bundle, that bundle is realized as kernel objects and devices, virtual processors start in a quiescent execution mode until explicitly advanced, and shutdown follows a single negotiated path that avoids reference cycles between the main coordinator and worker threads.

The internal contract between the coordinator and each virtual processor is message-based: ordered requests to pause, resume, persist architectural state, or finish, with bounded waits for acknowledgements. Devices are reached through bus abstractions; the coordinator does not duplicate device logic but does own the ordering of registration, activation, and persistence.

## Data Flow

Configuration data enters as structured requests or as a one-shot aggregate description. It is normalized into a resource bundle that separates machine parameters, boot inputs, and device builders. Guest memory is either allocated to match the bundle or reconstructed from snapshot metadata and an external memory backend.

The boot path transforms that bundle into concrete artifacts: the kernel image is loaded into guest memory, an optional initial ramdisk is placed at an architecture-defined region computed from memory layout, command-line fragments accumulate root devices and paravirtual attachment hints, and platform tables are prepared. Device attachment follows a deliberate order so that documented memory layouts and guest expectations remain stable.

Live execution data flows through paravirtual queues and legacy paths as the guest runs. For snapshots, the coordinator gathers architectural state from the virtualization layer, per-processor registers and model-specific state, device persistence records, and ACPI-related material. Memory contents are written through a separate channel that can capture full or incremental footprints depending on operator choice. After a snapshot, the monitor may mark additional guest pages dirty so queue metadata remains consistent with the memory image.

Restore inverts the flow: serialized state is validated and merged with operator overrides for host-specific attachments such as tap interface names. Guest memory is re-instantiated from a mapped file or through a userfault handshake that pairs anonymous regions with an external population service. Device records rehydrate into live devices using the same resource bundle so introspection APIs remain coherent.

The following diagram situates the major directional flows without naming implementation details.

```
                    +------------------+
                    |  Outer control   |
                    |  (API / CLI)     |
                    +--------+---------+
                             |
              configuration  |  actions (pause, snap, etc.)
                             v
                    +--------+---------+
                    |  Phase router    |
                    |  pre-boot vs     |
                    |  runtime         |
                    +--------+---------+
                             |
         +-------------------+-------------------+
         |                                       |
         v                                       v
+----------------+                    +------------------+
| Resource       |                    | Live coordinator |
| bundle &       |                    | (virtual CPUs,   |
| memory plan    |                    |  devices, host   |
+--------+-------+                    |  events)         |
         |                            +--------+---------+
         | build / restore                     |
         v                                       |
+----------------+                              |
| Host VM object |<-----------------------------+
| + devices      |
| + memory       |
+----------------+
         |
         |  guest memory & I/O
         v
+----------------+
| Guest workload |
+----------------+
```

Before the diagram, the intent is to show that all external inputs pass through a phase router that enforces which operations are legal. After the diagram, the takeaway is that the resource bundle feeds construction or restoration, while the live coordinator owns steady-state motion and snapshot capture.

## Control Flow

Execution is driven by a combination of incoming requests, the host event loop, and virtual processor exits. During assembly, the process either applies successive configuration actions or performs a single restore action that must not follow a boot-specific configuration path; those policies prevent contradictory guest definitions.

Boot assembly constructs the host virtual machine, registers memory, loads images, attaches devices, applies platform-specific CPU templates, wires optional debugging aids, starts virtual processor threads in a halted mode, installs the main-thread sandbox filter, and registers the coordinator with the event loop. A final transition moves virtual processors from halted to running unless the operator requested an explicit pause-first workflow.

Runtime control flows through a separate handler that locks the coordinator for short operations: pause and resume broadcast to all virtual processors and wait for acknowledgements; snapshot creation requires the guest to be paused first; hot adjustments touch specific device categories. Guest-initiated shutdown on certain architectures funnels through legacy input devices and raises a coordinated stop.

The critical path for a clean snapshot is: pause, collect state from processors and devices, write memory and metadata, then optionally resume. The critical path for restore is: load metadata, reconcile configuration, map memory, restore processors and devices, optionally notify identity-changing facilities, start processors paused, apply sandboxing, then resume if requested.

A compact view of the phase gate:

```
  [ Not started ]
       |
       | configure / restore
       v
  [ Built, paused ] ---- resume ----> [ Running ]
       ^                                  |
       |                                  | pause
       +----------------------------------+
       |
       | snapshot create (while paused)
       v
  [ Snapshot written ] --> resume or stop
```

Prose before the figure: the monitor distinguishes an uninitialized process, a constructed but not yet executing guest, and a running guest, with snapshots taken only from the paused state in the middle. Prose after: the loop emphasizes that pause and resume are symmetric bookends around long-running operation.

## State & Lifecycle

The coordinator owns instance metadata visible to operators, the connection to the host virtualization facility, the guest memory handle, optional userfault state for lazy memory, virtual processor handles, a shared exit notifier, an allocator for guest-visible system addresses, MMIO routing, legacy port I/O on applicable architectures, and ACPI orchestration. The resource bundle mirrors what operators configured so queries remain accurate even when some fields are absent after restore.

Initialization begins with capability checks and allocator setup, then constructs empty device managers and processor placeholders. Boot fills memory and devices; restore replaces managers from serialized forms and may patch serial initialization when firmware would not naturally rerun.

Steady-state mutation happens through runtime actions that lock the coordinator briefly. Shutdown is negotiated: a finish signal drains processor threads, clears handles, records metrics, and sets an exit disposition. Dropping the coordinator repeats stop logic to recover from partial construction failures.

Snapshot-related lifecycle notes: saving walks devices to produce parallel state records; loading reapplies them through constructor paths that also repopulate the resource bundle for consistency. ACPI-adjacent identity notifications are sent before execution resumes so guests observe a coherent resume story.

## Failure Modes

Invalid configuration fails fast before kernel objects are committed. Restore conflicts with prior boot-oriented configuration are treated as hard errors to avoid mixed semantics. Snapshot creation errors while paused leave the guest paused; the operator must decide whether to resume or tear down.

Virtual processor communication timeouts indicate potential deadlocks and surface as errors rather than silent continuation. Sandbox installation failure aborts the thread because continuing without confinement violates operational assumptions.

Signal handlers for severe faults terminate the process with distinct exit codes and flush metrics first. A dedicated handler for filter-induced faults records the offending syscall when available. Broken pipes on certain descriptors are logged without exiting because upstream disconnects are expected in some deployments.

Silent corruption risks center on mismatched memory images and metadata, host CPU vendor drift across migration, and userfault peers disappearing; the design mitigates through sanity checks, warnings, and keeping critical file descriptors alive in-process.

## Operational Characteristics

Memory dominates resource usage: guest RAM plus mapping overhead, with optional incremental tracking for dirty pages. Virtual processors consume host threads and stack space; device emulation adds file descriptors and event subscriptions. The event loop scales with the number of asynchronous sources attached.

Bottlenecks appear at memory snapshot and restore throughput, virtual processor synchronization barriers, and any path that serializes large device state. API handlers intentionally avoid holding locks across slow work except where atomicity demands it.

Observability relies on structured logging, counters for signals and sandbox faults, and latency measurements around major actions. Metrics flushing can be triggered explicitly in steady state.

## Design Rationale

The split between a resource bundle and a live coordinator keeps configuration replayable and testable independent of kernel handles. Starting virtual processors in a halted mode simplifies attaching buses and applying filters before execution races begin. Phase-gated APIs reduce the state machine exposed to operators and shrink the compatibility surface across releases.

Sandbox maps per thread category trade setup complexity for defense in depth: the main thread can still perform initialization syscalls while virtual processors receive a tighter profile suited to their loop. Late installation avoids ordering hazards with thread spawn.

Snapshot orchestration at this layer rather than inside each device centralizes versioning, memory image pairing, and validation while still delegating device-specific serialization to the device modules. The optional userfault path trades implementation complexity for lazy population and large-memory restore flexibility.

Initrd handling stays small and focused: if present, compute a valid load window from the memory map, copy bytes into guest RAM, and pass sizing metadata to platform configuration. This avoids duplicating broader loading policy already owned by the kernel image path.

Signal handling chooses process exit on serious faults to prevent undefined continuation after memory or sandbox violations, while lighter anomalies are metered. The overall spine therefore prioritizes predictable failure, explicit lifecycle edges, and clear separation between declarative intent, host virtualization mechanics, and guest-visible devices.
