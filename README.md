<picture>
   <source media="(prefers-color-scheme: dark)" srcset="docs/images/fc_logo_full_transparent-bg_white-fg.png">
   <source media="(prefers-color-scheme: light)" srcset="docs/images/fc_logo_full_transparent-bg.png">
   <img alt="Firecracker Logo Title" width="750" src="docs/images/fc_logo_full_transparent-bg.png">
</picture>

Mission and product context: [CHARTER.md](CHARTER.md), [FAQ.md](FAQ.md), [docs/design.md](docs/design.md).

The sections below are a **systems architecture** view of this repository and point to deeper per-tree documentation.

---

## 1. Problem shape and positioning

1.1. **Isolation model.** Guests run with hardware virtualization (KVM). The VMM does not aim to emulate a full PC; it exposes a deliberately minimal machine: virtio block and network, vsock, optional entropy, serial where policy allows, and (on x86) a narrow set of legacy devices needed for boot and reset semantics. Fewer emulated surfaces reduce memory footprint, startup time, and attack surface.

1.2. **Operational model.** An operator (or orchestrator) starts the monitor, then drives **configuration through a control-plane API** over a Unix-domain socket. The fast path is guest execution on vCPU threads; the API thread is explicitly off the emulation hot path. Production deployments typically wrap process startup in a companion **jailer** that creates namespaces, cgroups, chroot, and privilege boundaries before executing the monitor as an unprivileged principal.

1.3. **Threat stance.** Untrusted code is assumed to run inside the guest. vCPU threads are treated as hostile once the guest runs. Data leaving the guest toward the host (network frames, block I/O, vsock bytes) crosses **trust boundaries** where copying, policy, and rate limiting apply. Seccomp **BPF** filters constrain which host syscalls each thread may invoke, tightened further after initialization so the attack surface shrinks over the lifecycle.

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

---

## 3. Control plane and configuration lifecycle

3.1. **Pre-boot configuration.** Before the guest runs, the API accepts a schema-driven description: machine topology (vCPUs, RAM), kernel/initrd or boot ELF, block devices and their host backing, network interfaces and tap attachments, vsock, entropy device, balloon, logging and metrics sinks, MMDS JSON, rate limiters, and CPU template selections. Validation ensures coherent combinations (device IDs, capacity limits).

3.2. **Boot and run.** Starting the instance allocates guest memory maps, places the kernel per the chosen loader path, sets CPUID or device-tree exposure according to templates, and enters the KVM run state. From then on, most interaction is **exit-driven**: I/O instructions and virtio virtqueues generate work on the VMM thread and related backends.

3.3. **Runtime operations.** Pause/resume, device hotplug semantics where supported, drive rescan and backing swap, balloon inflation/deflation, and signal-driven shutdown integrate with the event loop. Metrics and logs flush through configured sinks (often FIFOs for log aggregation).

3.4. **Snapshots.** Memory and device state serialize to a versioned format with explicit compatibility rules. Restore reconstructs KVM objects and replays state so guests can resume; integrity checks guard against tampering depending on options.

---

## 4. Device and I/O architecture

4.1. **Virtio-first.** Network and block use virtio with Firecracker’s own device models and queue handling. Rate limiting applies at the boundary where host resources are consumed (token-bucket style controls on bandwidth and ops/sec), so fairness is enforced in the VMM rather than inside the guest.

4.2. **Vsock.** A virtio-vsock path bridges guest AF_VSOCK to host Unix sockets, enabling agent protocols without exposing new host attack surface beyond the configured socket endpoints.

4.3. **MMDS.** A minimal HTTP-oriented metadata tree can be exposed to the guest over a dedicated virtio-net channel (not general Internet access). Content is entirely operator-defined JSON; the feature is optional and gated by configuration.

4.4. **Entropy.** A virtio-rng style path can inject host-derived entropy into the guest where supported, subject to platform policies.

4.5. **Balloon.** Virtio balloon allows the host to reclaim guest-physical pages cooperatively, with careful synchronization so guest visibility and VMM bookkeeping stay consistent.

---

## 5. Memory and CPU presentation

5.1. **Guest RAM.** RAM is mapped for the guest with architecture-specific alignment and dirty tracking for migration/snapshot. Userfaultfd-based mechanisms can participate in demand paging or post-copy style experiments in supported configurations.

5.2. **CPU templates.** Templates adjust visible CPUID (x86) or device-tree CPU features (ARM) so guests see a stable, supportable feature set across heterogeneous hosts. Custom templates can be synthesized from baseline hardware fingerprints using helper tooling.

5.3. **ACPI.** Minimal ACPI artifacts may be generated to satisfy guest kernels’ expectations without turning Firecracker into a full PC emulator.

---

## 6. Security architecture

6.1. **Seccomp stages.** Syscall filtering is applied per-thread with different profiles for API, VMM, and vCPU contexts. The compiler toolchain turns JSON policy into BPF programs embedded at build time; runtime installation happens before guest code executes on vCPUs.

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
