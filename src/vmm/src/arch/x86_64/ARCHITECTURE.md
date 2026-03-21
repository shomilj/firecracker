# x86_64 Virtualization Architecture

This document explains how the x86_64-specific portion of the virtual machine monitor cooperates with the Linux kernel’s hardware virtualization interface. The focus is on processor identity exposed to guests, model-specific registers, interrupt delivery, and the boundary between in-kernel emulation and user-space handling when the guest stops running. It does not describe guest firmware layout, boot tables, or memory map policy except where those choices constrain virtualization behavior.

## Purpose & Boundaries

This subsystem’s responsibility is to configure and operate 64-bit x86 virtual processors and their supporting in-kernel devices so that a Linux guest can boot and run under the host’s virtualization extensions. It establishes what the guest believes about the CPU (through discovery leaves and associated registers), wires interrupt controllers so timer and device interrupts can reach the guest’s local advanced programmable interrupt controller, and defines which guest-visible state must be captured when the machine is snapshotted and in what order that state must be restored.

What lies outside this boundary includes generic device models, virtio transports, block and network backends, and the policy for how much RAM or how many virtual processors the user requests. If this layer failed entirely, virtual processors would not receive a coherent CPU identity or register file, interrupts would not be delivered in the PC-compatible way the guest kernel expects, snapshots would not round-trip, and the run loop would not translate the few exit reasons the monitor handles in user space.

## Interfaces & Contracts

The layer consumes services from the host kernel: creating a virtual machine context, registering guest physical memory, enabling a fixed set of optional capabilities, querying which processor capability leaves the host is willing to synthesize for guests, and driving per-virtual-processor operations such as setting capability leaves, bulk-writing model-specific registers, and running until the guest traps. It exposes, upward, a configured virtual processor ready for the generic execution engine, peripheral buses for port I/O and memory-mapped I/O dispatch, and serializable bundles of virtual processor and whole-virtual-machine state for migration and snapshot.

Callers must create the in-kernel interrupt infrastructure before creating virtual processors; the host requires that ordering. They must also have requested, at process startup, any extended save-state permissions the host needs so that capability discovery reflects features that use larger extended state areas. After boot configuration, the guest’s capability image must stay consistent with the registers that enable features; exposing a feature in discovery without the corresponding register setup invites faults inside the guest. In return, the layer guarantees deterministic boot-time register and interrupt setup for supported Linux boot paths, chunked bulk register reads and writes that respect host limits, and explicit ordering when saving and restoring so that dependent state does not diverge silently.

The following diagram situates the major conceptual pieces. The host kernel sits below; the generic monitor above uses architecture-specific outcomes from the run loop and state save and restore.

```
+------------------+       ioctl-based          +------------------+
|  Generic VMM     | <------------------------> |  Host KVM        |
|  (run loop,      |                            |  (VM + vCPU)     |
|   devices)       |                            +--------+---------+
+------------------+                                     |
        |                                                |
        |  port I/O / MMIO                               |  VM exits
        v                                                v
+------------------+                            +------------------+
|  Device models   |                            |  In-kernel       |
|  on PIO/MMIO     |                            |  PIC, IOAPIC,    |
|  buses           |                            |  PIT, LAPIC      |
+------------------+                            +------------------+
```

Before the diagram, the intent is to show that most interrupt timing and legacy compatibility live in the kernel, while the monitor only handles the exits that the kernel deliberately forwards. After the diagram, the same picture implies a contract: user space must not assume it will see every instruction or interrupt; it only sees the narrow exit types the integration code is built to interpret.

## Data Flow

Capability data enters as a large table the host reports as synthesizable for guests. That table is merged with policy from static or custom templates, then normalized so that each virtual processor sees a topology consistent with the requested count and threading mode. The normalized table is pushed into the kernel per virtual processor before registers are written. Model-specific register values arrive from the same configuration as key–value pairs, merge with mandatory boot-time values for time stamp counter and similar registers, and are applied in bulk. Extended state for floating-point and vector units flows through variable-sized save and restore buffers whose required size depends on host support for newer extensions.

At snapshot time, data flows outward in a defined sequence: multiprocessor power management state first, then general and segment-related register images, extended save area, control registers for extended state, debug state, local interrupt controller image, optional time stamp counter frequency metadata, capability table, and chunked model-specific register blobs, with virtual processor event masks last. Restoration reverses the order where dependencies demand it, for example applying segment-related state before local interrupt controller state so deadline-based timer registers can be written coherently.

This ASCII sketch summarizes data movement for snapshotting without naming internal structures.

```
  [Guest memory] <--- describe / restore ---> [Generic memory state]
        ^
        | page tables, segments (via saved register images)
        |
  [vCPU state blob] ---> serialize ---> [Snapshot store]
        ^
        | merges: CPU capabilities + MSRs + regs + LAPIC + events
        |
  [Host KVM]
```

Prose before the figure: snapshot data is not a single opaque byte range; it is a composite of register-level images and memory metadata. Prose after: restoring an inconsistent subset, such as events before general registers, can clear pending traps or interrupt shadowing incorrectly, which is why ordering is part of the architecture rather than an implementation detail.

## Control Flow

Execution is driven by repeated transitions into the host to run the guest until a condition stops execution. Most execution time stays in the kernel; the monitor wakes for exits that require user-space work. On this path, the critical handling applies to port I/O exits: the monitor dispatches reads and writes to an emulated bus connected to devices. Any other exit reason is treated as unexpected and fails the run, reflecting a design that relies on in-kernel emulation for memory-mapped I/O and hypercalls rather than expanding user-space decode.

Configuration control flow begins when the virtual machine is created: the architecture-specific initializer records which model-specific register indices the host reports as saveable, probes whether extended save buffers require a larger allocation, and sets a fixed task state segment address used by the virtual machine layout. Before any virtual processor exists, the interrupt controller and legacy timer stub are created so that later local interrupt controller setup can succeed. Each virtual processor then receives capability normalization, bulk register writes, general-purpose and segment register setup, floating-point setup, and local interrupt controller line configuration for external interrupt and non-maskable interrupt delivery.

Branching occurs where boot protocol differs: direct 64-bit handoff versus an alternate entry that uses a different register convention. Another branch is snapshot restore on a host whose time stamp counter frequency does not match the saved value within a small tolerance; in that case frequency scaling is applied so guest timekeeping remains consistent.

A simple state view of the run loop:

```
        +-------------+
        |   running   |
        +------+------+
               |
               | guest traps
               v
        +-------------+
        |  exit eval  |---- unexpected ----> fail run
        +------+------+
               |
        +-------+-------+
        |               |
   port I/O         (others: error)
        |
        v
   dispatch device
        |
        v
        +-------------+
        |   running   |
        +-------------+
```

Text before: the steady state is guest execution inside the kernel. Text after: only the port I/O branches complete successfully in the architecture-specific handler today; the diagram encodes that policy explicitly.

## State & Lifecycle

Owned state spans per-virtual-processor images including capability tables, model-specific register lists for migration, extended save areas, and debugging and event masks. At the whole-virtual-machine level, owned state includes programmable interval timer configuration, emulated programmable interrupt controller pair state, I/O advanced programmable interrupt controller state, monotonic clock parameters tied to the time stamp counter, and the cached list of register indices to snapshot.

Initialization begins with permission negotiation for dynamic extended state features on the host process, then querying synthesizable capability leaves. Virtual machine creation establishes common state and interrupt hardware. Virtual processor creation attaches to per-virtual-processor save lists augmented with template-driven and capability-inferred register indices. Boot-time configuration writes discovery state and registers, then configures interrupt lines.

Steady-state operation is the run loop with occasional snapshots. Shutdown tears down in reverse of host expectations; snapshots must be taken when other virtual processors are quiesced where required because some introspection calls are only safe without concurrent execution.

Recovery after restore replays the virtual machine-level interrupt and timer state, then per-virtual-processor images. A special case adjusts deadline timer registers when a zero value would otherwise suppress periodic delivery after restore. Deferred application of certain register writes ensures that dependent pairs, such as the time stamp counter and its deadline-based interrupt companion, are applied in an order the kernel’s internal logic expects.

## Failure Modes

Capability normalization can fail if the requested topology cannot be expressed consistently. Bulk model-specific register writes can partially apply; the layer treats incomplete application as failure. Individual register reads during snapshot can fail if an index is unsupported on the current host, surfacing as a targeted error rather than silent truncation.

Snapshot restore ordering violations could leave interrupts misconfigured or drop pending exceptions; the design mitigates this with a documented sequence. Mismatch between saved time stamp counter frequency and the current host without scaling can make guests observe time drift or break software that assumes a fixed frequency; the layer compares frequencies with tolerance and scales when needed. If frequency metadata is missing from older snapshots, portability across different host processor models is degraded and may be logged.

Silent corruption is most likely if a template exposes features in discovery without matching register initialization, or if restore applies register chunks in the wrong order; both are guarded by convention and ordering rather than automatic cross-validation of every dependency edge.

## Operational Characteristics

Resource use scales with virtual processor count and the size of extended save areas when wide vector or matrix extensions are enabled. Capability and register list processing is linear in the number of leaves and indices. The run loop adds latency only when port I/O exits occur; memory-mapped I/O and interrupt injection typically avoid user space.

Bottlenecks include bulk register read and write bandwidth during snapshot, extended save buffer size for feature-rich guests, and host-imposed limits on how many registers can be transferred per call, which forces chunking. Observability includes counters on port I/O exits and failure metrics when unexpected exit reasons occur; latency histograms may cover port I/O handling time.

## Design Rationale

x86_64 guests discover the processor through a large, versioned capability mechanism and configure features through model-specific registers. The design centralizes discovery normalization so every virtual processor presents a coherent view of sockets, cores, and threads, which reduces guest kernel confusion and matches how bare metal enumerates topology. Templates allow operators to pin or mask features for stability across hosts, trading some fidelity for predictable behavior.

The host’s in-kernel irqchip models legacy PC interrupt paths because many guests and operating system expectations assume that wiring. Handling only port I/O exits in user space keeps the hot path short and pushes complexity into the kernel’s device models where possible; unexpected exits fail fast to surface configuration gaps instead of masking them.

Snapshot completeness requires tracking many registers beyond the minimal architectural set, including performance, security-mitigation, and hypervisor-interaction ranges, filtered to what the host allows and what capability leaves imply. Chunking and deferred application exist because the host interface has fixed buffer sizes and because some register pairs have implicit dependencies enforced inside the kernel. Dynamic extended state permission exists so discovery matches what guests may legally enable with extended state instructions, avoiding half-enabled combinations that fault at boot.

Tolerance-based time stamp counter frequency comparison acknowledges that reported megahertz values fluctuate slightly with calibration; scaling only when divergence exceeds that band avoids unnecessary adjustment while preserving monotonicity expectations for migrated workloads.

---

**Topology of KVM-related setup (conceptual)**

```
   Host process
        |
        |-- request extended state permissions (when supported)
        |
        +-- open KVM interface
                |
                +-- record synthesizable capability leaves
                |
                +-- create VM
                        |
                        +-- register memory, set VM layout hooks
                        |
                        +-- build irqchip + PIT before vCPUs
                                |
                                +-- for each vCPU:
                                      normalize CPUID for topology
                                      set CPUID / MSRs / regs / LAPIC pins
```

Explanatory text before: setup is deliberately sequenced to satisfy kernel prerequisites and to avoid inconsistent feature exposure. Explanatory text after: any step omitted here tends to fail late, often as cryptic kernel errors or guest crashes during early boot, which is why the ordering is part of the architecture story rather than incidental procedure.

This document should be read together with generic virtual machine monitor documentation for cross-architecture concerns; here, the emphasis remains on how x86_64 discovery, registers, interrupts, and exits interact with the Linux virtualization interface.
