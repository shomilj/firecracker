<picture>
   <source media="(prefers-color-scheme: dark)" srcset="docs/images/fc_logo_full_transparent-bg_white-fg.png">
   <source media="(prefers-color-scheme: light)" srcset="docs/images/fc_logo_full_transparent-bg.png">
   <img alt="Firecracker Logo Title" width="750" src="docs/images/fc_logo_full_transparent-bg.png">
</picture>

Mission and product context: [CHARTER.md](CHARTER.md), [FAQ.md](FAQ.md), [docs/design.md](docs/design.md).

The sections below are a **systems architecture** view of this repository and point to deeper per-tree documentation.

---

## 1. Problem shape and positioning

1.1. **Isolation model.** Guests run with hardware virtualization (KVM). The VMM does not aim to emulate a full PC; it exposes a deliberately minimal machine: virtio block and network, vsock, optional entropy, serial where policy allows, and (on x86) a narrow set of legacy devices needed for boot and reset semantics. Fewer emulated surfaces reduce memory footprint, startup time, and attack surface. The design trades breadth of hardware compatibility for predictability: operators reason about a small virtio-centric contract rather than a moving catalog of chipset peripherals.

1.2. **Operational model.** An operator (or orchestrator) starts the monitor, then drives **configuration through a control-plane API** over a Unix-domain socket. The fast path is guest execution on vCPU threads; the API thread is explicitly off the emulation hot path. Production deployments typically wrap process startup in a companion **jailer** that creates namespaces, cgroups, chroot, and privilege boundaries before executing the monitor as an unprivileged principal. That split keeps configuration latency and HTTP parsing off the paths that interact with KVM and guest memory at high frequency.

1.3. **Threat stance.** Untrusted code is assumed to run inside the guest. vCPU threads are treated as hostile once the guest runs. Data leaving the guest toward the host (network frames, block I/O, vsock bytes) crosses **trust boundaries** where copying, policy, and rate limiting apply. Seccomp **BPF** filters constrain which host syscalls each thread may invoke, tightened further after initialization so the attack surface shrinks over the lifecycle. The model assumes a compromised guest may attempt to abuse MMIO, malformed virtio rings, or timing; device code is written to fail closed or terminate rather than loop on inconsistent guest state.

---

## 2. Structural decomposition

2.1. **Process and threads.** Conceptually four roles coexist in one process:

```
                    +------------------+
                    |   API thread     |  HTTP control plane (configure, start,
                    |                  |  pause, snapshot, metrics hooks)
                    +--------+---------+
                             |
                             v
                    +------------------+
                    |   VMM thread     |  Event loop: virtio devices, timers,
                    |                  |  MMIO dispatch, I/O threading, MMDS
                    +--------+---------+
                             ^
         guest I/O +         |
         MMIO exits ----------+--------> vCPU thread(s)
                             |          KVM_RUN loop per vCPU
                             |
                    +------------------+
                    |  I/O / workers   |  (optional helpers for block/network)
                    +------------------+
```

1. The **API thread** serves JSON-over-HTTP requests and never executes guest instructions.

2. The **VMM thread** runs an event-driven reactor tying together KVM fds, virtio queues, timers, and signals.

3. **vCPU threads** issue `KVM_RUN` in a loop. Exits funnel device work back into the VMM’s world (MMIO, interrupts, virtio kicks).

4. Additional **worker** or **I/O** paths may serve virtio backends (e.g., tap, files, io_uring) so guest-driven work does not block the central poll loop more than necessary.

2.2. **Crate layering (conceptual).**

- A **binary front-end** parses arguments, wires logging and metrics, loads seccomp metadata, and either connects an API server or runs a minimal non-API path (e.g., restore-from-snapshot flows).

- A **VMM library** encapsulates KVM interaction, guest memory layout, CPU templates, device models, virtio implementations, snapshots, ballooning, and optional debugging (e.g., GDB stub behind a feature flag).

- **Shared utilities** cover argument parsing, errno helpers, signal handling, and other cross-cutting host concerns.

- **Standalone tools** compile seccomp policies, edit snapshots, rebase snapshot files, dump or verify CPU templates, and assist development (e.g., tracing-related helpers).

2.3. **Host dependencies.** The VMM relies on Linux KVM ioctls, guest RAM abstractions, virtio and vhost-user front-end crates where applicable, and cryptography libraries where guest-facing services need AEAD. Architecture-specific code paths exist for x86_64 and aarch64 (interrupt controllers, boot protocol, CPUID or device-tree).

2.4. **Failure domains (API vs VMM vs vCPU).** Treating these roles as separate failure domains clarifies blast radius and debugging:

1. **API domain.** Request parsing, validation, and resource graph updates can fail without corrupting KVM state if the VMM rejects invalid transitions. Bugs here tend to surface as HTTP errors, failed configuration, or process exit before guest execution. The API thread does not hold the guest run loop; severe misuse may still stress shared locks, but it is not on the same path as per-instruction emulation.

2. **VMM / reactor domain.** The event loop coordinates devices, timers, and signals. Failures here—deserialization errors during restore, device activation errors, poll-loop panics—can leave the instance inconsistent or torn down entirely. This domain owns virtio queue processing for in-process backends, MMDS bridging, and interrupt injection. Many fault paths are process-fatal by design when guest-supplied state is deemed inconsistent.

3. **vCPU domain.** Each vCPU thread runs the architectural hypervisor loop. Bugs or hostile guest behavior that triggers unexpected exit types, mis-specified MMIO, or internal KVM errors surface here first. Seccomp profiles differ from the API thread; syscall denial on the hot path is a distinct failure mode. Cross-vCPU coordination (pause, migration preparation) relies on signals and shared memory; races or ordering bugs manifest as stuck vCPUs or failed snapshot quiesce.

4. **Crossing domains.** Control operations (pause, resume, snapshot) intentionally serialize work across API → VMM → vCPU boundaries. A failure during handoff (e.g., timeout waiting for vCPU pause) is attributed to the synchronization path, not necessarily to a single buggy device.

---

## 3. Control plane and configuration lifecycle

3.1. **Pre-boot configuration.** Before the guest runs, the API accepts a schema-driven description: machine topology (vCPUs, RAM), kernel/initrd or boot ELF, block devices and their host backing, network interfaces and tap attachments, vsock, entropy device, balloon, logging and metrics sinks, MMDS JSON, rate limiters, and CPU template selections. Validation ensures coherent combinations (device IDs, capacity limits). Invalid combinations fail before any KVM VM object is fully committed, reducing the need for partial rollback in the steady state.

3.2. **Boot and run.** Starting the instance allocates guest memory maps, places the kernel per the chosen loader path, sets CPUID or device-tree exposure according to templates, and enters the KVM run state. From then on, most interaction is **exit-driven**: I/O instructions and virtio virtqueues generate work on the VMM thread and related backends. Boot-time errors (loader failure, missing image) are surfaced to the operator without starting guest execution.

3.3. **Runtime operations.** Pause/resume, device hotplug semantics where supported, drive rescan and backing swap, balloon inflation/deflation, and signal-driven shutdown integrate with the event loop. Metrics and logs flush through configured sinks (often FIFOs for log aggregation). Pause forces vCPUs out of `KVM_RUN` so device and memory state can be observed consistently.

3.4. **Snapshots (save).** Memory and device state serialize to a versioned format with explicit compatibility rules. The save path captures an aggregate of hypervisor-visible state—VM configuration, interrupt controllers, vCPU registers and buffers, virtio transports and device-specific blobs, and memory either wholesale or filtered by dirty bitmaps depending on strategy. Integrity and versioning checks allow loaders to refuse incompatible or tampered inputs when options require it.

3.5. **Snapshot restore path.** Restore reconstructs a **paused** microVM from a saved state file and a chosen memory backend; it is not identical to cold boot.

   1. **State load.** The snapshot file is opened and deserialized with version checking. The embedded machine descriptor drives validation: CPU count, RAM size, SMT flag, CPU template, huge-page intent, and the serialized device and vCPU sub-states must form a coherent whole before building continues.

   2. **Operator overrides.** Network attachments can be remapped so restored instances attach to new TAP names or host interfaces without editing the snapshot blob; unknown device IDs in override lists fail the operation early.

   3. **Resource reconciliation.** Machine configuration in the running resource graph is updated to match the snapshot (vcpu count, memory size, template, dirty-tracking flag, etc.) so later API-visible state matches restored reality.

   4. **Sanity checks.** Cross-field validation (manufacturer ID, structural consistency) runs before any KVM object is created from the restored descriptor, reducing the chance of building a half-valid VM.

   5. **Guest memory materialization.** Two conceptual backends exist: **file-backed** restore maps the saved memory image (or equivalent) into guest regions with optional dirty-page tracking; **userfaultfd-backed** restore allocates anonymous regions, registers them with `userfaultfd`, and hands the kernel UFFD object to an external process over a Unix socket so pages can be filled on fault according to an out-of-band protocol. Huge-page configurations interact with which backend is legal—some combinations require the lazy path rather than mapping a snapshot file directly.

   6. **VMM construction.** With guest memory and optional UFFD handle in hand, the builder instantiates the VM, reinstalls devices from serialized state, and attaches to the event loop. The instance remains paused until the operator resumes, allowing external setup (network, cgroups) before execution.

3.6. **Restore vs boot.** Restore skips guest bootloader activity and replays hypervisor-level state; operators should treat compatibility (kernel, virtio, feature bits) as part of their release process. Network and block paths may need host-side resources recreated even when guest RAM is faithful.

---

## 4. Device and I/O architecture

4.1. **Virtio-first.** Network and block use virtio with Firecracker’s own device models and queue handling. Rate limiting applies at the boundary where host resources are consumed (token-bucket style controls on bandwidth and ops/sec), so fairness is enforced in the VMM rather than inside the guest. In-process virtio walks descriptor rings in the VMM thread or worker paths, with explicit dirty tracking of ring memory for migration-related scenarios.

4.2. **Vsock.** A virtio-vsock path bridges guest AF_VSOCK to host Unix sockets, enabling agent protocols without exposing new host attack surface beyond the configured socket endpoints.

4.3. **MMDS.** A minimal HTTP-oriented metadata tree can be exposed to the guest over a dedicated virtio-net channel (not general Internet access). Content is entirely operator-defined JSON; the feature is optional and gated by configuration. The implementation terminates a constrained HTTP/TCP subset inside the VMM and classifies virtio-net traffic toward either the TAP or the metadata path.

4.4. **Entropy.** A virtio-rng style path can inject host-derived entropy into the guest where supported, subject to platform policies.

4.5. **Balloon.** Virtio balloon allows the host to reclaim guest-physical pages cooperatively, with careful synchronization so guest visibility and VMM bookkeeping stay consistent. Balloon activity interacts with memory management features such as userfaultfd registration when both are in play.

4.6. **Vhost-user positioning.** For block (and analogous net scenarios where enabled), **vhost-user** moves virtqueue processing into a **separate host process** connected via a Unix-domain socket. The VMM retains ownership of the virtio **MMIO transport**—discovery, feature negotiation, queue programming, and configuration space—while the **backend** handles descriptor chains and host I/O. Conceptually:

   1. **When to use it.** Offload queue processing when an external storage or networking daemon already implements the vhost-user protocol, or when isolating backend crashes from the VMM process is worth the IPC overhead.

   2. **Division of responsibility.** Kicks and interrupts cross the protocol boundary; the transport still reflects virtio MMIO semantics to the guest, but interrupt-status behavior may differ slightly when the remote side cannot mirror every bit of legacy virtio status state— the transport adapts so guests remain forward-compatible.

   3. **Tradeoffs.** Simplicity and snapshot uniformity favor in-process virtio; vhost-user adds socket negotiation, protocol versioning, and operational dependency on a second process. Some snapshot scenarios are restricted when backends are externalized, because not all out-of-process state can be captured in the same versioned snapshot bundle as in-process devices.

   4. **Relationship to KVM fast paths.** `ioeventfd` still maps queue doorbells so many guest notifications avoid userspace; vhost-user changes where descriptors are consumed, not the guest-visible virtio programming model.

---

## 5. Memory and CPU presentation

5.1. **Guest RAM.** RAM is mapped for the guest with architecture-specific alignment and dirty tracking for migration/snapshot. Regions may be anonymous, file-backed, or tied to snapshot restore files; registration consumes KVM memory slots until host limits are reached.

5.2. **CPU templates.** Templates adjust visible CPUID (x86) or device-tree CPU features (ARM) so guests see a stable, supportable feature set across heterogeneous hosts. Custom templates can be synthesized from baseline hardware fingerprints using helper tooling.

5.3. **ACPI.** Minimal ACPI artifacts may be generated to satisfy guest kernels’ expectations without turning Firecracker into a full PC emulator.

5.4. **Userfaultfd (UFFD) and demand paging (conceptual).** Linux **userfaultfd** lets userspace handle page faults on registered memory ranges. In Firecracker’s restore flow, anonymous guest regions can be registered with a UFFD object and handed to a cooperating **page-server** process. When the guest touches a not-yet-resident page, the kernel delivers a fault event to that handler, which can populate the page (for example from a compressed image, remote store, or deduplicated cache) before guest execution resumes. This enables **lazy** or **demand-filled** memory: the guest can start before every page of RAM is present locally.

   1. **Demand paging mental model.** Traditional restore maps an entire memory image upfront; lazy restore trades **faster time-to-resume** and **lower immediate RSS** for **fault-time latency** and **runtime dependency** on the page server.

   2. **Coexistence with balloon.** UFFD features such as `EVENT_REMOVE` integrate with advice from balloon-style reclamation so removed pages can be tracked consistently when both mechanisms are active.

   3. **Operational implications.** The page server becomes part of the availability story: if it stalls or mis-handles the protocol, guest progress degrades or fails. This is distinct from file-mapped restore where the kernel and storage layer dominate I/O behavior.

5.5. **GDB stub (optional).** When built with the GDB feature, the monitor can expose a **remote GDB** interface over a configurable Unix socket. The debugger attaches to the guest’s execution state **from outside** the normal API-driven run loop: vCPU threads participate in stop/cont, register read/write, and breakpoint/single-step flows coordinated with the stub. This path is intended for **development and diagnosis**, not production guest management—it increases trusted computing base and operational surface, so it remains opt-in at compile time and is typically absent from hardened production binaries. Operators should treat debug sockets with the same care as any powerful local control channel.

---

## 6. Security architecture

6.1. **Seccomp stages.** Syscall filtering is applied per-thread with different profiles for API, VMM, and vCPU contexts. The compiler toolchain turns JSON policy into BPF programs embedded at build time; runtime installation happens before guest code executes on vCPUs. A syscall denied on the API thread may surface as a failed request; the same denial on a vCPU thread during emulation is more likely to be fatal, reflecting differing trust assumptions.

6.2. **Jailer.** The jailer sets up namespaces, cgroup limits, resource visibility, and file descriptor inheritance, then drops privileges before `exec` into the monitor. The monitor inherits only the capabilities required for KVM and configured resources.

6.3. **Cgroups and quotas.** Beyond the jailer, operators can bind CPU and NUMA affinity via cgroups so noisy neighbors are mitigated and scheduling stays predictable.

---

## 7. Observability

7.1. **Logging.** Structured logs go to a configured destination with explicit levels; panic hooks attempt to flush metrics and restore terminal state where relevant.

7.2. **Metrics.** Counters and gauges cover vCPU behavior, devices, API usage, and faults. Periodic emission plus event-driven updates give operators time-series signals without guest cooperation.

7.3. **Tracing (optional build).** Instrumentation hooks can compile in for deeper latency analysis when enabled via features.

---

## 8. Testing and validation

8.1. **Rust tests.** Unit and integration tests in the VMM and tools exercise KVM interactions (where the CI environment permits), virtio logic, and serialization.

8.2. **Python integration tests.** A host-driven framework boots real microVMs, drives the API, and asserts behavior across kernels and configurations, including performance and regression suites.

---

## 9. Navigation (subsystem documentation)

| Area | Document |
|------|----------|
| Core VMM (KVM, devices, virtio, snapshots, memory) | [src/vmm/README.md](src/vmm/README.md) |
| Virtual device layer (MMIO bus, virtio transports, backends) | [src/vmm/src/devices/README.md](src/vmm/src/devices/README.md) |
| MicroVM Metadata Service (MMDS) | [src/vmm/src/mmds/README.md](src/vmm/src/mmds/README.md) |
| Monitor binary, API server, seccomp wiring | [src/firecracker/README.md](src/firecracker/README.md) |
| Shared Rust utilities | [src/utils/README.md](src/utils/README.md) |
| Jailer (sandbox setup and exec handoff) | [src/jailer/README.md](src/jailer/README.md) |
| Seccomp policy compiler | [src/seccompiler/README.md](src/seccompiler/README.md) |
| CPU template helper CLI | [src/cpu-template-helper/README.md](src/cpu-template-helper/README.md) |
| ACPI table generation | [src/acpi-tables/README.md](src/acpi-tables/README.md) |
| Snapshot file editor | [src/snapshot-editor/README.md](src/snapshot-editor/README.md) |
| Snapshot rebase utility | [src/rebase-snap/README.md](src/rebase-snap/README.md) |
| Clippy tracing lint | [src/clippy-tracing/README.md](src/clippy-tracing/README.md) |
| Optional tracing instrumentation | [src/log-instrument/README.md](src/log-instrument/README.md) |
| Proc-macros for tracing instrumentation | [src/log-instrument-macros/README.md](src/log-instrument-macros/README.md) |
| Python integration tests (overview) | [tests/README.md](tests/README.md) |
| Python test framework (fixtures, microVM lifecycle) | [tests/framework/README.md](tests/framework/README.md) |
| Developer tooling (devcontainer, CI helpers, scripts) | [tools/README.md](tools/README.md) |
| Bundled policies, guest configs, overlays | [resources/README.md](resources/README.md) |
| Recursive documentation process for agents | [DIRECTIONS.md](DIRECTIONS.md) |

Human-facing guides (getting started, networking, jailer, snapshots) remain under [docs/](docs/).
