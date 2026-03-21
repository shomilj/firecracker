# Virtual machine monitor architecture

1. **Purpose and scope**

   1.1. A Linux-KVM–based virtual machine monitor for a single guest: one contiguous guest physical address space (possibly multiple mapped regions), one fleet of virtual CPUs, and a deliberately small emulated device surface. The architecture is not a generic plugin hypervisor; control, emulation, and kernel virtualization primitives stay tightly coupled so lifecycle, security boundaries, and observable behavior stay predictable for operators and tests.

   1.2. Four cooperating planes structure reasoning about the system:

   - **Control plane** — configuration ingestion and validation, sequencing of boot, pause, resume, snapshot create/load, and translation of external actions into mutations of resource graphs, KVM objects, and device wiring.

   - **Execution plane** — per-vCPU threads in the hypervisor run loop, architectural exit demultiplexing, MMIO and (on x86) port I/O dispatch into emulated devices, and cooperative preemption via signals and shared mapped run-state.

   - **Device plane** — range-based MMIO routing (and a parallel legacy I/O path on x86) into virtio transports and a narrow set of legacy or pseudo devices; virtio ties guest memory descriptor rings to host-backed I/O (block, net, entropy, vsock, balloon).

   - **Persistence plane** — versioned aggregate guest state (hypervisor capability modifiers, architecture-specific VM and interrupt-controller state, per-vCPU register and model-specific buffers, virtio and ACPI-related device blobs, metadata describing memory layout and boot parameters) plus guest RAM capture, dirty-page–filtered capture, or out-of-band population coordinated with userfault-style lazy restore.

   1.3. Steady-state relationships (control vs execution vs devices vs memory):

```
                    +------------------ control requests ------------------+
                    |                                                    |
                    v                                                    |
   +-----------+  channel   +----------------+    irqfd / ioeventfd     +-----------+
   | API layer | ---------> | VMM controller | <-----------------------> |  devices  |
   +-----------+            +--------+-------+                         +-----+-----+
                                   |                                         |
                    pause/resume   |  register buses, memory, MSRs           | MMIO/PIO
                    snapshot       v                                         v
                            +------+------+    hypervisor run loop / CPU   +----+----+
                            |  KVM VM fd  | <---------------------------> | vCPU x N |
                            +-------------+                              +----------+
                                   ^
                                   | guest RAM slots, dirty logging
                                   |
                            +------+------+
                            | guest memory |
                            +--------------+
```

2. **KVM handle and capability negotiation**

   2.1. A top-level KVM object wraps the host device node. Creation checks API version compatibility and merges a baseline capability set with additions or removals driven by CPU templates (exposing or withholding specific extensions to present a coherent feature story to the guest). The maximum number of guest memory slots reported by the host is stored; each registered RAM region consumes one slot when installed.

   2.2. Capability gating affects what save/restore must serialize (for example hypervisor-specific feature bits), what exits appear during execution, and which fast paths (irqfd, immediate exit, architectural timers) are assumed available. Templates that hide features from the guest also reduce the contract surface for migration and snapshots.

3. **Guest memory: representation, backing, and dirty tracking**

   3.1. Guest physical memory is a collection of mapped regions. Regions may be backed anonymously, by a single growable memory file (memfd) for contiguous layouts, by snapshot files during restore, or with huge-page–aware mapping flags when configured. Optional per-page dirty bitmaps attach at region construction; when enabled, registration with the hypervisor sets the flag that enables kernel dirty logging for that slot.

   3.2. Registration walks regions in order, assigning monotonically increasing slot indices until the host limit is reached. Each registration pairs guest physical base, size, host userspace mapping base, and optional dirty-log selection. The in-process guest memory abstraction and the kernel slot view are updated together so removal or insertion cannot desynchronize what the guest maps versus what KVM believes is installed.

   3.3. Dirty tracking attaches kernel logging so migration and incremental snapshot strategies can query per-slot bitmaps and reset them between epochs. Reset walks regions and invokes the dirty-log query per slot—consuming the kernel’s notion of dirtiness for that interval and aligning host-side bitmaps with the hypervisor’s epoch.

   3.4. Snapshot-related memory operations include: dumping all guest pages; dumping only pages marked dirty in a supplied bitmap; describing regions for external fill (host virtual bases and offsets into backing storage) for cooperative restore; and keeping a userfault object alive in-process while guest physical pages fault in under lazy restore. Anonymous mappings use private anonymous maps with optional huge-page hints; file-backed snapshot restore uses private mappings of the snapshot file so the process does not share writable mappings unexpectedly.

   3.5. **Edge cases and invariants.** Slot exhaustion is a hard failure—operators must size region count against host limits. Partial failure during multi-region registration leaves policy-dependent cleanup requirements; the design favors failing the operation rather than leaving a half-registered layout. Dirty bitmaps are not automatically marked for virtio queue pages during steady operation; after a memory snapshot, activated virtio devices explicitly mark queue pages dirty so subsequent incremental strategies do not miss ring state.

4. **Virtual machine object**

   4.1. The VM object owns the hypervisor VM file descriptor. Creation retries a bounded number of times on interruptible failure: heavy hosts can return `EINTR` on VM creation even when no user signal is pending, because the kernel path is CPU-intensive and may check for signals; exponential microsecond backoff between attempts improves reliability without busy-looping.

   4.2. Architecture splits into shared slot management and architecture hooks: boot protocol (kernel, optional initrd, command line) placement in guest RAM, interrupt controller and timer integration, and firmware tables (MP-table or ACPI on x86; device and GIC-oriented setup on Arm). Bootstrapping writes the image into RAM, lays out boot metadata, and sets initial registers so the boot vCPU enters at the correct entry with a valid command line.

   4.3. **Security and performance tradeoffs.** Fewer, larger memory regions reduce slot pressure and mapping overhead but can worsen dirty granularity for migration. Huge pages improve TLB behavior at the cost of alignment and potentially larger zeroed reservations. Memfd-backed contiguous layouts simplify backing-file handling for snapshots at the cost of a single file offset space that must stay consistent with region enumeration.

5. **Virtual CPUs: threading, state machine, preemption, and teardown**

   5.1. Each virtual CPU is created against the VM fd, shares an “any CPU exited” notification, and communicates with the control plane through a dedicated request/response channel pair. The control side holds handles that can inject pause, resume, save-state, dump CPU configuration, and finish; the vCPU thread acknowledges with structured responses including success, error, or “not allowed in this state.”

   5.2. vCPU threads load a seccomp-BPF program appropriate to the vCPU category before entering emulation. Failure to install is treated as fatal—a deliberate fail-closed posture on the hot path.

   5.3. A compact state machine governs lifecycle. Emulation begins paused. Resume transitions to running, where an inner loop drives architectural hypervisor execution until an exit requires device work or an external event interrupts the loop. Pause requests break out of the inner loop; on x86, optional kvmclock control calls reduce guest softlockup watchdog false positives across stop/start. Save and CPU-configuration dump are only accepted while paused—running vCPUs reject them explicitly.

   5.4. **Immediate exit and kicks.** To pull a running vCPU out of the hypervisor run quickly (pause, snapshot preparation, debugger interactions), a real-time signal handler registered per thread sets the immediate-exit field in the mapped run region via thread-local storage. A memory fence ensures the write is visible before returning to the kernel. This cooperates with the hypervisor’s immediate-exit mechanism to bound control-operation latency without polling. If a run is requested while immediate exit is still set, the run is skipped and treated as interrupted—avoiding spinning with a stale flag.

   5.5. **Exit handling semantics.** MMIO read/write exits dispatch to the MMIO bus when attached. Halt and shutdown exits (and system events for reset/shutdown where applicable) translate to a stopped emulation result on the boot vCPU; other vCPUs may remain inside the run loop but cease forward progress in practice. Fail-entry, internal hypervisor error, and unexpected system events are treated as faults. `EINTR` from the run syscall after a kick clears immediate exit and surfaces as interrupted emulation so outer logic can process pause or snapshot signals. `EAGAIN` is treated as handled (continue). `ENOSYS` is treated as a hard fault (kernel failed instruction emulation). Architecture-specific exits delegate to peripheral emulation (for example I/O instructions, IRQ windows, sregs) through a thin peripheral facade.

   5.6. **Teardown protocol.** Guest-driven shutdown and reboot paths signal an exit event, set a process exit category, and enter an exited state that only accepts a finish command from the controller—avoiding races during process shutdown. VMM-initiated stop follows the same finish handshake so thread join order remains deterministic.

   5.7. Conceptual flow (paused ↔ running ↔ exited):

```
                    +---------+
            +------>| Paused  |<-------------------+
            |       +----+----+                    |
            |            | Resume                  | Pause / save / dump
            |            v                         |
            |       +----+----+                   |
            +-------+ Running |-------------------+
                    +-----------+
                         |
                         | guest halt / shutdown / system event (reset/off)
                         v
                    +-----------+
                    |  exited   |
                    +-----------+
```

6. **Device management, buses, and KVM integration**

   6.1. MMIO devices attach to a range-sorted map keyed by inclusive address ranges; insertion rejects overlaps. Each slot holds a mutex-protected dispatcher to concrete implementations—virtio MMIO transports, timers, serial UARTs (platform-dependent), pseudo devices, and test doubles.

   6.2. A resource allocator hands out guest-physical MMIO windows and IRQ lines under policy constraints. Virtio devices receive a fixed-size MMIO aperture; virtio configuration space lives at a known offset inside that window. On x86, ACPI AML may be synthesized so firmware and the OS discover virtio MMIO and IRQ assignments in a stable order—ordering matters for deterministic disk naming (root block first).

   6.3. **irqfd and ioeventfd.** irqfd ties host event file descriptors to guest IRQ lines: device models signal interrupts by writing an eventfd, and the kernel injects the virtual IRQ without a userspace exit for the common case. Virtio uses ioeventfd for queue kicks so guest writes that should notify the host translate into eventfd posts, shrinking exits on the notification path. Registration failures surface as device manager errors during bring-up.

   6.4. **x86 legacy I/O vs Arm MMIO.** A separate port-I/O bus carries legacy devices—PS/2 keyboard controller for reset and Ctrl-Alt-Del, serial UART for console—alongside the MMIO fabric. On Arm, comparable devices are MMIO-mapped (UART, platform RTC). The split ensures the routing layer feeds the correct bus implementation based on exit type.

   6.5. **Virtio in the steady state.** Devices share descriptor rings in guest memory, feature negotiation, interrupt suppression flags, and queue processing that respects memory ordering and notification coalescing. Block paths may use host asynchronous I/O engines where available; network bridges to TAP with optional rate limiters; vsock pairs Unix domain sockets; entropy exposes a virtio-rng path; the balloon supports resize and statistics. Resume explicitly kicks devices after vCPUs resume so queue processing reconnects with any pending notifications.

7. **Configuration layering and API semantics**

   7.1. Typed configuration describes each attachable resource—machine size, CPU templates, drives, NICs, vsock, balloon, entropy, boot source, logger, metrics, metadata service options—with serde-backed serialization for API and one-shot JSON bring-up.

   7.2. A higher-level resource aggregate holds builders and validated cross-field constraints: memory sizing ties to balloon and snapshot restore; CPU counts interact with templates; network definitions reference TAPs and MACs; the metadata service may be lazily allocated. Updates split into pre-boot vs post-boot categories enforced at the API translation layer.

   7.3. Single JSON initialization loads optional logger and metrics first, then applies machine configuration, CPU templates from inline definitions or external inputs, and each device stanza in deterministic order so failures surface before expensive allocations.

   7.4. The public action set enumerates permitted operations: boot-time-only configuration (drives, NICs at insert, entropy, vsock, balloon placement, machine sizing), runtime toggles (pause/resume, balloon resize and stats interval, NIC rate limit updates, block path swaps), introspection (full config dump, instance info, version), metadata document CRUD and token configuration, snapshot create/load, and platform controls such as injected three-finger salute on x86.

   7.5. Errors distinguish “not supported after boot,” “not supported before boot,” validation failures, and internal faults so orchestrators can classify retries. Metadata operations may return datastore limit errors separate from HTTP parsing issues inside the guest-facing server.

8. **Snapshotting, migration-oriented state, and format hardening**

   8.1. Saved state is an aggregate: hypervisor capability modifiers, architecture-specific VM registers and interrupt-controller state, per-vCPU register and model-specific buffers, virtio and ACPI-related device blobs, and metadata describing memory layout and boot configuration. vCPU threads must be paused so registers reflect a quiescent point; the controller broadcasts save events and collects responses with timeouts to detect deadlocks.

   8.2. The snapshot container wraps a versioned header (magic discriminates architecture families), semver version string, serialized payload, and optional CRC64 for integrity. Bincode with fixed integer encoding and explicit deserialization byte limits reduces attack surface against malicious snapshots (allocation bounds on untrusted input).

   8.3. Version checks enforce compatibility bands (major must match; minor must not exceed the reader’s supported minor). Unchecked load paths still validate magic before deserializing body data.

   8.4. **Memory snapshot paths.** Full memory dump writes all guest pages. Dirty-memory snapshot intersects guest memory with KVM’s dirty bitmap (after appropriate reset boundaries) to minimize I/O. Post-save, virtio queue pages are explicitly marked dirty in software because runtime did not track them in the guest dirty maps.

   8.5. **Restore.** Reconstruction reloads state, re-registers memory (optionally mapping snapshot files), reinstantiates devices, and loads hypervisor state before attaching vCPU threads. Startup remains paused until explicitly resumed. Lazy restore keeps a userfault object open and describes region mappings so another process can populate pages while the guest faults them in.

   8.6. **Tradeoffs.** Full snapshots are simpler but larger and slower. Dirty snapshots reduce size but depend on correct bitmap reset ordering and device marking. Lazy restore improves time-to-first-guest-code at the cost of operational complexity and fault-time latency spikes.

9. **MicroVM Metadata Service (MMDS) and in-guest HTTP**

   9.1. MMDS exposes a guest-accessible HTTP interface to a JSON document store modeling cloud instance metadata. A token subsystem supports session tokens with TTL semantics: clients obtain tokens with posted TTL headers, then present tokens on subsequent GETs; compatible header names allow AWS-like clients to work unmodified.

   9.2. The datastore enforces size limits and versioning to bound memory and to let clients detect updates. JSON Merge Patch updates documents partially; full replacements are also supported through the API surface.

   9.3. **Network plumbing** sits beside virtio NIC handling: a minimal userland IPv4/TCP/HTTP stack terminates connections destined to the metadata address, parses requests, routes them to datastore or token logic, and crafts responses—avoiding reliance on the host OS network stack for this path while keeping behavior predictable for agents.

   9.4. **Failure modes.** Token missing when required, invalid token, bad URI, wrong HTTP method, missing TTL on token acquisition, and missing JSON resources produce distinct error categories for metrics and troubleshooting.

10. **Minimal TCP/IP stack (“DUMBO”)**

   10.1. Supporting layers implement Ethernet, ARP, IPv4, UDP, and a deliberately small TCP subset—enough for listener accept, connection progression, and HTTP/1.1 request/response exchange sized for metadata workloads. Buffer traits abstract byte sources for parsing efficiency.

   10.2. The TCP endpoint coordinates retransmission and window behavior sufficient for localhost-adjacent traffic patterns; it is not aimed at general WAN throughput but at correctness and bounded resource use inside the monitor.

11. **io_uring integration**

   11.1. On capable host kernels, asynchronous block I/O can use io_uring with registered file descriptors and restricted operation sets. The wrapper builds submission/completion queues, probes mandatory opcodes (read/write at minimum), registers fixed files and eventfds, and applies restriction flags so the ring cannot be repurposed for unrelated syscalls.

   11.2. Backpressure appears as queue-full conditions distinct from permanent failures, allowing upper layers to throttle or retry. Integration ties into block virtio completion paths and host rate limiters.

12. **Rate limiting**

   12.1. Token-bucket limiters gate throughput and optionally burst separately for read and write directions. Buckets refill based on elapsed time toward configured ceilings; an initial one-time burst credit can absorb startup spikes. When tokens are exhausted, the limiter arms a timer file descriptor and transitions to blocked; embedding code must invoke the limiter’s event handler when the timer fires to unblock producers.

   12.2. The limiter is not self-driving—it expects an external poller—matching the epoll-centric architecture used for device I/O orchestration.

13. **Seccomp, signals, and process hygiene**

   13.1. Seccomp maps categorize threads (VMM control, API, vCPU, etc.) with independent BPF bytecode blobs deserialized from compact binary encodings with strict size caps. Installation uses the appropriate prctl/seccomp sequence for the target thread, and filter size respects kernel limits.

   13.2. Signal handlers cover vCPU kick (real-time signal offset), terminal resize forwarding where applicable, and fatal host signals mapped to distinct process exit codes. The design favors failing fast with coded exits over silent degradation.

14. **CPU configuration and architecture modules**

   14.1. CPU templates adjust hypervisor capabilities and guest CPUID or ID-register views to present coherent, sometimes vendor-specific, feature sets. Static templates encode known-good profiles; custom templates allow operator-defined filtering and feature bits with serde support.

   14.2. Architecture modules encapsulate differences: x86 builds MP-table or ACPI structures, programs APIC/IOAPIC expectations, and handles MSR filtering; Arm programs GIC state, aligns MPIDR views with the hypervisor, and uses MMIO UART/RTC where guest kernels expect them. Shared per-architecture facades hide raw ioctl details behind typed errors.

15. **Logging, metrics, and observability**

   15.1. Logger configuration can be applied early in bring-up; metrics initialization likewise. Metrics increment counters for device events, kvmclock control failures on pause/resume paths, and error categories—feeding external monitoring without guest cooperation.

16. **Optional debugging**

   16.1. When compiled with GDB support, additional channels export a GDB stub: duplicated vCPU descriptors, stop/start coordination with the vCPU state machine, and kvmclock adjustments around debugger-induced pauses. This augments but does not replace the primary hypervisor execution flow.

17. **RPC adapter and the outer event loop**

   17.1. The RPC layer maps external actions to typed operations on the VMM, enforcing boot-state gates (pre-boot-only vs post-boot-only), aggregating errors into stable categories, and coordinating long-running paths such as snapshot and resume that require device kicks and vCPU synchronization.

   17.2. The outer process typically combines an epoll-driven event manager with device subscribers: virtio queues, rate limiters, serial I/O, and network backends register file descriptors; readiness drives handler entry points while vCPU threads progress independently except at pause/snapshot barriers.

18. **Builder and resource graph**

   18.1. Construction walks from validated resources to a concrete VM: KVM object, VM object, memory regions, device graph, ACPI where applicable, and vCPU list. Boot attaches MMIO and legacy buses to vCPUs, applies seccomp maps per thread category, and transitions instance state through paused-at-thread-start to running-on-resume.

   18.2. **Restore-from-snapshot** rebuilds the resource graph from serialized state, reuses compatibility-checked versions, and leaves the instance paused until the API resumes—allowing orchestration to reconnect host resources (TAP, disk paths) before guest execution.

19. **Persistence helpers and cross-cutting serde**

   19.1. Device and subsystem state implement a common persistence trait pattern: serialize for snapshot, deserialize with explicit error typing, and reconcile with live KVM objects during restore. Aggregates compose these shards into the microVM state blob written beside memory images.

20. **Testing and validation hooks**

   20.1. Test utilities provide mock resource sets, synthetic devices on the bus, and kernel-noise harnesses where needed. Integration tests exercise device plug-in, io_uring paths, and cross-cutting snapshot scenarios—guarding regressions in aggregate behavior rather than only unit-isolated components.

21. **Operational summary: latency, safety, and coupling**

   21.1. **Latency.** Fast interrupt injection and queue kicks rely on kernel-side file descriptors; slow paths fall back to MMIO exits and string processing in device models. Pause and snapshot latency depend on kicking every vCPU out of the run loop promptly—hence real-time signals and immediate exit.

   21.2. **Safety.** Seccomp reduces syscall exposure; snapshot deserialization bounds limit memory bombs; CRCs optional detect storage corruption; virtio queue memory is validated on activation to keep guest-driven pointers within guest RAM.

   21.3. **Coupling.** The monitor intentionally trades extensibility for a linear story: device ordering affects discovery; template choice affects CPUID and save payloads; memory layout choices affect migration compatibility. Understanding those couplings is essential when changing any one layer.
