# Monitor Utilities and ACPI Integration

This document describes two related concerns inside the virtual machine monitor: a small collection of shared helpers used throughout the monitor, and the glue that assembles and installs ACPI tables so the guest operating system can discover virtual hardware. The two areas are documented together because both sit at the boundary between low-level platform conventions and the rest of the monitor: utilities normalize byte layout, alignment, signals, and simple network representations, while the ACPI path turns device-model state into firmware-visible tables and writes them into guest memory before boot.

---

## Purpose & Boundaries

**Shared utilities** exist to centralize repetitive, correctness-sensitive operations that would otherwise be duplicated across device code, builders, and memory management. Their responsibility is narrow: they provide deterministic conversions and small abstractions without owning long-lived monitor state or participating in guest execution. They do not decide policy about how much memory a guest receives, which devices are present, or how interrupts are routed; they only supply building blocks that other subsystems use when implementing those policies.

If this layer failed or behaved incorrectly, the impact would be subtle but severe: mis-decoded integers could corrupt protocol handling or guest-visible data; incorrect alignment could place structures on wrong boundaries; misuse of signal range helpers could collide with the host’s real-time signal allocation; malformed network address handling could accept invalid configurations. The utilities are not on the hot path of every exit from the guest, but they underpin configuration parsing, serialization, and any code that manipulates raw byte buffers in a host-guest contract.

**ACPI integration** has a different scope: it is responsible for constructing the set of tables that describe the virtual platform to the guest kernel, serializing those tables into guest RAM, and chaining pointers so firmware discovery works. It is explicitly not a full ACPI implementation. It does not interpret AML at runtime in the monitor, does not emulate a physical embedded controller, and does not implement dynamic table updates after boot except through whatever separate mechanisms the broader device model might provide. Its job ends once the tables are written and the root structure is placed where the architecture expects it.

If ACPI integration failed, the guest might not boot with ACPI-aware paths, might misconfigure interrupt controllers, or might fail to bind virtio or other described devices. The monitor would still exist, but the guest’s view of “hardware” would be incomplete or inconsistent with the actual emulated devices.

---

## Interfaces & Contracts

The utility layer exposes several conceptual surfaces to callers. **Byte-order helpers** read and write multi-byte integers to and from slices using fixed endianness rules, copying only as many bytes as the buffer provides (the effective width is the minimum of buffer length and integer width). **Alignment and sizing helpers** round addresses to power-of-two boundaries and convert common sizing units used when laying out guest memory. **Pointer-width bridging** narrows or widens between fixed-width integers and machine word size in environments where the monitor assumes a sixty-four-bit host. **Signal helpers** expose the host’s range of real-time signals and re-export common blocking and mask operations from the shared system utility crate so callers do not duplicate unsafe foreign-function declarations.

The **network-oriented helpers** offer a compact representation of link-layer addresses with parsing from human-readable form, conversion to and from raw octets, and serialization for configuration interchange. A separate helper validates whether an IPv4 address falls in the usable portion of the link-local range defined for automatic private addressing, excluding reserved sub-ranges at the edges of that block.

The **state-machine abstraction** wraps an arbitrary context type with a chain of handler functions. Each handler receives mutable access to the context and returns the next handler or a terminal marker. Callers own the context and define the graph of transitions; the abstraction only drives the loop until a terminal state is reached.

ACPI integration consumes **guest memory**, a **resource allocator** that can reserve regions of guest RAM for firmware blobs, the **MMIO device manager** (which accumulates AML fragments for virtio and similar devices), the **ACPI-oriented device manager** (which contributes AML for platform devices such as generic event delivery and virtual machine generation identifiers), and the list of virtual CPUs. It produces a fully linked table set in guest memory: a differentiated system description table whose bytecode aggregates contributions from those sources; a fixed-feature table pointing at that bytecode and advertising a reduced ACPI profile; a multiple-APIC description table describing the I/O APIC and per-CPU local APIC entries; an extended system description table listing the addresses of the fixed-feature and multiple-APIC tables; and a root pointer written at a fixed guest physical address for this architecture.

Callers must supply consistent inputs: the MMIO and ACPI device managers must have finished appending their AML contributions before table construction; the virtual CPU count must match the machine the guest will run. The integration layer guarantees that table bodies are allocated from the guest RAM budget tracked by the allocator, that writes either succeed fully or surface an error, and that architectural constants (local APIC base, I/O APIC MMIO location, root pointer placement) match the layout the rest of the monitor implements.

---

## Data Flow

Configuration and device setup produce **byte streams and scalar values** that need normalization. Command-line or snapshot data might supply addresses as strings; virtio setup produces AML blobs; ACPI-specific devices produce additional AML. Raw table bodies from the ACPI table library are **buffers with known length** that must land in guest RAM without overlapping unrelated regions.

The following diagram summarizes how ACPI content is assembled before any guest write.

Before the diagram: inputs are logically independent streams. Virtio-related description is collected by the MMIO path as devices register. Platform ACPI devices contribute through their own manager. Architecture-specific legacy port devices append further AML so operating systems can discover fixed resources. These streams are concatenated in a defined order to form the differentiated system description payload.

```
  +------------------+     +------------------------+
  | Virtio / MMIO    |     | ACPI platform devices  |
  | AML fragments    |     | (event device, ID dev) |
  +--------+---------+     +-----------+------------+
           |                           |
           v                           v
  +--------------------------------------------------+
  |           DSDT payload (byte vector)            |
  |   [ virtio ... | platform ... | arch legacy ... ] |
  +---------------------------+----------------------+
                              |
                              v
  +---------------------------+----------------------+
  | SDT writer: allocate RAM, write table bytes       |
  +---------------------------+----------------------+
                              |
              +---------------+---------------+
              v               v               v
        [ DSDT addr ]   [ FADT addr ]   [ MADT addr ]
              |               |               |
              +-------+-------+-------+-------+
                      v
              [ XSDT: pointer list ]
                      |
                      v
              [ RSDP at fixed PA ]
```

After the diagram: each standard table is written through the same path: the allocator hands back a guest physical address, the table library serializes into guest memory at that address, and logging records size and placement for debugging. The fixed-feature table holds the guest physical address of the differentiated system description table. The extended system description table holds addresses of the fixed-feature and multiple-APIC tables. The root pointer references the extended table. Only the root pointer uses a predetermined address; other tables use allocator-chosen regions within the configured RAM map.

---

## Control Flow

Utilities are **invoked synchronously** when other code needs a conversion or a small computation. There is no background thread or event loop inside the utility modules themselves. The state-machine runner is synchronous as well: a caller passes an initial handler and a mutable context, and the runner loops until a handler returns a terminal state.

ACPI table creation is triggered during **machine construction or restoration**, before the guest is run. The orchestrating code constructs the writer object with references to guest memory and the resource allocator, then walks a fixed sequence: build the differentiated system description payload and write it; build the fixed-feature table with a pointer to the first table’s address; build the multiple-APIC table from the virtual CPU count; build the extended system description from the addresses of the prior two; write the root pointer. Architecture-specific hooks fill in processor interrupt controller records, set IA-PC boot architecture flags in the fixed-feature table, and append legacy AML to the differentiated payload.

The critical path is strictly linear. Branching occurs only on error returns from allocation, serialization, or AML generation. There is no retry or partial success: a failure at any step aborts the whole operation and propagates upward.

The next diagram sketches control flow from the builder’s perspective.

Before the diagram: the builder owns the high-level lifecycle of the microVM. It prepares device managers and memory, then asks ACPI integration to publish tables. Success implies all guest writes completed; failure implies the guest should not be started with a partial ACPI setup.

```
   [ Builder / restore path ]
              |
              v
   +----------------------+
   | Create table writer  |
   +----------+-----------+
              |
              v
   +----------------------+     failure --> propagate
   | Build & write DSDT   |
   +----------+-----------+
              |
              v
   +----------------------+
   | Build & write FADT   |
   +----------+-----------+
              |
              v
   +----------------------+
   | Build & write MADT   |
   +----------+-----------+
              |
              v
   +----------------------+
   | Build & write XSDT   |
   +----------+-----------+
              |
              v
   +----------------------+
   | Write RSDP @ fixed   |
   +----------+-----------+
              |
              v
           success
```

After the diagram: nothing in this sequence runs inside the guest. The virtual CPUs are not required to be active. The work is entirely host-side preparation of guest memory contents.

---

## State & Lifecycle

Utility modules are **stateless** except for the state-machine wrapper, which does not store data beyond the optional pointer to the current handler. Context data lives in the caller’s structure. Network address types hold six octets for link-layer addresses; validation is structural (correct field count and hex width) rather than semantic (uniqueness on a LAN).

ACPI integration state is **transient per build**. The writer holds borrows to guest memory and the allocator during the operation. Table payloads exist briefly on the host heap before being copied into guest RAM. After successful completion, the authoritative state is the bytes in guest memory; the host-side writer can be dropped. On subsequent boots or migrations, tables are rebuilt from current device configuration rather than reused from opaque snapshots of guest RAM (though the broader persist layer may save device manager state that influences the next build).

Startup for ACPI means “once memory and devices are ready.” Steady state means “guest has a valid root pointer and linked tables.” Shutdown does not require ACPI teardown in the monitor because nothing in this layer registers host callbacks for teardown; guest RAM is reclaimed with the VM. Recovery after a failed build is simply “do not run the guest” until the error is addressed.

---

## Failure Modes

**Allocation failure** occurs when the resource allocator cannot find a contiguous guest RAM region large enough for a table. This is a hard failure: no partial table is written at the returned address because allocation failed first.

**Guest write failure** happens when the serialized table length exceeds the guest memory map or crosses an unmapped region. The error type distinguishes allocator exhaustion from memory access errors. Tests in the monitor explicitly cover the case where allocation succeeds but the write cannot complete because the guest map is smaller than the reserved ACPI region would require; this is treated as exceptional but is guarded so assumptions do not silently rot.

**AML generation failure** can arise if architecture-specific or device-specific bytecode production returns an error. Such failures are propagated and abort table publication.

**Silent corruption** is most likely if endianness were wrong in hand-rolled layouts; the shared byte-order helpers exist partly to reduce that risk. If alignment assumptions were violated at call sites, structures could be misread by the guest kernel—callers must respect the same alignment rules the utilities enforce.

Signal-related misuse could cause the monitor to register handlers in the wrong numeric range relative to other host processes; the real-time signal bounds helpers exist to align with libc’s dynamic limits.

---

## Operational Characteristics

Utilities are **lightweight**: small stack usage, no I/O, no locks. Byte-order operations copy at most eight bytes per call. Parsing a link-layer address from text is linear in string length. The state-machine runner is linear in the number of states traversed.

ACPI table construction allocates **host heap vectors** for the differentiated system description payload, which can grow with the number of virtio devices and platform objects. Each standard table write allocates guest RAM through the allocator; fragmentation behavior depends on allocator policy (the monitor uses a first-fit style policy for these allocations in the relevant tests). Logging emits debug lines per table with size and address, which aids troubleshooting without being high volume in production.

There is **no dedicated metrics** inside these modules; observability is primarily through logs on failure paths and through the ability to inspect guest memory in debug builds.

---

## Design Rationale

**Centralized byte-order helpers** avoid scattering endian conversions that must match guest-visible wire formats and ACPI structures. Using slice-length-aware read and write semantics reduces panic risk when handling partially filled buffers at API boundaries.

**Fixed root pointer placement** matches how real firmware exposes ACPI on this architecture: the guest or bootloader knows where to look without scanning. Arbitrary placement would require additional firmware interfaces or bootloader cooperation that this monitor does not implement.

**Aggregator pattern for the differentiated system description** keeps virtio, ACPI platform devices, and legacy devices decoupled. Each subsystem appends to a shared byte vector through narrow interfaces; the ACPI module only sequences those contributions and wraps them in a standard table header. That separation limits merge conflicts and keeps device ownership clear.

**Reduced ACPI profile** in the fixed-feature flags reflects the truth of the microVM: there is no legacy hardware sleep state machine as on a laptop, and power buttons are modeled as flags rather than physical circuits. The hypervisor vendor identifier field advertises the monitor identity to guests that inspect it.

**Per-CPU local APIC entries** mirror the virtual CPU count so the guest’s processor enumeration matches the actual thread count exposed by the monitor. The I/O APIC is placed at the MMIO address the interrupt model already uses, keeping one source of truth for address layout.

---

## Topology: Utilities in the Monitor

Before the diagram: callers span the crate. Utilities sit at the bottom as pure helpers. ACPI sits beside the device managers, consuming their outputs and guest memory services.

```
                    +------------------+
                    |  Builder / VMM   |
                    +--------+---------+
                             |
           +-----------------+------------------+
           |                                    |
           v                                    v
   +---------------+                  +----------------+
   | Device layers |                  | ACPI assembly  |
   | (MMIO, ACPI,  | -- AML bytes --> | + guest writes |
   |  legacy ports)|                  +--------+-------+
   +---------------+                           |
           |                                     v
           |                            +----------------+
           +--------------------------> | Guest memory |
                                        +----------------+

   +---------------------------------------------------+
   | Utilities: byte order, align, signals, net, SM    |
   | (used widely, no direct ACPI dependency)          |
   +---------------------------------------------------+
```

After the diagram: ACPI depends on guest memory and allocators; utilities generally do not depend on ACPI. The dependency direction keeps low-level helpers reusable in tests and in code paths unrelated to firmware.

---

## Closing Note

Together, the shared utilities and ACPI integration form a thin but essential layer: utilities normalize representations and control-flow patterns for the rest of the monitor, while ACPI integration turns emulated devices and architectural constants into a coherent firmware description installed before the guest runs. Understanding both clarifies how configuration becomes guest-visible bytes and how those bytes remain consistent with the interrupt and MMIO layout the rest of the virtual machine implements.
