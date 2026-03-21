# Virtual Machine State Layer Architecture

## Purpose & Boundaries

This layer is responsible for everything that sits directly on top of the host kernel’s hardware-assisted virtualization interface: opening and validating access to that interface, constructing a virtual machine container, mapping guest physical address space into the process, registering those mappings with the hypervisor, creating virtual processors, and coordinating how guest execution threads interact with the hypervisor through exit handling and explicit lifecycle events. It also owns the mechanics of describing guest memory for persistence, dumping full or incremental memory images, and cooperating with per-page dirty tracking when the hypervisor exposes it.

What lies outside this boundary is equally important. Device models, buses, firmware tables, boot loading, high-level snapshot serialization formats, networking and block backends, and the main event loop that orchestrates user-visible requests are not defined here. The layer assumes that something else will configure CPU identity and features, attach MMIO dispatch tables for emulation, and decide when to pause, resume, or tear down the machine. If this layer fails, the micro virtual machine cannot run guest code at all: there is no valid mapping between guest addresses and host memory, no virtual processors to execute, and no trustworthy path to save or restore execution context. Conversely, if only higher layers fail, the hypervisor-facing state may still be consistent, but the guest would lack devices or configuration to do useful work.

## Interfaces & Contracts

Upstream callers receive a narrow but deep surface: a validated connection to the host virtualization facility, a virtual machine object holding the guest memory view and the virtual machine file descriptor, and one virtual processor object per configured CPU, each with a path to run, pause, persist registers, and participate in coordinated shutdown. Callers must supply capability lists that reflect their CPU templates, memory region descriptions that respect alignment and sizing rules imposed by the hypervisor and the memory mapping layer, and they must respect ordering constraints when creating virtual processors relative to interrupt controller setup, which differs by instruction set architecture.

The layer consumes the host kernel’s virtualization interface, standard memory mapping and anonymous file facilities, optional sealed anonymous files for contiguous backing stores, signal delivery for forcibly leaving nested guest execution, and event notifications for virtual processor exit. Invariants that callers must uphold include registering guest memory before expecting any virtual processor to run meaningfully, not exceeding the maximum number of memory slots the hypervisor advertises, and only requesting register-level persistence while virtual processors are quiesced in a paused state. In return, the layer guarantees that memory regions registered with the virtual machine file descriptor stay consistent with the in-process guest memory object, that dirty page queries align slot indices with region iteration order, and that virtual processor control messages are not processed in ways that corrupt hypervisor state—for example, refusing to serialize registers during active execution when that would race with in-flight guest instructions.

## Data Flow

Guest memory enters the system as a list of guest physical base addresses paired with byte lengths. Those tuples may be realized as anonymous mappings, as a single sealed anonymous file sliced by offset across multiple contiguous regions, or as private mappings of an on-disk snapshot image. Each region optionally carries a per-page dirty bitmap used to merge hypervisor-reported dirtiness with explicitly marked ranges from emulation paths. Creation paths apply huge-page hints where configuration requests them and set mapping protections for read and write access from the host process.

When guest memory is registered with the hypervisor, each region receives a monotonically assigned slot index matching the order of insertion. The hypervisor translates guest physical addresses to the corresponding host mapping for execution and for direct memory access exits. For snapshots, full dumps stream entire regions in order; differential dumps consult a composite view that merges the hypervisor’s dirty bitmap with the per-region bitmap, writing contiguous runs of dirty pages while preserving the layout expected when the image is mapped back. If differential tracking was never enabled, an over-approximation based on which pages are resident can substitute, with documented limitations when swapping is involved.

Virtual processor data flows from the hypervisor on every save: general-purpose registers, segment or system register equivalents, model-specific registers or their architecture-specific counterparts, floating-point and extended state, interrupt controller state visible to the processor, and metadata such as multiprocessor coordination state. Restore paths apply that data back through the same interface. Virtual-machine-wide data includes legacy interrupt controller images, advanced interrupt delivery units, programmable interval timer state, and paravirtual clock parameters on one architecture, and global interrupt controller device state on another, always accompanied by a description of guest memory layout for the serialized bundle.

## Control Flow

Construction begins with opening the host virtualization device, verifying the expected application programming interface version, merging default and template-driven capability sets, and rejecting hosts that lack required features. The next step creates a virtual machine container from that connection, retrying transient interruptions that can occur under load because the host kernel may temporarily refuse the creation syscall. The empty machine receives guest memory regions one by one; each insertion updates the aggregate guest memory object and programs the hypervisor slot before the next insertion.

Virtual processor creation is gated by architecture-specific hooks that run immediately before and after the per-CPU file descriptors are allocated. On one architecture the in-kernel legacy interrupt chip and interval timer stub must exist before any virtual processor is created; on another architecture virtual processors must exist before the global interrupt controller device is instantiated, because the host rejects the opposite order. Each virtual processor shares a non-blocking event notifier used to wake the outer runtime when a processor leaves guest mode with a terminal condition.

Execution threads load sandbox policies, install signal handlers that can force quick exit from guest mode via an immediate-exit flag in the mapped run control block, and enter a state machine that separates paused and running phases. While running, the thread repeatedly enters the hypervisor until an exit requires emulation or an external event interrupts the loop. While paused, the thread blocks on a control channel that accepts resume requests, persistence requests, and teardown directives. Control messages use an out-of-band signal to ensure a waiting hypervisor call returns promptly so the state machine can observe new commands.

## State & Lifecycle

### Host virtualization file descriptor lifecycle

The outermost handle represents the process-wide connection to the virtualization facility. It is acquired once per process instance that needs to run guests, validated for version and capabilities, and retained for the lifetime of all virtual machines created from it. From this handle, each virtual machine receives its own child file descriptor. That virtual machine descriptor owns the guest memory map as seen by the hypervisor and serves as the parent for per-virtual-processor descriptors. Virtual processor descriptors are created in index order and remain associated with their threads for the lifetime of those threads. Closing proceeds implicitly when objects drop: tearing down a virtual machine releases slots and mappings; tearing down virtual processors ends the associated host resources; the outer handle may outlive individual machines if the runtime multiplexes multiple guests serially.

The following diagram situates these objects in a simple containment hierarchy.

```
+-------------------+
| Host connection   |  (validates API, capabilities, templates)
+---------+---------+
          |
          v
+-------------------+
| Virtual machine   |  (guest memory slots, VM-wide devices)
+----+---------+----+
     |         |
     v         v
+---------+ +---------+
| vCPU 0  | | vCPU n  |  (per-CPU run state, local interrupt ctrl)
+---------+ +---------+
```

The host connection is the narrow waist through which capability checks flow; the virtual machine node is where address space is published to the hypervisor; each virtual processor leaf is where execution and most saved register state live.

### Guest physical memory regions

Guest memory is not assumed to be a single contiguous array. It is a sequence of regions, each with a guest starting address and length, backed either by anonymous memory, a sealed grow-disabled anonymous file spanning the sum of all region sizes, or a snapshot file whose offsets line up with the serialized layout. Regions are inserted into an ordered collection and mirrored into hypervisor slots with flags that optionally enable kernel-assisted dirty page logging. The number of regions cannot exceed the lesser of the hypervisor’s slot limit and practical configuration. Registration failures roll back by refusing inconsistent states: the in-memory region list and the hypervisor’s view are updated together only after both succeed.

### Virtual processor state

Each virtual processor bundles an architecture-specific facade around the hypervisor file descriptor, peripheral hooks for MMIO dispatch, channels for commands and responses, and an event object used to signal abnormal completion to the outer monitor. Thread-local storage may retain a mapped view of the run control structure so asynchronous signals can set the immediate-exit field without racing undefined behavior. Operational state distinguishes running inside the guest, paused awaiting commands, and finished after a controlled shutdown. Persistence of registers is permitted only from the paused state; attempts while running are rejected to avoid inconsistent snapshots. Resume clears stale immediate-exit indications so the next entry into guest mode is clean.

### Interrupt controllers and in-kernel timers

Two architectures diverge sharply. For the legacy personal-computer style platform, the hypervisor exposes an integrated interrupt chip created before virtual processors exist, together with a programmable interval timer configured so common speaker-port writes do not trap to user space. Saved state captures timer state, paravirtual clock data with stability flags scrubbed before restore, and three interrupt controller images: primary and cascaded programmable interrupt controllers and the input/output advanced programmable interrupt controller. Restoration applies programmable timer state first, then clock parameters, then the three controller images in that fixed order so downstream delivery logic sees a coherent wire model.

For the reduced-instruction-set platform, a global interrupt controller device is created only after all virtual processors exist, because the host implementation ties interrupt distribution to already-allocated processor objects. The device handle is retained for save and restore operations. Persisted state is expressed in terms of per-processor identifiers so redistribution matches the restored topology.

The next diagram contrasts creation order constraints.

```
x86-style (irq chip before vCPUs):

  [Create IRQ chip + PIT] ---> [Create vCPU 0..n]

ARM-style (vCPUs before GIC):

  [Create vCPU 0..n] ---> [Create global interrupt controller]
```

These ordering rules are not interchangeable; violating them produces immediate errors from the host rather than soft misconfiguration.

### Snapshot and restore ordering

Snapshot content interleaves descriptions from multiple layers; within this layer the critical ordering principles are about hypervisor-visible state versus guest RAM bytes. When rebuilding from a saved image, guest memory mappings are established first so addresses used during register restore refer to valid backing storage. On one architecture, virtual timekeeping may need scaling to match recorded frequency before applying per-processor state. Virtual processor register images are then restored for every CPU. Only after that does virtual-machine-wide state resume—on the personal-computer style platform this means timer and interrupt controller images; on the reduced-instruction-set platform this means global interrupt controller registers keyed by the saved processor identifiers. Device models and firmware objects restore later in the outer builder; this layer’s contract ensures the processor and interrupt delivery hardware are consistent before devices reconnect and before virtual processors begin running threads again.

A linearized view of the hypervisor-centric portion:

```
1. Map guest RAM regions and register with hypervisor
2. (Arch-specific prep, e.g., time scale)
3. Restore each virtual processor’s registers and local state
4. Restore VM-wide interrupt/timer/clock state
5. (Higher layers: devices, then start vCPU threads paused)
```

This sequence avoids restoring interrupt routing before processors know who they are, and it avoids applying deferred timing registers before baseline counters are in place—some architectures defer specific model-specific registers until after the bulk of state is written so countdown timers reprogram correctly.

## Failure Modes

Capability or version mismatch fails fast at connection time rather than mid-boot. Virtual machine creation can fail transiently; the implementation retries a bounded number of times with short backoff before surfacing an error. Memory registration fails if regions are misaligned, too numerous, or incompatible with hypervisor limits; such failures are explicit and do not partially register opaque state. Dirty logging queries can fail if the host reports an error; differential snapshots then cannot proceed reliably. If a differential memory dump fails midway, internal bitmap merging may retain hypervisor dirtiness so a subsequent attempt can reconcile state instead of silently dropping pages.

Virtual processor execution surfaces hard failures from the hypervisor as fatal to that execution thread: unrecoverable exit reasons, internal hypervisor errors, or certain illegal entry conditions. User-initiated pause uses signals to interrupt long-running hypervisor calls; mishandling would appear as stuck threads, so immediate-exit must be cleared on resume. If control channels disconnect unexpectedly, the state machine moves to an error exit path to avoid deadlocks.

Restore operations can fail per component: invalid interrupt controller identifiers, inconsistent timer state, or mismatched processor topology relative to saved global interrupt state. Such failures are intended to be observable rather than silently corrupting guest memory.

## Operational Characteristics

Resource use scales with guest RAM size, virtual processor count, and whether dirty tracking adds per-page metadata. Each memory slot consumes hypervisor bookkeeping; many small regions approach slot limits faster than a few large ones. Snapshot throughput depends on storage speed, dirty fraction, and whether full or differential mode avoids touching clean pages. Virtual processor threads spend most time in the hypervisor during active workloads; when idle, behavior depends on guest action. Observability is primarily indirect through metrics and logs emitted by the enclosing monitor: this layer increments failure counters on problematic exits, logs rare paravirtual clock control failures around resume, and traces uncommon hypervisor return codes.

Parallelism is bounded by the number of virtual processors; memory registration is sequential because slots are assigned in order. The kick signal path adds latency to control operations but prevents indefinite blocking inside hypervisor entry.

## Design Rationale

The split between a reusable host connection object and per-virtual-machine state isolates policy around capabilities and templates from the mechanics of address space registration. Combining default and template-driven capability sets lets product requirements add or remove features without forking the core validation logic. Retrying virtual machine creation on transient host errors trades a few microseconds of backoff for fewer spurious failures in dense fleets.

Guest memory supports multiple backing strategies because snapshots, anonymous runtime memory, and sealed files serve different operational needs: file-backed restore enables copy-on-write or userfault-driven paging in upper layers, while anonymous memory minimizes host file descriptor coupling for ephemeral workloads. Optional dirty tracking trades memory and hypervisor work for smaller incremental snapshots; the merge path between hypervisor bitmaps and user-maintained marks acknowledges that not all dirtiness originates in hardware exits.

Architecture-specific hooks around virtual processor creation encode ordering rules that the host enforces anyway; centralizing them avoids distributing easy-to-misuse assumptions across the builder. The divergent placement of interrupt controller setup reflects real kernel constraints rather than stylistic preference.

Virtual processor threading with explicit paused and running states keeps persistence and migration hooks deterministic. Rejecting save while running prevents snapshot races that would otherwise require heavyweight global synchronization. Signal-based kicking balances responsiveness against complexity compared with alternative polling schemes.

The restore ordering—memory and processors before global interrupt state, with deferred timing registers applied after bulk restore where needed—addresses subtle timer loss and mis-priming issues documented around counter and deadline interactions. Separating virtual-machine-wide restore from device restore lets interrupt delivery hardware come up cleanly before MMIO devices reconnect and before virtual threads resume, shrinking the window where the guest observes inconsistent firmware-visible state.

Overall, the design prioritizes explicit failure over silent mismatch, keeps hypervisor slot indexing aligned with region iteration for dirty bitmap coherence, and respects host-imposed ordering to make virtualization setup predictable across supported architectures.
