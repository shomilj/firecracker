# Python integration test framework

## 1. Purpose and scope

1.1. The Python integration test framework is the programmatic layer that drives Firecracker end-to-end in automated tests: it launches the VMM under the jailer, configures guests through the HTTP control plane, attaches storage and networking, exercises lifecycle operations (boot, pause, snapshot, restore), and tears everything down deterministically. It is not a standalone product; it is the glue between the test runner, host tooling (network namespaces, SSH, process control), and the Firecracker binaries.

1.2. Design goals include: strict validation of API outcomes (failures surface as test failures with rich context), reproducible isolation per VM (separate network namespaces and chroots), minimal surprise around asynchronous startup (polling and retries at well-defined boundaries), and optional observability hooks (logs, metrics, memory sampling, latency checks) that can be turned off when they would add noise—for example under parallel test workers where timing assertions become meaningless.

---

## 2. Pytest integration surface (what the framework assumes)

2.1. **Runner-first contract.** The framework is consumed almost exclusively through pytest fixtures defined at the test-tree root: a session-scoped temporary directory, a session-scoped network-namespace factory, and a function-scoped factory object that yields microVM handles. Tests rarely import core types directly except in specialized subtrees; instead they depend on parametrized fixtures for kernels, root filesystems, CPU templates, and boot-vs-restore constructors.

2.2. **Property recording.** The runner’s property-recording hook is wrapped so that key-value pairs propagate both into machine-readable reports and into the embedded metrics logger. Custom templates, selected guest kernel stems, and binary identities thus appear consistently in downstream dashboards without each test manually duplicating calls.

2.3. **Failure-aware teardown.** The factory’s finalizer consults phase reports: only when the *call* phase failed does it attempt an expanded diagnostic capture (metrics flush, host ring-buffer text, chroot file copies, console buffers). Successful tests still run full VM teardown but skip expensive preservation work.

2.4. **Command-line extensions.** The framework cooperates with extra options for overriding binary directories and injecting a custom CPU template file for entire sessions. Those options are parsed once per session and flow into factory construction so individual tests do not reimplement policy.

2.5. **Interaction with distributed execution.** When multiple workers run in parallel, each worker owns its own session directory, its own namespace pool, and its own compiled helper binaries. Locks around shared filesystem mutations (for example cloning reference trees or hardlinking release artifacts) serialize only the critical sections that would otherwise race across workers.

2.6. **Diagram: runner hooks and framework objects.**

```
  pytest session start
         │
         ├─► session temp root (unique per session)
         ├─► compile helper C binaries into session root
         └─► netns factory (per worker id)
                    │
                    v
         per-test: factory + results_dir + metrics
                    │
                    v
              microVM handle(s)
                    │
                    v
         teardown: phase report → conditional artifact capture → kill sweep
```

---

## 3. Layered architecture inside the library

3.1. **Factory tier.** The highest-level entry point knows where release-quality or workspace-built binaries live and how to obtain a network namespace from the pool. It mints microVM handles, each with a unique identity, an associated jailer context (chroot layout, UID/GID, daemonize vs interactive mode, cgroup hooks), and a namespace for TAP devices and SSH to the guest.

3.2. **MicroVM tier.** Once spawned, each microVM exposes an HTTP API client bound to the Unix domain socket inside the jail. Resources are modeled REST-style: machine configuration, boot source, drives, network interfaces, VM actions, snapshots, optional entropy, balloon, vsock, metadata service, and more. The client uses HTTP over Unix sockets with a connection pool sized to stay just under the server’s concurrent-connection limit, avoiding a class of “server full” races when connections churn during bursty configuration.

3.3. **Helpers tier (orthogonal).** Interactive debugging support—SSH attachment, debugger hooks, serial consoles via terminal multiplexers, bridging guests to wider host networks—lives conceptually beside the core lifecycle object. These helpers mutate launch parameters (for example disabling daemonization) and favor developer workflows over CI determinism.

3.4. **Cross-cutting utilities.** Host command execution with consistent logging, CPU topology mapping for containerized environments, process lifetime helpers, small HTTP builders for guest metadata probes, optional tracing or iperf conveniences, and vhost-user backend orchestration each extend the core without bloating the central object.

---

## 4. Jailer integration and filesystem view

4.1. **Context record.** Each microVM’s jailer context captures everything needed to construct a correct command line: identity string, executable path, UID/GID, chroot base, optional network namespace path, whether to detach as a daemon, whether to create a new PID namespace, cgroup-related parameters, and an open-ended map of VMM-specific flags that the jailer passes through to the Firecracker executable after the double-hyphen separator.

4.2. **Chroot layout.** The layout follows the jailer’s contract: a per-VM directory under the chroot base, containing a root tree where linked artifacts live. The context exposes the full host path to the API socket (defaulting under the runtime directory inside the jail) and the path to the PID file written for the jailed Firecracker process.

4.3. **Resource publication.** Publishing block files or kernels into the jail uses hardlinking when source and jail share a device, or copying when they do not, optionally creating block special nodes, then changing ownership to the jail UID/GID. Returned paths are always absolute *inside* the jail (leading slash) so API payloads match what the VMM truly sees.

4.4. **Cgroup cleanup.** Teardown walks cgroup controllers when configured: waits until task lists are empty with bounded retries, then removes per-VM cgroup directories. This complements brute-force process termination by reducing leftover kernel state.

---

## 5. MicroVM lifecycle in depth

5.1. **Launch modes.** Two primary paths exist: daemonized jailer (Firecracker fully detached; tests interact only via API socket and log files) and terminal-multiplexer–wrapped jailer (stdio attached so serial console traffic lands in a scrollback file). Interaction between new PID namespaces and the multiplexer path matters: the supervised process may exit early while Firecracker reparents, so the framework clears stale supervisor PIDs to avoid signaling the wrong process during teardown.

5.2. **Readiness layering.** Readiness is not monolithic. If a JSON config file is injected and at least one network interface exists, waiting for guest SSH is the strongest gate—userspace and networking are far enough along for most workloads. If the API is enabled without that combination, the framework may wait for log lines proving the API socket is live, or poll for the socket file with backoff. Quiet log settings push responsibility to the caller for synchronization. Optional NUMA binding is implemented by prepending policy wrappers ahead of the jailer so the entire stack inherits affinity without embedding policy inside HTTP payloads.

5.3. **Configuration sequence.** Typical setup applies machine sizing (vCPU count, SMT flag, memory, optional dirty tracking for differential snapshots, huge-page policy), records memory size for later metric labels, optionally starts memory sampling, then configures boot source (kernel, optional initrd, command line). Root filesystem attachment infers read-only mode from squashfs vs writable ext4. CPU templates may be static names or structured JSON posted to the CPU configuration endpoint; names normalize for telemetry.

5.4. **Block and backend lifetimes.** File-backed drives link into the jail. Vhost-user drives spawn an external backend, wait for its socket, change ownership on the socket into the jail, then register the drive. Replacing a vhost-user identifier kills the previous backend first to avoid orphaned host processes—a subtle ordering rule that prevents teardown races.

5.5. **Networking.** Each TAP is created inside the microVM’s network namespace with host and guest addresses derived from a structured interface object. The framework retains both high-level config and TAP handles for snapshot restore and SSH. SSH uses a persistent connection bound to the namespace, private keys copied beside the root filesystem, and a control master socket colocated with the chroot; connections cache per interface index and must close before Firecracker exits cleanly.

5.6. **Snapshots: kinds and files.** Snapshots bundle vmstate, memory images, disk id-to-path maps, network interface records, SSH key material, snapshot kind (full vs differential variants), and free-form metadata (kernel path, vCPU count). Full snapshots are self-contained. Differential kinds require dirty tracking to have been enabled before capture; some variants require rebasing incremental files onto a base using either a dedicated rebase binary or an editor tool. One differential variant aligns with mincore-oriented hypervisor behavior; local enums distinguish these cases so tests cannot accidentally mix semantics.

5.7. **Snapshot create and restore.** Creation pauses the VM first, then calls the snapshot endpoint with paths relative to the jail. Serialization to a directory copies or hardlinks components and writes a JSON manifest. Restore copies files into a fresh jail with distinct basenames to avoid clobbering golden inputs, recreates TAP devices from saved objects, hardlinks disks, then posts a load request. Memory backends may be plain files or userfaultfd sockets; the latter publishes an external page-fault handler into the jail, executes it with dropped privileges matching the jail user, and wires its socket as the memory backend. Optional network override maps rename host-side TAPs for compatibility across baseline vs candidate binaries in comparative runs.

5.8. **Userfaultfd path.** The handler runs jailed, logs to a dedicated file, ensures executable permissions inside the chroot, and listens on a fixed socket path. Firecracker connects as a client; ownership must match the jail UID/GID after creation. Startup failure surfaces log contents immediately because silent hangs here obscure root causes (bad paths, seccomp denials).

5.9. **Multi-VM and chain scenarios.** The factory remembers every VM for a coordinated kill: stop monitors, close SSH, terminate vhost-user backends, signal Firecracker and any multiplexer supervisor, wait on modern wait primitives, optionally assert no stray identity processes remain, terminate userfaultfd helpers, optionally validate API latency from logs, remove chroots when safe. A generator pattern can restore many VMs from one snapshot or walk incremental chains: yield a running VM to the test, optionally capture the next layer, delete superseded files to save disk, tear down, repeat—supporting stress without manual bookkeeping.

5.10. **Serial and interactive testing.** When not daemonized, Firecracker output goes to a multiplexer log file. A serial helper polls that file, injects keystrokes via the multiplexer’s input command, and reads with deadlines that fail the test and kill the VM on excessive delay. A small state machine matches incremental input against target strings for firmware menus without fragile full-string compares on bursty I/O.

5.11. **Lifecycle swimlane (conceptual).**

```
  test code                framework                     host / guest
      │                         │                            │
      ├─ build VM handle ───────►                            │
      ├─ configure API ─────────► configuration requests ───► VMM applies
      ├─ start ─────────────────► explicit start action ─────► boot
      │                         ├─ wait SSH / socket / log ──► userspace ready
      ├─ workload / assert ─────► SSH / vsock / virtio ─────► guest
      ├─ snapshot? ─────────────► pause + create ────────────► disk files
      ├─ restore? ──────────────► new process + load ───────► resume
      └─ implicit teardown ─────► ordered kill + cgroup/ns ─► clean host
```

---

## 6. Observability, guardrails, and failure modes

6.1. **Logging and API latency.** Verbose logging enables parsing of per-request durations from the log stream; the framework can assert ceilings on API handling time for synchronous operations, with long operations like snapshots exempt. When parallel workers contend for CPUs, these checks disable by default because wall-time noise dominates signal.

6.2. **Metrics files.** Optional newline-delimited JSON metrics can be linked into the jail and flushed via API; readers tolerate partial trailing writes from crash mid-flush.

6.3. **Memory monitors.** Optional sampling validates memory behavior on shutdown—useful for ballooning or leak regressions.

6.4. **Debug dumps.** Low-level HTTP exceptions route through an error callback tied to microVM diagnostics. On failure, the framework may collect Firecracker logs, userfaultfd logs, and thread stack traces scraped from the kernel’s thread listings for every VMM thread—unless the VM was already known dead. Transport failures thus surface alongside guest SSH failures in a unified troubleshooting story.

6.5. **Invariant-style failure categories.**

6.5.1. **Misconfiguration before start** — invalid API payloads raise immediately; there is no silent fallback.

6.5.2. **Hung readiness** — outer test timeouts fire; inner polls have bounded retries but cannot escape a wedged hypervisor.

6.5.3. **Partial teardown** — kill sweeps are ordered, yet kernel or device bugs can still leave namespaces or files behind; best-effort cleanup logs are not always actionable.

6.5.4. **Comparative tests** — git-based A/B clones a reference revision to a temporary workspace using a branch trick so arbitrary commitishes materialize without mutating the developer tree; lock contention or disk exhaustion surfaces as test failures distinct from product bugs.

---

## 7. Artifacts, discovery, and versioning

7.1. **Kernel and disk discovery.** Guest kernels and root filesystems resolve from a machine-specific artifact root with a fallback when shared storage is absent. Regex allowlists keep special kernels (for example ACPI-less variants) opt-in. Parametrized fixtures record normalized stem properties for reporting dimensions.

7.2. **Release binaries as parameters.** Shipped Firecracker builds can be discovered, version-sorted, capped relative to the workspace’s next minor boundary, and fed as pytest parameters alongside the workspace-built tip linked to mirror release naming—so performance suites treat local builds like another release channel.

7.3. **Snapshot format version.** When snapshot formats decouple from semantic versioning, the binary exposes a dedicated version query; tests gate behavior on that string when comparing across baselines.

7.4. **Static reference data.** JSON payloads for invalid metadata, CPU template fingerprints per host flavor, custom template documents for multiple architectures, and CSV register enumerations live alongside tests as versioned inputs—not as code—to keep assertions data-driven.

---

## 8. Host assumptions the framework makes

8.1. **Privilege and devices.** Root-equivalent capability, accessible KVM, and namespace/cgroup operations are assumed for normal integration tests. Seccomp demonstration builds invoke the Rust workspace to produce example binaries; that assumes a working toolchain layout.

8.2. **Container vs bare metal.** CPU maps translate logical indices to cgroup-visible CPUs because container views of processor lists can diverge from hardware reality; affinity helpers depend on that mapping for reproducible pinning experiments.

8.3. **Network reachability for optional probes.** Cloud metadata endpoints may be absent; code paths guard probes so local laptops do not fail purely for lacking a cloud environment.

---

## 9. Performance-oriented features vs functional defaults

9.1. **Thread and CPU controls.** For throughput or latency experiments, helpers enumerate threads by name, map them through the cgroup-aware CPU map, and pin VMM, API, and vCPU threads to disjoint cores—returning free cores for paired host workloads. This machinery exists to stabilize performance tests; functional tests often ignore pinning unless the behavior under test is scheduling-sensitive.

9.2. **Metrics emission without assertions.** Performance modules frequently record series into the embedded metrics sink with dimensions that include guest kernel stem, rootfs label, vCPU count, and memory size—copied from a process-wide host properties singleton plus per-VM configuration. Functional modules use the same sink for pass/fail counts but rely on explicit assertions for correctness.

9.3. **Statistical helpers.** The framework exposes permutation-style hypothesis testing utilities so callers can compare two samples (for example baseline vs candidate throughput) without embedding fixed p-value thresholds in every test file.

9.4. **A/B infrastructure.** Git-based A/B runs a callable twice across revisions and compares with pluggable equality or subset constraints. Binary-directory A/B skips git and compares runs against different prebuilt artifact directories—ideal for snapshot compatibility matrices. Host-command A/B tightens stdout/stderr equality in pull-request CI while relaxing outside CI for faster local iteration.

9.5. **Separation summary.**

```
  Concern              │ Functional emphasis          │ Performance emphasis
  ─────────────────────┼──────────────────────────────┼────────────────────────
  Primary signal       │ Asserted state / output      │ Measured distributions
  Typical fixture      │ Broad kernel/template matrix │ Release binary params,
                       │                              │ pinned CPUs, warmups
  Runner sensitivity   │ Tolerates parallelism        │ May disable latency
                       │                              │ parsing when distributed
```

---

## 10. HTTP control plane client (behavioral summary)

10.1. **Strict errors.** Resources know how to issue reads and writes; non-empty error responses deserialize JSON bodies and raise with fault text when present—tests fail loudly on unexpected API errors instead of silently proceeding.

10.2. **Connection pool rationale.** When the pool evicts connections, close and open events can reorder at the kernel level; keeping headroom below the advertised server limit avoids spurious “server full” conditions during bursty test traffic.

10.3. **Error callback.** Transport-level failures tie into the same diagnostic path as SSH failures so operators see a single coherent bundle.

---

## 11. Static analysis and heavy tools

11.1. **Seccomp introspection.** A separate module disassembles binaries, tracks syscall numbers through simplified register reasoning, and compares against installed policies to flag redundant rules. It is intentionally isolated from the hot path of booting guests.

---

## 12. Behavioral invariants worth relying on

12.1. API calls that should succeed are checked strictly; there is no silent fallback when the control plane errors.

12.2. Snapshot operations that need the VM paused perform the pause explicitly—tests do not depend on implicit pausing side effects.

12.3. Differential snapshot types that need rebasing encode that requirement in the type system so misuse raises before touching disks.

12.4. Kill ordering prefers stopping external backends before Firecracker when both exist, avoiding socket races during teardown.

12.5. When userfaultfd is enabled, memory backends switch from file paths to the userfaultfd socket transparently to the rest of restore logic—only the backend descriptor changes.

---

## 13. Extensibility model

13.1. New device types generally arrive as additional API resources plus optional host-side helpers (for example a new virtio backend process). The core microVM object should only gain thin forwarding logic; heavier orchestration belongs in focused utility modules.

13.2. New CI dimensions should extend the global host properties singleton cautiously—everything constructed there runs once per process and must remain side-effect free beyond probing.

---

## 14. End-to-end data flow (typical test)

```
  +------------------+       build / select       +------------------+
  | pytest parameter | ---- binaries & rootfs ---> |     Factory      |
  +------------------+                            +--------+---------+
                                                           |
                    +----------------------------------------+
                    v
           +------------------+     jailer + netns     +------------------+
           |  MicroVM handle  | ---- spawn ---------> | Firecracker proc |
           +--------+---------+                       +--------+---------+
                    |                                          |
        configure   |  HTTP/UDS                                | vCPUs
        via API     v                                          v
           +------------------+                         +------------------+
           | Guest (SSH/serial)| <------ TAP -------- |     devices      |
           +------------------+                         +------------------+

   teardown: stop monitors -> close SSH -> kill vhost-user -> fatal signal FC
             -> wait on waitable handles -> verify no stray PIDs -> optional latency parse
             -> cleanup cgroups/netns/chroot
```
