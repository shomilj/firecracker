# Python Integration Test Framework Architecture

This document describes the architecture of the Python layer that powers Firecracker integration tests: how it models virtual machines, how it separates host-side orchestration from guest-side behavior, how shared dependency-injection hooks wire tests into that machinery, and how tests exercise the product through stable external interfaces. It is about the test system, not about Firecracker’s internal VMM design.

---

## Purpose & Boundaries

**Responsibility.** The framework’s job is to make it practical to run repeatable, automated integration tests against real Firecracker processes and real guest kernels. It provides abstractions for allocating an isolated execution environment on the host, launching the hypervisor under the same constraints production uses, configuring the machine through its public control plane, observing behavior from both host and guest, and tearing everything down cleanly. It also supplies supporting utilities: discovering and selecting build artifacts, capturing environment metadata for triage, coordinating concurrent test workers, and optional patterns for regression-style comparisons across revisions or binaries.

**Out of scope.** The framework does not implement the hypervisor, device models, or guest operating system behavior. It does not replace low-level unit tests in Rust or host-side tools that compile artifacts; it consumes those outputs. It is not a general-purpose orchestration system for arbitrary workloads—it is shaped around the lifecycle of a single microVM instance per test case unless a test explicitly composes more.

**Failure impact.** If this layer misbehaves, integration tests become flaky or misleading: false positives hide regressions; false negatives burn CI time and erode trust. Incorrect cleanup can leak network namespaces, stray processes, or disk under shared session-root paths, destabilizing shared runners. Conversely, when it works, tests validate end-to-end contracts that unit tests cannot cover—boot, control-plane semantics, snapshots, networking, and metrics—against binaries built from the same tree under test.

---

## Interfaces & Contracts

**Dependency injection layers.** The reusable wiring is split between this package, which implements factories and handles, and the top-level test configuration, which declares which kernels, root filesystems, CPU template variants, snapshot modes, and binary releases participate in parametrized runs. Session-wide hooks establish an isolated working root, compile tiny host-side helpers once, and provide a pool of network namespaces per worker so tests do not pay full setup cost every time. Function-scoped hooks build a factory bound to chosen binaries and namespace allocation, record which artifact paths were used, and on failure flush metrics and copy troubleshooting material from the jail. Optional hooks expose result directories for artifacts uploaded after the run. Together, these layers realize the “declare once, run many matrices” model integration suites need.

**What tests consume (conceptual surfaces).** Tests interact with a small number of conceptual surfaces. The most important is a *microvm handle*: an object that knows how to prepare a jail, start the supervisor and guest binary, expose a small request client bound to the control socket, attach block and network devices, trigger boot and lifecycle actions, take and restore snapshots, and shut down. Secondary surfaces include *artifact selectors* that yield kernel images and root filesystem blobs from a well-known artifact layout, *parameter bundles* for CPU templates and snapshot modes, and *helper bundles* for interactive debugging (for example, printing a secure-shell command line into the guest). Host networking is accessed through a *namespace factory* that hands out isolated network namespaces reused across a session to reduce churn.

**What the framework consumes from outside itself.** It expects to run with sufficient privilege to create network namespaces, taps, and iptables rules where tests require them—typical integration runs assume root. It expects prebuilt Firecracker and companion binaries (or a directory supplied at test invocation time), guest kernels, and rootfs images under configurable roots, with sensible fallbacks from shared CI storage to local build trees. It depends on standard host utilities where tests shell out for disk resize, tracing, or similar. Environment metadata (cloud instance identity when present, kernel version, CPU model) is gathered read-only for reporting.

**Contracts and invariants.** Callers must not assume a microvm is booted until they have completed configuration and issued an explicit start action; the framework separates “process up and control socket available” from “guest has userspace ready,” and offers waits keyed on logs or remote shell depending on configuration. Control-plane calls are expected to succeed with empty-body success responses for mutating operations; failures surface as structured errors. Session directories must remain unique when parallel schedulers run multiple test sessions; the session-scoped root directory hook establishes that. File-backed locks serialize certain artifact-mutating paths so parallel workers do not corrupt shared hardlinks.

The following diagram situates the framework between the test runner and the product’s external surfaces.

```
  +------------------+          +---------------------------+
  |  Test case       |  uses    |  Framework abstractions   |
  |  (test module)   +--------->|  factory, microvm handle, |
  +------------------+          |  artifacts, helpers       |
          |                     +------------+--------------+
          |                                  |
          | configures / asserts               | local socket requests,
          v                                  | signals, files
  +------------------+          +------------v--------------+
  |  Host OS         |  runs    |  Jailed Firecracker       |
  |  (netns, tap,    +--------->|  process + control thread |
  |   iptables)      |          +------------+--------------+
          ^                                  |
          | remote shell / guest sockets /     | paravirtual devices
          | metrics files                      v
  +-------+----------+          +--------------+-------------+
  |  Helper binaries |          |  Guest Linux + workloads    |
  |  (session-built) |          |  (shell, traffic tools)     |
  +------------------+          +----------------------------+
```

Before the diagram, the key idea is that tests never link against Firecracker as a library; they drive a live process through a narrow remote interface and observe side effects on disk and network. After the diagram, the split is intentional: host-side concerns stay in the test process and OS, while guest behavior stays inside the VM except where tests deliberately punch holes (port forwards, socket-based guest channels).

---

## Data Flow

**Inputs.** Data enters from four streams: command-line options (binary locations, optional CPU template files), the filesystem (kernels, rootfs, golden snapshots, serialized seccomp filter bundles used in ancillary checks), the environment (CI variables, parallelism hints), and runtime probes (host OS version, CPU vendor, optional cloud metadata). Parameterized matrix dimensions expand these into concrete paths and version tuples per test invocation.

**Transformations.** Artifact metadata turns file names into comparable version tuples and snapshot-format versions. Kernel and rootfs paths are copied or hardlinked into the jail so the hypervisor only sees confined paths. Network configuration objects become tap devices inside namespaces with IPv4 addressing chosen to avoid collisions. When snapshots are taken, memory and device state are written to files; when restored, those files are re-homed into a new jail and loaded through the snapshot endpoints. Metrics emitted by the product are newline-delimited structured records that tests parse after explicit flush actions.

**Outputs.** Tests read control-plane responses, log files, serial console captures, forwarded metrics, and guest command output (often through a remote shell). On failure, teardown logic may copy jail contents, host kernel ring buffer excerpts, and console text into per-test result directories for upload. CloudWatch-style metrics aggregate duration and outcome per test with rich dimensions.

```
  CLI + env ---------> session / worker identity
                              |
  artifact tree --------> kernel/rootfs paths -----> jail hardlinks
                              |
  factory ----------------> microvm handle -----> control writes (config)
                              |                      |
                              v                      v
  guest boot -----------> remote shell / logs / metrics <---- lifecycle actions
```

The prose before this flow: data is not “injected” as in a database; it is mostly files and network endpoints transformed into jail-relative paths and configuration payloads. The prose after: the valuable output for assertions is often indirect—through logs and guest-visible state—because the hypervisor’s contract is process-isolated and driven through its remote interface.

---

## Control Flow

**Triggers.** A collected test function starts when the runner schedules it. Automatically applied hooks attach metadata to each report. Session-scoped hooks create temporary roots and compile small C helpers once. Function-scoped hooks obtain a namespace from the pool, construct a factory with binary paths, and yield a builder. Tests or higher-level hooks then construct an instance with chosen kernel and rootfs, optionally start the supervisor process, apply a standard template of machine settings, add devices, and boot the guest. Specialized tests bypass shortcuts and issue fine-grained control-plane writes instead.

**Critical path.** The critical path for a typical boot test is: acquire namespace, allocate microvm id, prepare jail layout, launch jail wrapper (daemonized or under a terminal multiplexer for serial work), wait for control-socket readiness via log substring or socket existence, configure machine and drives, start instance, wait for guest readiness through the remote shell if networking was configured early. Snapshot tests branch: create snapshot, kill, build fresh handle from snapshot artifacts, restore, resume assertions.

**Branching.** Daemonize versus interactive mode chooses whether the supervisor runs detached or under a detachable console session. Dirty-page tracking and snapshot type select different preparatory flags. Userspace block backends spawn separate host processes that must be stopped before the main process in teardown order to avoid races. Optional monitors attach after spawn when metrics collection is requested.

A compact control-flow sketch:

```
  [test starts]
       |
       v
  [acquire net namespace] --> [build microvm handle]
       |                              |
       |                              v
       |                    [spawn: jail + process]
       |                              |
       |                              v
       |                    [wait: socket / logs / shell]
       |                              |
       v                              v
  [configure: machine, boot, drives, net]
       |
       v
  [start guest] --> [exercise & assert] --> [teardown: kill, unlock, cleanup]
```

Before: branching happens mostly at spawn and device attachment, not inside assertions. After: teardown is ordered so backends and monitors stop before signals, and so request-duration checks from logs run only when logging is verbose enough and parallelism does not distort timings.

---

## State & Lifecycle

**Owned state.** Each microvm handle tracks identifiers, paths to binaries, jail configuration, optional monitors, open remote-shell sessions, attached disk and network maps, and lifecycle flags. A factory tracks all handles it created so a single teardown can terminate leftovers. Network namespace objects know whether they are checked out from a pool. Snapshot bundles hold paths to saved machine-state and memory image files plus ancillary disk and interface metadata.

**Initialization.** Session initialization creates unique roots under a shared base. The factory picks binary directories from defaults or CLI overrides. Handles receive namespace references and optional NUMA pinning prefixes. Global metadata singletons initialize once per interpreter for host characteristics.

**Mutation.** Configuration mutates through the control socket until boot; after boot, some devices can be hot-patched within product rules. Logs and metrics files append on disk. Memory monitors sample if enabled.

**Cleanup.** Killing sends a non-catchable termination signal to the supervisor, closes remote shell sessions, stops monitors, tears down userspace block backends first, validates no stray processes share the jail identity, optionally validates control-plane latency histograms from logs, and clears spawned flags. Factory cleanup iterates all virtual machines. The namespace pool destroys pooled namespaces at session end. Failed tests trigger best-effort artifact harvesting without masking the original assertion failure.

**Recovery.** The framework does not implement automatic recovery from partial failure mid-test; it favors hard teardown and process isolation. Retry wrappers exist for short waits (socket presence, log substrings) to absorb scheduling jitter.

State machine helpers for serial-stream matching maintain incremental string state for console-driven tests; they are generic scaffolding unrelated to the VMM state machine but useful for staged boot scripts.

---

## Failure Modes

**Explicit handling.** Timeouts and missing log lines raise assertion errors. Control-plane errors become runtime errors with parsed fault bodies. Spawn and wait steps use bounded retries for benign races. Parallel test workers disable strict latency checks derived from logs to avoid false positives. Kill paths detect inconsistent process identifiers and dump debug context.

**Propagation.** Unexpected control-plane failures bubble up immediately. Guest remote-shell failures surface as command errors. File lock contention blocks briefly rather than corrupting.

**Silent risk areas.** If logging levels are too low, request-duration validation from logs is skipped, masking performance regressions. If cleanup fails partially, stray processes might remain until external janitors run. Incorrect assumptions about artifact naming could pair incompatible kernels and rootfs without failing until boot. A/B comparison utilities can hide absolute drift if both sides move together—by design for supply-chain alerts but requiring care for performance.

---

## Operational Characteristics

**Resources.** Each microvm uses CPU, memory, and disk proportional to guest configuration; host-side taps and namespaces add marginal overhead. Pooling namespaces amortizes creation cost. Session-scoped compilation avoids rebuilding tiny helpers per test.

**Bottlenecks.** Parallel worker processes multiply concurrent virtual machines; contention appears on disk IO, CPU for compilation locks, and kernel networking objects. Artifact hardlinking depends on same-device paths.

**Observability.** Tests record rich properties on reports (host and guest kernel strings, CPU model, optional CI identifiers). Embedded metrics log durations and failures with dimensional slicing. Optional monitors ingest the hypervisor’s own metrics stream. Logs are the primary semantic trace for control-plane and guest progress.

---

## Design Rationale

**Why a jail-first model.** Tests aim to mirror production invocation: the supervisor runs confined with a chroot and cgroup parameters the jail wrapper sets. That exercises real path handling, real privilege dropping, and real socket locations—properties a stub would miss.

**Why request-style traffic over a local socket.** The product’s supported automation surface is the documented remote interface; the framework mirrors what operators and control planes use, maximizing fidelity over in-process hooks.

**Why separate host helpers and guest workloads.** Keeping small compiled tools on the host (socket probes between host and guest, model-specific register readers, seccomp demonstrations) avoids bloating guest images while still testing cross-boundary behavior. Guest actions go through remote shell or guest-facing sockets to reflect realistic usage.

**Why dependency hooks live outside this package conceptually.** The directory under discussion supplies mechanisms; the shared top-level test configuration wires them into defaults so adding a new kernel matrix entry is a data concern, not a framework code change. That separation keeps reusable logic here and policy—which kernels to sweep—in collection hooks.

**Tradeoffs.** Root requirement improves fidelity but complicates local dev. Asserting latency from log lines couples tests to log format stability. Pooled namespaces trade a little complexity for throughput. Snapshot testing adds file management overhead but enables migration and resume coverage impossible with narrow in-process unit tests alone.

---

## Diagram: Host versus guest responsibilities

The next diagram clarifies the mental model of *host* versus *guest* in this framework—not as “Python versus Linux,” but as *orchestration plane* versus *workload under test*.

```
  HOST (test process + OS)                 GUEST (VM)
  =========================                =================

  * test runner drives lifecycle         * Linux kernel boots
  * creates netns / tap / routes         * userspace (ssh, systemd, etc.)
  * launches jail + hypervisor           * runs benchmarks or assertions
  * request client to control socket     * remote shell from host netns
  * reads host kernel ring buffer        * sees paravirtual devices only
  * may attach a debugger to host PID    * optional: kernel tracing, traffic tools

           ^                                        ^
           |  paravirt devices / network / block      |
           +----------------------------------------+
```

Before: the host owns all orchestration and observation of the hypervisor process; the guest owns what a real user VM would run. After: the only intentional cross-links are virtual devices, forwarded ports, and explicit remote commands—mirroring how operators debug production VMs.

---

## Closing synthesis

This framework is a thin but strict layer: it turns the runner’s collection and teardown semantics into reliable process isolation, ties tests to the same remote control surface as production tooling, and uses the host OS deliberately—namespaces, taps, optional forwarding—to make the guest reachable. Shared dependency hooks, declared in top-level test configuration, exist to expose that machinery with minimal boilerplate per test. Understanding the split between host orchestration, jail confinement, control-plane-driven configuration, and guest execution is the key to reading integration failures: first determine whether the anomaly lies in setup, control-plane behavior, or guest workload, then use the captured logs and artifacts from teardown to narrow further. That discipline is the architectural payoff of keeping the test system explicit, bounded, and aligned with how Firecracker is actually operated.
