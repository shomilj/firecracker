# Device Emulation Core (Non-Paravirtual Subtree)

This document describes the architecture of the device emulation layer that lives alongside the paravirtual device implementation. The paravirtual transport and queue machinery are intentionally out of scope here; they are documented separately. What remains is a compact set of facilities: a generic memory-mapped routing layer, legacy platform devices, host-oriented pseudo devices, and a small ACPI-oriented component that bridges firmware tables, guest memory, and interrupt delivery. Together they give guests a believable minimal platform while keeping the virtual machine monitor’s responsibilities narrow and testable.

## Purpose & Boundaries

The responsibility of this area is to model how the guest’s load and store operations reach emulated hardware at specific addresses, and to host a handful of devices that are not implemented through the paravirtual stack. The routing layer maps contiguous address ranges to mutually exclusive device instances, serializes concurrent access, and forwards byte-granular read and write operations. On top of that, legacy devices reproduce behaviors that operating systems expect from classic PC or ARM boards: a byte-wide UART for consoles, an optional real-time clock block on AArch64, and a heavily simplified keyboard controller that nonetheless supports power and reset semantics. Pseudo devices exist purely for the monitor’s observability goals rather than guest-visible hardware fidelity; the boot timing helper is the primary example. The ACPI-oriented piece supplies a generation identifier that guests running certain enterprise workloads use to detect machine identity changes, including wiring that description into ACPI Machine Language so the guest firmware can discover the device, plus mechanisms to notify the guest when that identity changes after restore operations.

This code is explicitly not responsible for virtio queues, virtqueues, block or network backends, or the detailed MMIO protocol used by paravirtual devices. Those concerns live under the paravirtual subtree. Nor does this layer decide global machine layout, IRQ routing at the kernel virtual machine interface, or construction of full ACPI table blobs; higher layers allocate resources, register interrupt file descriptors, and assemble tables, while this subtree supplies the device semantics and, for the generation identifier, the AML fragments and guest memory image that those layers stitch together. If the routing layer failed outright, guest MMIO would misroute or stall, breaking every device hung off it including paravirtual transports. If legacy devices failed selectively, consoles might disappear, reboot signaling might break, or timekeeping might drift depending on architecture. If pseudo devices failed, observability would degrade but correctness of guest execution would typically remain intact. If the ACPI-oriented generation device failed to integrate, affected guests might not see identity changes reliably after migration or restore.

## Interfaces & Contracts

The routing layer exposes a narrow contract to the rest of the monitor: callers may insert a device at a base address with a non-zero length if and only if that range does not overlap any existing range. Insertion failure is the only explicit error from the allocator’s perspective. Reads and writes take a physical address and a buffer; the layer returns a boolean indicating whether any device claimed the access. Unclaimed accesses leave the buffer untouched, which implies callers that care about architectural behavior for holes must handle that elsewhere. Each resident device is held behind a mutex so that individual emulation can mutate internal state without requiring the whole map to be exclusive. The outward contract for emulated devices is uniform: translate an offset within the registered window into device-specific behavior for reads and writes. One variant of the aggregate device representation participates in the host’s event loop for asynchronous input; every other variant panics if driven through that path, which documents an invariant that only the serial path is poll-driven at this layer.

Legacy UART emulation consumes an interrupt trigger abstraction backed by an event notification object, optional standard input as a readable byte source, and hooks for metrics. It exposes byte-wide MMIO behavior aligned with what the imported serial core expects. The AArch64 real-time clock wrapper accepts only aligned four-byte accesses at valid offsets and delegates to a stock real-time clock model with event hooks for invalid accesses. The simplified keyboard controller accepts reset and interrupt event notifications from construction time and exposes byte-wide data and status ports; it can also synthesize key sequences for management operations such as control-alt-delete when the internal buffer allows.

The boot timing pseudo device accepts only a specific single-byte write at offset zero to mark guest boot completion; all other accesses are ignored. Reads are explicitly no-operations at this layer.

The generation identifier component, from the perspective of this subtree, is constructed with a guest physical address, a global system interrupt number, a handle used to raise an interrupt, and access to guest memory for writing the identifier bytes. It can serialize its allocation metadata for snapshots and rebuild itself when given matching restored resources. It also implements an ACPI Machine Language exporter that declares a device node with a vendor-specific hardware identifier, a compatible identifier string aligned with the industry specification, a human-readable name, and a package encoding the guest physical address split into low and high parts. External subsystems are expected to register the interrupt notification with the virtual machine monitor file descriptor at the allocated line, append AML from this exporter into firmware blobs, and invoke notification after the guest-visible memory has been updated following a resume path.

## Data Flow

Guest-initiated MMIO begins when the virtual CPU performs a load or store whose physical address falls into a region registered on the routing map. The map looks up the latest region whose start is less than or equal to the access address, then checks whether the address lies before the end of that region. If so, it subtracts the region base to compute an offset and dispatches to the mutex-protected device with that offset and the access buffer. If no region matches, the operation reports failure upward without side effects.

For the UART path, MMIO reads and writes move single bytes between the guest and an emulated register file. When standard input is wired and a companion buffer-ready notification exists, the host event loop may wake on either the readable input descriptor or the buffer-ready descriptor. Data read from the host flows into an internal FIFO subject to capacity checks; conversely, guest writes may flow to a sink or standard output. Metrics record successful and missed operations separately.

For the AArch64 clock, valid traffic is four bytes in or out at supported offsets; invalid shapes increment error and miss counters and emit warnings.

For the keyboard controller, reads return synthesized status or dequeue buffered scan data, possibly re-triggering an interrupt if more data remains. Writes interpret command bytes on the status port or data port, update internal registers, push acknowledgments into the buffer, or signal a host reset event when the guest issues the reset command.

For the boot timer, the only semantically meaningful traffic is a one-byte write matching a fixed magic value at offset zero, which triggers a log line containing elapsed wall and CPU time relative to a captured start timestamp.

For the generation identifier, creation time writes sixteen random bytes into guest memory at the chosen address in little-endian order as a single large integer. After snapshot restore, the restore path must regenerate or reload identity as policy dictates; the device supports notifying the guest through the interrupt trigger so operating systems that cache the value can refresh. AML generation flows outward as bytes appended into firmware image buffers; those buffers eventually become part of tables the guest reads before boot.

The following diagram summarizes MMIO routing at a high level.

```
                    +------------------+
                    | Guest MMIO access |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Address lookup    |
                    | (base, length)    |
                    +--------+---------+
                             |
               +-------------+-------------+
               | no match                   | match
               v                            v
        +--------------+           +----------------+
        | Unhandled     |           | Compute offset  |
        | (buffer       |           | Lock device     |
        |  unchanged)   |           +--------+-------+
        +--------------+                    |
                                              v
                    +-------------------------+-------------------------+
                    | Dispatch by device kind (legacy, pseudo, transport)|
                    +-------------------------+-------------------------+
```

After routing, data either flows between guest buffers and emulated registers, or in the generation identifier case, between host randomness or saved state and guest RAM.

## Control Flow

Execution is driven by two distinct patterns. Synchronous emulation runs entirely on the VCPU thread or whichever thread services MMIO exits: the routing layer’s read or write path locks the target and invokes the device handler. Asynchronous handling applies only to the UART when input is available: initialization registers file descriptors with a central event multiplexer, and when readiness or hang-up events arrive, a callback drains input, refills the FIFO, adjusts registration when error conditions occur, and may signal buffer readiness through an event file descriptor pair.

Insertion into the routing map is a control point that happens during machine construction: the monitor chooses bases and lengths consistent with the platform model, then registers devices. Failure at insertion time prevents overlapping maps, which protects against ambiguous decoding.

The keyboard controller exposes another control path from the host side: management logic can inject a multi-byte key sequence into the internal ring buffer subject to capacity, which may raise the keyboard interrupt if not disabled by guest configuration.

Generation identifier notification is an explicit control action invoked by outer layers after identity in guest memory is known to be current, typically following snapshot resume when a new random value was written.

State machine thinking applies lightly to the UART’s event-driven loop: descriptors start registered, may be removed on end-of-file, resource exhaustion, or configuration error, and the buffer-ready edge channel coordinates between FIFO state and poll interest.

## State & Lifecycle

The routing map owns an ordered mapping from start addresses to length-tagged device slots. Cloning duplicates the map structure but shares the same underlying device arcs, so readers should treat clone semantics carefully if they expect independence. Devices inserted into the map are typically created immediately before insertion during VM build; removal is not modeled as a first-class operation in this layer, matching a static platform assumption.

Legacy devices carry mutable register state. The UART holds FIFO and line state inside the imported model plus optional input and output handles. The keyboard controller tracks status, control, output port values, the last command, and a bounded ring buffer with head and tail indices. The AArch64 clock holds timekeeping state inside the imported model with static metrics hooks.

The boot timer stores only the timestamp captured at construction; it has no guest-visible state beyond the side effect of logging.

The generation identifier stores the current numeric identity, the guest address and interrupt line metadata, and the interrupt trigger. Snapshot persistence records the guest address and interrupt index so that restore can re-reserve the same resources and rebuild the object against the restored memory image.

Startup ordering is implied by construction: devices are allocated, potentially wired to host resources like standard input, then inserted into the map. Shutdown is not uniformly implemented inside this subtree; the keyboard controller’s reset path signals an external event to request process teardown. Steady-state operation is dominated by MMIO handling and, for UART, periodic event-loop callbacks.

## Failure Modes

Overlap insertion failure is explicit and safe: the map rejects conflicting ranges. Lookup failure for an MMIO access is silent from the device perspective and may manifest as guest-visible unpredictable values if higher layers do not define bus behavior for holes.

Lock poisoning on the mutex guarding a device is treated as fatal; the emulation assumes locks never fail except under catastrophic conditions. UART event path errors detach sources and log warnings, favoring continued operation without stuck polling on broken descriptors. FIFO overflow when injecting input returns resource errors to callers rather than corrupting memory.

The keyboard controller can fail to raise interrupts when the guest has disabled them; management operations surface errors when the buffer cannot accept a full sequence. Reset signaling failures increment metrics and log errors.

The AArch64 clock increments error metrics on malformed access shapes without attempting to interpret partial data.

The boot timer silently ignores malformed writes, which could hide guest bugs but avoids crashing the monitor for benign traffic.

Generation identifier creation can fail if random number generation fails, guest memory is not writable at the target, or resource allocation fails; these propagate as structured errors to the caller. Notification failures are logged per attempt. AML generation failures propagate as parsing or encoding errors from the AML toolkit.

Silent corruption risks are limited by strict access width checks on UART, clock, and keyboard paths; the generation identifier relies on correct guest physical addresses from the allocator. If the guest address were wrong, the monitor could write identity data into an unintended region—this is a contract obligation on the resource allocator and table builders, not something this layer can fully defend against in isolation.

## Operational Characteristics

The routing map uses a tree keyed by start address, giving logarithmic insertion and lookup costs in the number of regions. Each MMIO operation takes a lock on one device; contention appears only if multiple threads access the same device concurrently, which is uncommon for these legacy peripherals but possible for shared transports owned elsewhere.

Memory overhead is dominated by the device structs and the map nodes; no large buffers are held except the keyboard ring and UART FIFO within imported implementations.

Observability for legacy devices includes serialized metrics buckets flushed on demand for the keyboard controller, UART, and AArch64 clock. Logs capture serial detachment, buffer issues, and generation identifier steps. The boot timer emits an informational log line with timing breakdown when triggered.

Scaling bottlenecks are not a primary concern for this layer because device count is tiny and MMIO frequency for these peripherals is low compared with paravirtual I/O. The UART path is the busiest in interactive workloads due to console traffic.

## Design Rationale

The design favors a single unified routing abstraction so multiple device kinds—including the paravirtual MMIO transport not detailed here—share one decoding and locking model. That reduces duplicated address math and keeps guest physical decoding consistent.

Legacy devices are intentionally partial emulations: the keyboard controller implements enough command handling for operating systems to probe, reset, and deliver management keystrokes without simulating a full PS/2 keyboard matrix. That reduces code size and attack surface while preserving operational hooks operators rely on. The UART integrates with the host event loop only where asynchronous input matters, and it avoids registering standard input when it would be incompatible with process sandboxing, accepting reduced interactivity in those configurations.

Pseudo devices acknowledge that some observability signals do not warrant real hardware models. The boot completion marker trades strict architectural purity for a simple, zero-configuration signal that correlates wall time and CPU time from a fixed start marker captured during request handling.

The generation identifier exists because clustered enterprise software uses machine identity to gate features like distributed caches. Placing AML construction next to the device state keeps the guest-visible description aligned with the memory layout and interrupt assignment actually wired at runtime. Using a host interrupt trigger matches how other devices signal the guest through the virtual machine file descriptor. Persisting only address and interrupt line keeps snapshots compact while requiring the restore path to regenerate cryptographic material, which is desirable for freshness after resume.

The exclusion of paravirtual internals from this document reflects modularity: transport protocols and queue processing change far more often than platform scaffolding. Keeping the routing and legacy layers stable makes regression testing cheaper and clarifies which subsystems new platform features must extend.

The following diagram sketches how ACPI-related responsibilities split between the device semantics in this subtree and the surrounding monitor, without implying that those outer pieces live here.

```
  +------------------+       AML bytes, IRQ, GPA        +-------------------+
  | Firmware table   | <------------------------------ | Generation ID     |
  | assembly (outer) |                                 | logic (this area) |
  +------------------+                                 +---------+---------+
                                                                 |
                                                                 | writes identity
                                                                 v
                                                        +-------------------+
                                                        | Guest physical RAM |
                                                        +-------------------+
```

In steady state, the guest discovers the device through firmware, reads the identifier from memory, and subscribes to change notifications through the exposed interrupt. After migration or restore, outer layers rewrite memory if needed, then pulse the notification path so guest drivers refresh their caches.

## Diagram: Legacy UART Input Path

The UART bridges synchronous MMIO with asynchronous host input. Before the diagram, note that registration decisions depend on whether the input descriptor refers to an interactive terminal or a pipe, because some sandboxed deployments cannot poll certain standard input configurations.

```
  Host stdin / pipe                    Guest MMIO
       |                                   ^
       | read()                            | read/write
       v                                   |
  +------------+    FIFO    +---------------------------+
  | Event loop | <-------> | Emulated UART registers   |
  +------------+           +---------------------------+
       ^                                   ^
       | buffer-ready evt                   |
       +-----------------------------------+
```

After the diagram, recall that misconfiguration yields graceful detachment: sources unregister, metrics increment, and the guest may lose interactive input but the virtual machine keeps running.

---

Together, the routing layer, legacy peripherals, pseudo instrumentation, and ACPI-oriented generation identifier form a thin but essential platform rim around the paravirtual devices that carry most I/O throughput. The separation keeps protocol-heavy code paths apart from address decoding and long-lived platform quirks, which is appropriate for a security-conscious, minimal hypervisor core.
