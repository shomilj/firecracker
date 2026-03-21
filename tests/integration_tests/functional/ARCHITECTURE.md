# Functional Integration Test Suite — Architecture

## Purpose & Boundaries

This directory holds the primary end-to-end integration tests for the microvirtual machine runtime. The suite’s responsibility is to exercise the product as a complete system: a host process that exposes a configuration surface, boots a Linux guest inside a constrained execution environment, and connects that guest to virtual devices and host resources. Tests assert observable behavior across the boundary between operator intent, hypervisor implementation, and guest-visible semantics. They answer whether configuration changes are accepted or rejected at the right lifecycle stages, whether virtual devices honor protocol and ordering guarantees, whether snapshots preserve and restore guest state in ways that remain consistent with security and compatibility rules, and whether operational surfaces such as metrics and logging remain stable enough that downstream automation will not break silently.

What this suite is not responsible for is equally important. It does not replace unit tests inside the core implementation; it does not duplicate the dedicated security-focused test track that probes hardening assumptions in isolation; and it is not a performance benchmark suite, though some scenarios apply sustained I/O or network load to flush out stability bugs. Documentation of internal algorithms, data structures, or module boundaries belongs elsewhere. If this suite were removed or left unmaintained, regressions would surface late: broken API contracts, incompatible snapshot formats, guest-visible CPU identity drift, subtle virtio or networking failures, and silent changes to metrics schemas would reach users without a cohesive system-level safety net.

The following diagram situates the suite within the larger validation landscape. The tests themselves sit above the product binaries and host tooling; they drive configuration and observe outcomes through the same channels an operator or orchestrator would use.

```
                    +-----------------------------+
                    |   Human / CI orchestrator   |
                    +-------------+---------------+
                                  |
                                  v
                    +-----------------------------+
                    |  Functional integration     |
                    |  tests (this suite)         |
                    +-------------+---------------+
                                  |
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
    +---------------+    +---------------+    +---------------+
    | Configuration |    | Host helpers  |    | Guest actions |
    | & lifecycle   |    | (network,     |    | (remote exec, |
    | via HTTP API  |    | disk build,   |    | data paths)   |
    +-------+-------+    +-------+-------+    +-------+-------+
            |                    |                    |
            +--------------------+--------------------+
                                 |
                                 v
                    +-----------------------------+
                    |  MicroVM runtime + jailer   |
                    +-----------------------------+
                                 |
                                 v
                    +-----------------------------+
                    |  Linux guest kernel +     |
                    |  userland (root filesystem) |
                    +-----------------------------+
```

The suite assumes a Linux host with appropriate virtualization facilities, elevated privileges for network namespace and device management, and prepared guest artifacts. Those prerequisites are not validated here in depth; they are part of the environment contract implied by running the tests successfully.

## Interfaces & Contracts

The tests consume several external interfaces and in turn impose expectations on callers of the overall system.

From the host environment, the suite requires superuser capability, isolated network namespaces, ability to create tap interfaces, and the ability to launch and signal processes under test. A session-scoped workspace is established so concurrent test runs do not collide on temporary paths. The harness can build ancillary host binaries used only for probing guest connectivity or device behavior, which keeps guest-side checks small and deterministic.

From the product artifacts, the suite expects a pair of executables: the main virtual machine monitor and the companion jail helper that applies namespace and filesystem confinement. Tests may iterate across multiple published build variants to ensure compatibility matrices stay honest. An optional override allows injecting pre-built binaries instead of compiling from source, which matters for release validation and for reproducing issues against known bits.

From guest images, tests rely on kernels and root filesystems exposed through shared fixtures. Different scenarios pin different kernel generations where behavior is version-sensitive, for example when validating timekeeping after resume or when exercising snapshot restore paths that depend on guest features. The guest must expose a remote execution channel so tests can run commands, inspect kernel logs, and generate workload inside the virtual machine without embedding all logic in the host harness.

The outward-facing contract of this suite to the wider organization is primarily pass or fail signals augmented with rich properties: host metadata, artifact identity, and timing are attached to results so dashboards can correlate failures with machine types and kernel versions. Some scenarios additionally record size metrics for shipped binaries, turning the same pipeline into a lightweight guardrail against accidental bloat.

Callers of the system under test—meaning these tests acting as orchestrators—must maintain invariants such as configuring network attachments before boot when using static configuration files, providing valid paths inside the jail, and sequencing snapshot operations only when the virtual machine is in a compatible state. The tests document those invariants implicitly by pairing happy paths with negative cases that expect specific error messages when rules are violated.

## Data Flow

Configuration data enters the system as structured messages sent to the HTTP surface before and after boot, as command-line bootstrapping parameters when the process starts without a live socket, and as files placed into the jail that represent disks and kernel images. Network definitions bind host tap devices to guest-facing MAC addresses. Block definitions bind files or socket-backed backends to guest disk slots with explicit ordering. Machine-wide parameters carry processor count, memory size, optional tracking of modified guest pages for incremental snapshots, and CPU personality templates on architectures where the host exposes tunable models.

During execution, traffic flows across virtio network and block devices, vsock channels for stream semantics beyond TCP inside the guest, and optional entropy devices. The metadata service path accepts published JSON documents from the host side and serves them to the guest over a link-local address, with versioned authentication behavior. Metrics data flows out as newline-delimited JSON records reflecting counters and aggregates for devices and signals. Human-oriented logs land in configured sinks with predictable formatting when options for verbosity and origin tracing are enabled.

The next diagram summarizes the main data paths the suite touches during a typical scenario that boots a guest, runs workload, and captures telemetry.

```
  Host JSON / CLI          Guest workload           Artifacts on disk
        |                        |                         |
        v                        v                         v
   +----------+           +-----------+            +------------+
   | HTTP API | --------> | virtio /  | ---------> | snapshots  |
   | requests |           | vsock /   |            | (memory +  |
   +----------+           | MMDS      |            |  vmstate)  |
        |                 +-----------+                 |
        v                        |                      v
   +----------+                  |               +------------+
   | logs &   | <----------------+----------------| metrics    |
   | metrics  |   host observes guest I/O       | JSON lines |
   +----------+                                 +------------+
```

For snapshot-heavy scenarios, memory and machine state files are written atomically in cooperating pairs, sometimes layered as full versus differential forms. Incremental snapshots may depend on dirty page tracking being enabled ahead of time; the suite verifies that turning those features on and off aligns with the snapshot mode under test. Restoration can target eager loading of memory or deferred population through a userfault-style backend, which introduces another data path between a handler process and the restoring monitor.

## Control Flow

Execution is driven by the test runner collecting cases, wiring shared fixtures, and invoking each scenario in isolation. A critical path common to many tests begins with constructing a fresh virtual machine handle, spawning the host process with jail parameters, applying baseline configuration, attaching devices, and issuing the start transition. After boot, tests branch into domain-specific actions: sending API mutations that should be rejected after boot, driving block rescan and resize behavior, flooding the network interface, or initiating save and restore cycles.

Branching is explicit around lifecycle gates. Some operations are only valid before the guest runs; others only after. The suite encodes those expectations as paired tests that succeed when updates are allowed and raise structured failures when they are not. Signal injection tests branch on whether the process should terminate or continue after a particular signal, reflecting different operational policies. Snapshot tests branch on whether restoration should immediately run the guest or remain paused until an explicit resume, validating both orchestration styles.

The control-flow diagram below abstracts the lifecycle states the suite repeatedly validates.

```
                    [spawn process]
                           |
                           v
                    [configure resources]
                           |
              +------------+------------+
              |                         |
              v                         v
    [pre-boot API mutations]    [start guest]
              |                         |
              |                         v
              |                 [running workload]
              |                         |
              |              +----------+----------+
              |              |                     |
              v              v                     v
    [expect errors if        [pause / resume]   [shutdown / signals]
     invalid]                     |                     |
              |                     v                     v
              |              [inspect state]      [metrics flush /
              |                                    log messages]
              v
    [boot when valid]
```

Concurrency-oriented cases fan out parallel launches to stress independent instances, disabling certain timing instrumentation that becomes unreliable under parallel API load. That is a control-flow compromise: the suite prefers breadth over micro-timing accuracy in those scenarios.

## State & Lifecycle

The suite owns no long-lived production state; its state is ephemeral per test and per session. Each scenario typically constructs an isolated instance directory, populates disk images as needed, and tears down processes afterward. Snapshot artifacts may persist across steps within a single test when exercising chained restore operations, and helper routines copy or hardlink large files between host and jail paths to mimic realistic operator workflows.

Virtual machine lifecycle phases—unconfigured, configured but halted, running, paused, snapshotting—are exercised systematically. Balloon device scenarios track host resident set size and guest free memory over inflation and deflation cycles, including deliberate memory pressure. Network and block device metrics scenarios validate aggregation keys across multiple attached devices of the same class, which implies steady-state operation long enough for counters to reflect activity.

Startup behavior includes cold boot from API configuration, cold boot from a JSON configuration file passed at process launch, and combinations with optional metadata files that seed guest-visible configuration. Shutdown paths include guest-initiated reboot expectations, signal-driven termination, and flush operations that force metrics to disk while the guest still runs.

Recovery semantics receive heavy emphasis: restoring from a snapshot should resurrect virtio and vsock usability, reset certain security-sensitive tokens while preserving interface configuration, and in some cases require regenerating session secrets before metadata access succeeds again. The suite checks kernel log signatures after resume where virtual machine generation identifiers affect randomness reseeding, connecting low-level guest behavior to the snapshot contract.

## Failure Modes

Failures surface as assertion errors, timeouts, or mismatched string patterns in logs and API responses. The suite distinguishes expected failures—where an invalid operation must be rejected with a specific message—from defects where a valid operation silently misbehaves. Negative testing covers permission problems on snapshot files, invalid socket paths for deferred memory backing, malformed metadata, oversubscribed device counts, and API calls issued in wrong order relative to snapshot load.

Some failure modes are environmental rather than product bugs: tests may skip or mark non-default continuous integration when they rely on large artifact directories or extended runtime budgets. Intermittent failures have been observed around high-churn socket and remote execution paths; the suite mitigates race windows with deliberate delays in specific long sequences, acknowledging that flakiness is a failure mode for automation even when the product is correct.

Silent corruption is guarded by filesystem checks inside the guest after snapshot rounds, cryptographic or hash comparisons on data moved through vsock, and validation that aggregate metrics still relate to per-device metrics numerically. Where silent divergence could occur—such as token reuse across incompatible metadata protocol versions—the suite asserts explicit error strings rather than accepting ambiguous success.

## Operational Characteristics

Resource consumption is dominated by launching many full virtual machines, copying multi-hundred-megabyte memory images for snapshots, and occasionally driving sustained UDP ingress or parallel instances. Disk tests create scratch images and may truncate or resize backing files while the guest retains open handles. Network tests manipulate queue lengths to increase pressure on tap buffers.

Scaling bottlenecks appear at the host level: CPU for parallel instances, disk bandwidth for snapshot churn, and kernel networking for overload scenarios. Observability within tests relies on structured metrics lines, log scraping, and remote command output. External telemetry hooks push per-test durations and outcomes to a metrics backend on supported runners, annotating results with host characteristics so regressions can be triaged without reproducing the entire matrix locally.

The suite is not designed for frequent interactive runs on laptops; it expects a server-class Linux host aligned with the project’s CI environment. Build steps for helper tools add upfront cost per session but amortize across many cases.

## Design Rationale

Black-box integration testing trades execution cost for confidence. Exercising the HTTP surface ensures that the contract operators depend on remains stable even when internals refactor. Combining API tests with guest-side validation catches mismatches between what the host believes is configured and what the guest actually sees in CPU identity, topology, and device enumeration.

Snapshot tests exist because serialization formats and restore paths are high-risk change areas: a subtle incompatibility might only appear when chaining restores or when mixing full and differential images. Cross-version restore scenarios using archived artifacts extend that guarantee beyond what a single build can simulate, at the expense of heavier storage and maintenance.

The separation between this functional track and narrower tests reflects a classic tradeoff: unit tests localize faults but cannot prove orchestration across jail boundaries, networking namespaces, and real kernels. This suite exists to provide that proof, accepting longer runtimes and flakier environmental dependencies as the price of realism.

The following state-machine sketch highlights why snapshot-related cases are structured as multi-phase stories rather than single assertions. Each transition corresponds to explicit operator actions the suite sequences.

```
        +-------------+
        | fresh VM    |
        +------+------+
               | configure + start
               v
        +-------------+       save snapshot
        | running     |------------------+
        +------+------+                  |
               |                         v
               |                  +-------------+
               |                  | snapshot    |
               |                  | files       |
               |                  +------+------+
               |                         | restore
               |                         v
               |                  +-------------+
               +----------------->| restored VM |
                                  +------+------+
                                         | resume / verify
                                         v
                                  +-------------+
                                  | steady or   |
                                  | shutdown    |
                                  +-------------+
```

Taken together, the behaviors validated here form a systems perspective on correctness: the product must configure predictably, run reliably under stress, explain failures legibly, emit telemetry that automation can parse, and preserve guest-visible semantics across upgrades and snapshot operations. This suite is the primary automated guardian of that holistic promise.
