# Device Manager Architecture

## Purpose & Boundaries

This subsystem is the virtual machine’s central place for wiring emulated devices into the guest’s address spaces, the kernel’s boot-time discovery mechanisms, and the host’s interrupt delivery path. Its core responsibility is to take abstract device implementations—virtio transports, simple timers, and on some architectures early consoles and real-time clocks—and give them stable guest-visible resources: a memory-mapped window, an assigned interrupt line when one is required, and registration with the emulated bus so the virtual CPU’s loads and stores reach the right handler. On platforms that describe hardware to the guest through firmware tables, it also contributes the corresponding table fragments so the operating system can enumerate devices in a consistent order.

What this subsystem does not do is implement the full virtio protocol, emulate legacy UART keystrokes, or own the long-running event loop that services device activity after registration. Those concerns live in dedicated device implementations and in higher-level runtime integration. If this layer failed or mis-registered a device, the guest would either never see the device at boot, would fault on access, or would receive interrupts on the wrong line—typically manifesting as missing storage or network, hung boot, or spurious interrupts. It is therefore a correctness-critical seam between resource policy, bus emulation, and KVM integration.

## Interfaces & Contracts

The subsystem exposes a memory-mapped coordinator that owns an emulated memory bus, a mapping from logical device identity to the guest physical base address, span, and optional interrupt assignment, and on one architecture an ordered accumulation of firmware description bytes for virtio devices so that enumeration order matches registration order—important for predictable root disk naming. It consumes a resource allocator that hands out non-overlapping regions in the guest’s memory-mapped device window and, when requested, distinct global system interrupt numbers from an architecture-defined pool. It also requires a handle to the virtual machine as seen by the host kernel so that interrupt file descriptors and I/O event notifications can be bound to guest-visible resources.

A separate legacy port coordinator exists only on the 32-bit-compatible server architecture. It owns a distinct I/O port bus and fixed legacy addresses for serial ports and the keyboard controller. It wires several event sources to fixed interrupt numbers that match PC-class expectations. Another small component manages a particular ACPI-oriented device that signals the guest through a dedicated interrupt and contributes its own firmware description.

The persistence layer defines how the memory-mapped coordinator’s contents are serialized for snapshots and reconstructed afterward. That layer coordinates with guest memory, the same virtual machine handle, the resource allocator, aggregated virtual machine configuration, and the host-side event dispatch facility so that restored devices are re-registered and re-subscribed in a single coherent sequence.

Callers must supply consistent identifiers for virtio instances so lookups and configuration updates remain unambiguous. The memory-mapped registration path for virtio assumes exactly one interrupt line per device; asking for zero or more than one in a single allocation is treated as a configuration error. For virtio, the virtual machine handle must already have an appropriate interrupt infrastructure so interrupt file descriptor registration succeeds.

## Data Flow

Resource allocation precedes exposure to the guest. When a new device needs a window, the allocator picks a base in the reserved memory-mapped range with alignment matching the platform’s fixed slot size for such devices. When an interrupt is needed, it allocates the next available global system interrupt from the allowed band. The coordinator records the triple of base, length, and optional interrupt in its identity map and inserts the device at that base on the emulated memory bus with the same span, so subsequent guest accesses dispatch to the correct emulator.

For virtio over memory-mapped transport, additional host kernel bindings connect queue activity to the guest’s notification register offset within the device window. Each queue’s readiness signal is associated with an MMIO write target derived from the device’s base plus a fixed notify offset; the host kernel injects those writes into the emulator’s event path. Separately, the device’s single interrupt signaling object is bound to the allocated interrupt number. Thus data moving from guest to host splits into two channels: doorbell-like notifications for queue work, and level or edge signaling upward through the interrupt controller when the device asserts completion or configuration needs attention.

On the architecture that boots with a kernel command line, virtio discovery also flows through that command line: each registered device contributes a textual description of size, base address, and interrupt so the kernel driver can attach before full ACPI enumeration. On the same architecture, virtio firmware fragments are appended in registration order. The legacy port path is orthogonal: I/O cycles hit a different bus, and interrupt file descriptors attach to fixed legacy interrupt numbers independent of the allocator’s virtio pool.

The persistence path captures, per device, transport state together with device-specific state and the same registration metadata. On restore, exact address re-reservation ensures the guest sees the same layout; the coordinator is rebuilt, virtio transports are re-instantiated from saved state, host kernel bindings are re-established, and subscribers are reattached to the host event loop. A post-restore path can also nudge activated virtio devices so in-flight work queues are reconsidered after snapshot gaps.

The following diagram summarizes how guest access and host-side signals relate for a typical memory-mapped virtio device after registration.

```
  Guest vCPU                    Emulated MMIO bus              Host kernel / devices
  ----------                    -----------------              ----------------------

  load/store  ------------->   decode by GPA range   ----->   transport + virtio impl
  to device window                  |                              |
                                    |                              |
  (optional)                           queue notify writes --------> ioeventfd -> host wake
  MMIO notify                                                            |
                                                                         v
                                                                    process queues

  interrupt                           irqfd tied to GSI  <--------- completion / config change
  injection                           asserted by host
```

Before the diagram, the key idea is that ordinary memory traffic is handled entirely inside the virtual machine monitor’s bus decoding. The diagram highlights that notification and completion are decoupled: notification uses an efficient doorbell mechanism tied to a specific MMIO offset, while completion uses the interrupt infrastructure.

After registration, steady-state operation therefore interleaves synchronous MMIO handling with asynchronous event-driven queue processing, bridged by the host kernel’s file-descriptor signaling primitives.

## Control Flow

Boot-time construction is driven by the platform builder: devices are created, the coordinator allocates or accepts fixed resources, inserts into the bus, and performs host registrations. For virtio devices intended to be visible at boot, a combined path allocates resources, binds the virtual machine, augments the command line and firmware description on relevant architectures, and returns the allocation record for logging or later correlation.

Runtime control also includes iteration helpers that walk all registered devices or only virtio ones, invoking caller-provided logic for introspection, migration preparation, or testing. Targeted access paths resolve a logical type and instance identifier to the live object for configuration changes that must reach a concrete implementation.

Snapshot save walks the identity map, skips devices that are intentionally ephemeral, and for virtio extracts transport and implementation state. Special cases exist: one network path records metadata service versioning when implied by configuration; one socket-oriented device may emit a reset signal before capture so the guest observes a consistent transport reset around migration.

Restore branches by architecture: on one architecture, certain legacy devices that live in the memory-mapped space are re-created from saved parameters before virtio restoration continues. Virtio restoration follows a shared helper pattern—rebuild implementation from saved state, reconstruct transport, re-reserve the exact guest window, re-register with the virtual machine, and subscribe to host events.

The critical path for bringing a virtio device online after restore is therefore linear and transactional at the coordinator level: any failure during host binding or bus insertion aborts the whole restore step for that device, surfacing as a persistence error rather than a half-wired device.

## State & Lifecycle

The memory-mapped coordinator owns the emulated bus instance, the authoritative map from logical identity to allocation metadata, and on one architecture the ordered firmware buffer for virtio. It does not own guest RAM layout; that remains elsewhere. The resource allocator owns the pools of interrupt numbers and memory-mapped ranges; the coordinator consumes but does not mutate allocator internals beyond calling allocation APIs.

Lifecycle phases:

**Startup.** Empty bus and empty maps; firmware buffer cleared where applicable. Devices register in a deterministic builder-driven sequence. Virtio devices receive host bindings at registration time. Boot-only devices such as a simple timer may allocate a window without an interrupt when policy allows.

**Steady state.** The bus serves guest accesses; identity map supports introspection and targeted updates. Virtio queue processing occurs outside this module but depends on the correctness of earlier registration.

**Snapshot.** The coordinator serializes sufficient information to reconstruct bus placement and per-device state. Some device categories are omitted intentionally from saved state when they are not meant to persist.

**Restore.** A fresh coordinator is populated from saved state. Exact address matching re-reserves prior windows so guest-visible addresses remain stable even though allocator identity is not itself serialized. Interrupt file descriptor registration happens again against the new virtual machine handle. Event subscribers are re-added so asynchronous activity resumes.

**Post-restore nudges.** A dedicated routine walks virtio devices and, depending on type, either reprocesses queues or signals completion in a way that compensates for events lost across snapshot boundaries. One family of devices is acknowledged to discard in-flight protocol state; the nudge exists mainly to propagate a reset-style signal already sent during capture.

A compact state view:

```
                    +------------------+
                    |    not built     |
                    +--------+---------+
                             | initialize empty structures
                             v
                    +------------------+
                    |  bootstrapping   |  allocate + register devices
                    +--------+---------+
                             |
              +--------------+---------------+
              |                              |
              v                              v
     +----------------+             +----------------+
     |  running live  |             | after snapshot |
     +--------+-------+             +--------+-------+
              |                              |
              | save state                   | restore from snapshot
              v                              v
     +----------------+             +----------------+
     |   serialized   | -----------> |    restored    |
     +----------------+             +--------+-------+
                                           |
                                           v
                                  +----------------+
                                  | steady / kick  |
                                  +----------------+
```

Before the state sketch, note that “bootstrapping” is not a long-lived mode; it is the single pass where resources are committed. Afterward, the system is either live or transitioning through migration artifacts.

After the diagram, the important invariant is that restore always constructs a new coordinator rather than mutating a partially live one, reducing the risk of mixing pre- and post-snapshot wiring.

## Failure Modes

Allocation failures are explicit: exhausted interrupt numbers or impossible alignment requests surface as allocator errors rather than silent overlap. Virtio registration fails if the single-interrupt assumption is violated. Host kernel registration failures—whether for interrupt file descriptors or for queue notification bindings—abort the operation and leave the coordinator’s map consistent with an all-or-nothing intent for that registration call, though callers must still treat partial multi-device boot sequences as a higher-level concern.

Bus insertion failures indicate overlapping regions or internal bus invariants broken by caller misuse. Lookup failures return absence or typed errors depending on the API, preventing silent mutation of the wrong device instance.

Persistence introduces semantic failures separate from mechanical deserialization: unsupported device categories may be skipped with warnings; some block acceleration backends cannot be snapshotted and are omitted, which is visible in logs rather than as a hard error during save. Restore can fail if memory cannot be re-reserved at the exact prior address or if host re-registration fails.

Assumptions that would cause subtle breakage include mismatched architecture features between capture and restore, corrupted snapshots that still deserialize, or external mutation of allocator limits between versions. The design favors failing closed on host integration errors.

## Operational Characteristics

The memory-mapped device window and interrupt pools are finite; the dominant scaling limit is the number of distinct global system interrupts available to virtio and other allocator-backed devices. Each virtio device uses a fixed-size MMIO window, so device count also consumes guest physical address space from the reserved region.

Per-device work at registration is small and linear in the number of virtio queues for setting up notifications. Snapshot size grows with the number and complexity of connected virtio devices and their internal state.

Observability is mostly indirect: informational logs during artificial queue kicks, warnings when snapshot excludes certain implementations, and error logs around transport reset signaling for socket devices. There is no dedicated metrics layer inside this subsystem; higher layers would aggregate behavior if needed.

## Design Rationale

The split between a memory-mapped coordinator and a legacy port coordinator reflects the split between portable virtio transports and fixed-layout PC compatibility hardware. Keeping virtio registration centralized ensures uniform allocation policy, consistent host kernel binding, and ordered firmware exposure where guest discovery depends on order.

Using host kernel file descriptors for both interrupts and queue notifications minimizes polling and aligns the monitor with KVM’s preferred integration model. Separating queue doorbells from completion interrupts matches virtio’s split between “work available” and “service required,” even though both cross the host boundary.

The persistence design prioritizes stable guest-visible addresses over reconstructing internal allocator state exactly, which simplifies restore at the expense of future hot-plug scenarios that might require stricter interrupt identity preservation. Comments in the persistence layer acknowledge that tradeoff explicitly.

The resource allocator’s distinct pools for MMIO and system ranges keep device registers out of unrelated memory planning. Legacy fixed interrupt numbers for serial and keyboard paths avoid consuming the dynamic pool used by virtio, reflecting hardware history rather than abstract fairness.

Overall, the subsystem is a thin but strict orchestration layer: it does not innovate on device semantics, but it enforces the invariants that let those semantics operate reliably inside a virtual machine.

## Additional Topology View

The next diagram places the major pieces relative to one another on a typical hybrid setup where virtio lives behind MMIO while legacy devices stay on I/O ports.

```
                    +-----------------------------------------+
                    |           Virtual machine monitor         |
                    |                                         |
   Guest MMIO  +--> |  Memory-mapped bus  +--> virtio + misc   |
   traffic          |        ^                  |               |
                    |        |                  v               |
                    |   coordinator <---- host kvm bindings    |
                    |                                         |
   Guest I/O   +--> |  I/O port bus  +--> UART / i8042 (x86)    |
   traffic          |        ^                  |               |
                    |        |                  v               |
                    |   legacy coordinator -- fixed IRQ map     |
                    +-----------------------------------------+
                                         |
                                         v
                              host interrupt controller model
```

Before this topology, recall that only one architecture uses the legacy coordinator path; others fold early serial and clock devices into the memory-mapped coordinator instead.

After the topology, the takeaway is complementary addressing: MMIO and port I/O never share decode logic, but both ultimately converge on host interrupt delivery and on-monitor device implementations.
