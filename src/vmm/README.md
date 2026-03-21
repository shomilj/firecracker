# Virtual machine monitor architecture

1. **Purpose and scope**

   1.1. The crate implements a Linux-KVM–based virtual machine monitor oriented around a single guest: one address space, one set of virtual CPUs, and a deliberately small device surface. The design favors deterministic control paths, explicit lifecycle transitions, and tight coupling between configuration, emulation, and the KVM execution model rather than a pluggable hypervisor framework.

   1.2. Work splits naturally into four cooperating planes:

   - **Control plane** — ingestion of configuration, validation, sequencing of boot and pause/resume/snapshot operations, and translation of external requests into mutations of in-memory resource graphs and KVM objects.

   - **Execution plane** — per-vCPU threads running the `KVM_RUN` loop, architectural exit handling, MMIO/PIO dispatch into emulated devices, and cooperative preemption via signals and shared `kvm_run` state.

   - **Device plane** — MMIO routing (and PIO on x86) to virtio transports and a small set of legacy or pseudo devices; virtio queues tie guest memory to host-backed I/O paths (block, net, entropy, vsock, balloon).

   - **Persistence plane** — serializable aggregate guest state (KVM VM object, vCPU registers and model-specific state, device virtio state, ACPI bookkeeping where applicable) plus guest RAM capture or lazy population via userfaultfd-style mechanisms.

   1.3. A lightweight ASCII view of steady-state runtime relationships:

```
                    +------------------ control requests ------------------+
                    |                                                    |
                    v                                                    |
   +-----------+  channel   +----------------+    irqfd/ioeventfd    +-----------+
   | API layer | ---------> | VMM controller | <--------------------> |  devices  |
   +-----------+            +--------+-------+                       +-----+-----+
                                   |                                       |
                    pause/resume   |  register buses, memory, MSRs         | MMIO/PIO
                    snapshot       v                                       v
                            +------+------+    KVM_RUN loop per CPU   +----+----+
                            |  KVM VM fd  | <------------------------> | vCPU x N |
                            +-------------+                            +----------+
                                   ^
                                   | guest RAM slots, dirty logging
                                   |
                            +------+------+
                            | guest memory |
                            +--------------+
```

2. **KVM and guest memory**

   2.1. A top-level KVM handle wraps the host `/dev/kvm` object. On creation it validates API version compatibility and merges a baseline capability set with additions or removals driven by CPU templates (for example exposing or withholding specific KVM extensions). The maximum number of guest memory slots exposed by the host is recorded because each RAM region consumes one slot when registered with `KVM_SET_USER_MEMORY_REGION`.

   2.2. Guest physical memory is represented as a collection of mapped regions with optional per-page dirty tracking bitmaps. Regions can be backed anonymously, by a single growable memory file (memfd) for contiguous layouts, by snapshot files during restore, or with huge-page–aware mapping flags when configured. Dirty tracking attaches `KVM_MEM_LOG_DIRTY_PAGES` so migration and incremental snapshot strategies can query dirty logs and reset bitmaps between epochs.

   2.3. Registration walks regions in order, assigning monotonically increasing slot indices until the host limit is reached. Each registration pairs a guest physical base, size, host userspace pointer, and optional dirty-log flag. The monitor keeps the in-process guest memory abstraction and the KVM slot view synchronized: inserting a region updates both the canonical mapping abstraction and the kernel’s view.

   2.4. For snapshots, memory can be dumped wholesale, dumped using KVM’s dirty bitmap to minimize I/O, or described for out-of-band filling. Descriptions can include host virtual addresses and offsets into backing files so an external process knows where to place pages—this meshes with lazy restore approaches where the guest starts before every page is resident.

3. **Virtual machine object**

   3.1. The VM object owns the `KVM_CREATE_VM` file descriptor. Creation retries a small number of times if interrupted, mirroring production experience that heavy hosts can return `EINTR` even without a pending signal on this path.

   3.2. The object splits into architecture-independent slot management and architecture-specific hooks: MP table or ACPI generation on x86, interrupt controller and timer wiring on Arm, and placement of kernel, initrd, and command line according to a fixed memory map. Bootstrapping loads the kernel image into guest RAM, optionally loads an initrd, writes boot protocol data structures, and configures registers so the first vCPU starts at the correct entry with a valid command line.

   3.3. Dirty bitmap reset walks all regions and invokes KVM’s dirty log ioctl per slot—this is used around snapshot boundaries to align host-side bitmaps with KVM’s internal epoch.

4. **Virtual CPUs: threading model, state machine, and preemption**

   4.1. Each virtual CPU is created against the VM fd, receives a shared “any CPU exited” event notification, and communicates with the control plane through a dedicated request/response channel pair. The control side holds handles that can inject `Pause`, `Resume`, `SaveState`, `DumpCpuConfig`, and `Finish` events; the vCPU thread acknowledges with structured responses including success, error, or “not allowed in this state.”

   4.2. vCPU threads load a seccomp-BPF program specific to the vCPU category before entering emulation. Failure to install is treated as fatal—reflecting a fail-closed posture for syscall surface reduction on the hot path.

   4.3. A small state machine governs lifecycle: emulation begins in `Paused`. A resume transition moves to `Running` where a tight loop calls into architectural `KVM_RUN` handling until an exit requires device emulation or an external event interrupts the loop. Pause requests are honored by breaking out of the inner emulation loop, optionally adjusting kvmclock-related controls on x86 to avoid guest watchdog false positives across stop/start, and acknowledging pause. Save and CPU configuration dump operations are only accepted while paused—running vCPUs reject them explicitly.

   4.4. To kick a running vCPU out of `KVM_RUN` quickly (for pause, snapshot setup, or debugger interactions), a real-time signal handler registered per thread flips the `immediate_exit` field in the mapped `kvm_run` structure via thread-local storage. A memory fence ensures the write is visible before returning to KVM. This cooperates with KVM’s immediate-exit mechanism to bound latency of control operations without busy-waiting.

   4.5. Guest-driven shutdown and reboot paths are detected via KVM exit reasons such as system shutdown or reset. The first vCPU surfaces these events; others may remain inside `KVM_RUN` but become inactive from a scheduling perspective. The teardown protocol writes the shared exit event, signals an overall process exit code category, and waits for an explicit `Finish` event from the controller to avoid races during process shutdown.

   4.6. Conceptual vCPU state flow:

```
                    +---------+
            +------>| Paused  |<-------------------+
            |       +----+----+                    |
            |            | Resume                 | Pause / save paths
            |            v                        |
            |       +----+----+                   |
            +-------+ Running |------------------+
                    +-----------+
                         |
                         | guest halt / shutdown exit
                         v
                    +-----------+
                    |  exited   |
                    +-----------+
```

5. **Device management and buses**

   5.1. MMIO devices attach to a range-sorted map keyed by inclusive address ranges; insertion rejects overlaps. Each slot holds a mutex-protected enum that dispatches reads and writes to concrete device implementations—virtio MMIO transports, timers, serial UARTs (platform-dependent), and test doubles.

   5.2. A resource allocator hands out guest-physical MMIO windows and IRQ lines under policy constraints. Virtio devices receive a fixed-size MMIO aperture; the virtio specification’s configuration space lives at a known offset inside that window. For x86 guests, ACPI AML may be synthesized so firmware and the OS discover virtio MMIO and IRQ assignments in a stable order—ordering matters for deterministic disk naming (root block device first).

   5.3. KVM irqfd ties host-side event file descriptors to guest IRQ lines: device models signal interrupts by writing to an eventfd, and the kernel injects the virtual IRQ without exiting to userspace. Complementarily, virtio uses ioeventfd for queue kicks: guest writes that should kick the host queue handler are translated into eventfd posts, reducing VM exits for the common queue-notification path.

   5.4. On x86, a separate PIO bus carries legacy devices—PS/2 keyboard controller for reset/Ctrl-Alt-Del, serial UART for console—behind a parallel manager. On Arm, some of these migrate to MMIO UART and RTC models. The split ensures the KVM routing layer (`KVM_EXIT_IO` vs `KVM_EXIT_MMIO`) feeds the correct bus implementation.

   5.5. Virtio devices share a common transport: descriptor rings in guest memory, feature negotiation, interrupt suppression flags, and queue processing that respects memory ordering and notification rules. Block devices may use host-side asynchronous I/O engines where available; network devices bridge to TAP devices with rate limiters; vsock pairs Unix domain sockets; entropy exposes a virtio-rng path; the balloon supports resize and statistics.

6. **Configuration layering**

   6.1. Typed configuration structs describe each attachable resource—machine size, CPU templates, drives, NICs, vsock, balloon, entropy, boot source, logger, metrics, MMDS options—with serde-backed serialization for API and one-shot JSON bring-up.

   6.2. A higher-level resource aggregate holds builders and validated cross-field constraints: memory sizing ties to balloon and snapshot restore; CPU counts interact with templates; network definitions reference TAPs and MACs; MMDS may be lazily allocated. Updates fall into pre-boot vs post-boot categories enforced at the API translation layer.

   6.3. Single JSON initialization loads optional logger and metrics first, then applies machine configuration, CPU templates from inline definitions or external paths, and each device stanza in a deterministic order so failures surface before expensive allocations.

7. **External interface semantics**

   7.1. The public action set enumerates every permitted operation: boot-time-only configuration (drives, NICs at insert, entropy, vsock, balloon placement, machine sizing), runtime toggles (pause/resume, balloon resize and stats interval, NIC rate limit updates, block path swaps), introspection (full config dump, instance info, version), MMDS CRUD and token configuration, snapshot create/load, and platform-specific controls such as injected three-finger salute on x86.

   7.2. Errors distinguish “not supported after boot,” “not supported before boot,” validation failures, and internal VMM faults so orchestrators can classify retries. MMDS operations may return datastore limit errors separate from HTTP-level issues inside the guest-facing server.

8. **Snapshotting and migration-oriented state**

   8.1. Saved state is an aggregate: KVM capability modifiers, architecture-specific VM registers and interrupt controller state, per-vCPU register and model-specific buffers, virtio and ACPI-related device blobs, and metadata describing memory layout and boot configuration. vCPU threads must be paused so registers reflect a quiescent point; the controller broadcasts save events and collects responses with timeouts to detect deadlocks.

   8.2. The snapshot container format wraps a versioned header (magic discriminates architecture families), semver version string, serialized payload, and optional CRC64 for integrity. Bincode with fixed integer encoding and explicit deserialization byte limits reduces attack surface against malicious snapshots.

   8.3. Restore reconstructs resources, re-registers memory (optionally mapping snapshot files), reinstantiates devices, and loads KVM state before attaching vCPU threads. Guest startup remains paused until explicitly resumed, allowing orchestration to reconnect network or validate disks.

   8.4. Userfaultfd-based lazy memory ingestion can keep a snapshot file open inside the process while guest physical pages fault in; mappings describe where external tooling should place data relative to host addresses.

9. **MicroVM Metadata Service (MMDS) and in-guest HTTP**

   9.1. MMDS provides a guest-accessible HTTP interface to a JSON document store, modeling cloud instance metadata. A token subsystem supports IMDSv2-style session tokens with TTL semantics: clients obtain tokens with posted TTL headers, then present tokens on subsequent GETs; compatible header names allow AWS-like clients to operate unmodified.

   9.2. The datastore enforces size limits and versioning to bound memory and to let clients detect updates. JSON Merge Patch updates documents partially; full replacements are also supported through the API surface.

   9.3. Network plumbing for MMDS sits beside virtio NIC handling: a minimal userland IPv4/TCP/HTTP stack terminates connections destined to the metadata address, parses requests, routes them to the datastore or token logic, and crafts responses. This avoids pulling a full network stack into the host OS while preserving predictable behavior for guest agents.

10. **Minimal TCP/IP stack (“DUMBO”)**

   10.1. Supporting layers implement Ethernet, ARP, IPv4, UDP, and a deliberately small TCP subset—enough for listener accept, connection state progression, and HTTP/1.1 request/response exchange sized for metadata workloads. Buffer traits abstract byte sources for zero-copy-friendly parsing.

   10.2. The TCP endpoint coordinates retransmission and window concepts sufficient for localhost-adjacent traffic patterns; it is not a general-purpose high-throughput stack but a correctness-focused component scoped to the monitor’s needs.

11. **io_uring integration**

   11.1. Where host kernels support it, asynchronous block I/O can leverage io_uring with registered file descriptors and restricted operation sets. The wrapper builds submission/completion queues, probes for mandatory opcodes (read/write at minimum), registers fixed files and eventfds, and applies restriction flags so the ring cannot be abused for unrelated syscalls.

   11.2. Backpressure surfaces as queue-full errors distinct from permanent failures, allowing upper layers to throttle or retry. This integrates with block virtio completion paths and host rate limiters.

12. **Rate limiting**

   12.1. Token-bucket rate limiters gate throughput and optionally burst separately for read and write directions. Buckets refill based on elapsed time toward configured ceilings; an initial one-time burst credit can absorb startup spikes. When tokens are exhausted, the limiter arms a timerfd and transitions to a blocked state; integration contracts require the embedding event loop to invoke a handler when the timer fires to unstick producers.

   12.2. The implementation is not self-driving—it expects an external poller—matching the monitor’s epoll-centric architecture.

13. **Seccomp, signals, and process hygiene**

   13.1. Seccomp maps categorize threads (`vmm`, `api`, `vcpu`, etc.) with independent BPF bytecode blobs deserialized from compact binary encodings with strict size caps. Installation uses `prctl`/`seccomp` appropriately for the target thread, and filter length is bounded to kernel limits.

   13.2. Signal handlers cover vCPU kick (SIGRTMIN offset), terminal resize forwarding where applicable, and fatal host signals mapped to distinct process exit codes for operability. The design prefers failing fast with coded exits over silent degradation.

14. **CPU configuration and architecture modules**

   14.1. CPU templates adjust KVM capabilities and guest CPUID or ID-register views to present coherent, sometimes vendor-specific, feature sets to the guest. Static templates encode known-good profiles; custom templates allow operator-defined filtering and feature bits with serde support.

   14.2. Architecture modules encapsulate differences: x86 builds MP-table or ACPI structures, programs APIC/IOAPIC locations, and handles MSR filtering; Arm programs GICv3 state, sets MPIDR views consistently with KVM, and uses MMIO-mapped UART/RTC where expected by kernels. Shared per-architecture vCPU and VM facades hide raw hypervisor ioctl details behind typed error paths.

15. **Logging, metrics, and observability**

   15.1. Logger configuration can be applied early in bring-up; metrics initialization likewise. Metrics increment counters for device events, vCPU kvmclock control failures, and error categories—feeding external monitoring without guest cooperation.

16. **Optional debugging**

   16.1. When compiled with GDB support, additional channels allow the monitor to export a GDB stub: duplicated vCPU file descriptors, stop/start coordination with the vCPU state machine, and kvmclock adjustments around debugger-induced pauses. This augments but does not replace the primary KVM execution flow.

17. **Testing and validation hooks**

   17.1. Test utilities provide mock resource sets, synthetic devices on the bus, and noisy-kernel harness scripts where needed. Integration tests exercise device plug-in, io_uring paths, and cross-cutting snapshot scenarios—guarding regressions in the aggregate behavior described above.
