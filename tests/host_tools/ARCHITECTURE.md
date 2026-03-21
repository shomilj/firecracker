# Auxiliary Host-Side Test Tools

This document describes the auxiliary tooling that runs on the Linux test host to support Firecracker integration and performance tests. The tooling is deliberately split between high-level orchestration written in Python and small, purpose-built native programs compiled on demand. Together they bridge the gap between the test harness, the hypervisor under test, the guest virtual machine, and external observability systems.

## Purpose & Boundaries

The responsibility of this subsystem is to provide reusable host-side capabilities that tests need but that do not belong inside the virtual machine or the main product binary. These capabilities include establishing and reusing secure shell sessions into guests, constructing isolated network namespaces and tap interfaces with deterministic addressing, creating disk images backed by files on real block storage, sampling process resource usage with guest memory excluded, validating and exporting structured metrics emitted by the virtual machine monitor, invoking the Rust build and auxiliary binaries with consistent flags and locking, exercising virtio and security-related behaviors from user space, and sending specially crafted network traffic that depends on host kernel socket options.

What this subsystem does not do is implement the hypervisor itself, define the guest operating system image, or replace the test framework’s fixtures and assertions. It does not own long-term storage of test artifacts beyond ephemeral files tied to object lifetimes in Python or the lifetime of a build directory. If this layer failed wholesale, integration tests that depend on host networking setup, guest reachability, disk preparation, metrics validation, or specialized syscall and memory exercises would fail to run or would produce false negatives. Tests that exercise only the API surface without booting a full microVM might still pass, but any scenario that assumes a working host-side environment would be blocked.

## Interfaces & Contracts

The Python-facing portion of the system exposes object-oriented helpers that tests import through the shared test package. Callers supply paths to keys and control sockets, network namespace identifiers, guest addressing information, and handles to running microVM abstractions. The networking helper establishes a multiplexed secure shell connection with strict key permissions, retries until the guest’s daemon is ready, and optionally wraps every subprocess invocation inside a network namespace command prefix. Tap interfaces are created inside namespaces with optional Internet Protocol configuration and queue length tuning. Immutable value objects describe host and guest addresses, prefix lengths, tap names, and derived link-layer addresses derived deterministically from guest addresses.

The memory and central processing unit monitoring components run as background threads. They require a stable process identifier or microVM reference and a threshold. The memory monitor assumes knowledge of how guest physical memory is mapped into host address space so it can subtract guest regions from resident set size totals. The central processing unit monitor polls aggregated utilization data from shared framework utilities and records over-threshold samples for the hypervisor process specifically.

The metrics subsystem wraps a third-party embedded metrics logger when an environment variable selects a namespace. It documents that dimension setting is stateful and that flushing between dimension changes is required to avoid accidental overwrites. A companion module builds dynamic schema objects from nested dictionary descriptions of expected counters, validates incoming JSON documents against that schema, compares timestamps to wall clock time within a one-second window, optionally augments the schema for architecture-specific and dynamically named device sections, and can flatten nested structures for export with dot-separated paths. A long-running monitor thread periodically pulls all accumulated metric snapshots from the microVM, forwards them to the cloud metrics sink, and asserts that keys suggesting failure modes are zero except for an explicitly enumerated set tied to known noisy scenarios.

The build and tooling helper exposes synchronous shell invocations for compiling Rust tests with pinned thread counts, locating prebuilt binaries in a default release directory, running the seccomp compiler and snapshot manipulation tools, and compiling small C sources with static linking behind a file lock to avoid races in parallel test runs.

The standalone Python script that exercises user datagram protocol offload accepts an address and port, opens a datagram socket, attempts to set a segmentation option, and sends a minimal payload. It is intended to be invoked as an interpreter script with command-line arguments.

The native helpers are compiled through the shared compile helper and invoked by tests as ordinary executables. One program connects to a virtio socket stream address and pipes standard input and output through the connection. Another loads a Berkeley Packet Filter program from disk, enables the appropriate privilege restrictions, and issues a single system call with numeric arguments for validation testing. A minimal program prints monotonic time in microseconds and echoes a command-line fragment for jailer timing tests. Another attempts to execute instructions that should raise an undefined opcode on hosts where that extension is not intended to be available. A memory-mapped utility opens privileged physical memory, maps a page-aligned region covering a device configuration window, and writes six bytes representing a link-layer address.

Callers must maintain invariants such as exclusive use of control socket paths, correct ordering of monitor start and stop relative to virtual machine lifetime, and availability of host privileges where raw memory access is required. The system guarantees deterministic cleanup for network namespaces when used through the object model, schema validation that rejects malformed metric trees, and fail-fast behavior when secure shell multiplexing cannot be established after bounded retries.

## Data Flow

The following narrative traces representative paths without reference to implementation symbols.

Host test code requests a secure shell session. The session establishment path sends a remote no-op command through the multiplexing client until success, optionally prefixing every host command with network namespace execution. Standard streams flow between the host runner and the remote shell. File copy operations use the secure copy protocol over the same multiplexed channel. Data enters as local paths and remote command strings and exits as byte streams and exit codes.

Network namespace setup flows from identifier strings through operating system commands that create or destroy namespaces. Tap creation flows from namespace context into interface creation, optional address configuration, and carrier polling for reuse safety.

Disk image creation flows from desired size and format into zero-filled files, filesystem formatting commands, and paths consumed by virtual machine configuration. Resize operations extend the underlying file and grow the filesystem metadata.

Memory monitoring ingests process memory maps from the operating system, filters regions by size to match known guest memory layouts on the primary 64-bit architecture, sums resident pages for remaining regions, and compares against a threshold. Samples surface as either quiet success or a structured error carrying the measured value.

Metrics validation ingests a JSON object produced by flushing the virtual machine’s metrics endpoint. The validator removes or checks timestamp fields, builds a structural schema, runs standard schema validation, and optionally performs mutation tests that temporarily remove required fields to ensure the schema catches omissions. Export to observability flattens nested dictionaries into dot-separated metric names, maps name suffixes to unit types, pushes values through the embedded metrics logger, and asserts absence of non-zero failure indicators in the flattened map.

The build helper moves string commands into subprocess execution with environment overlays for target directories and backtrace behavior. Binary resolution flows from artifact directory conventions to absolute paths.

The user datagram script sends a single small payload datagram toward a destination after socket option configuration, producing standard output status lines.

The virtio socket helper moves bytes between standard input and a connected stream socket. The seccomp helper reads binary filter bytecode from an open file, installs it, and passes through to a raw system call. The timing helper emits two lines to standard output. The undefined opcode helper prints status from microarchitectural wait instructions. The configuration space helper maps physical addresses through the memory device and writes octets.

The diagram below summarizes the major external data sources and sinks.

```
                    +------------------+
                    |  Test framework  |
                    |  & assertions    |
                    +--------+---------+
                             |
         +-------------------+-------------------+
         |                   |                   |
         v                   v                   v
 +---------------+   +---------------+   +---------------+
 | Host OS       |   | MicroVM API   |   | CloudWatch /  |
 | (ip, ssh,     |   | & guest       |   | EMF agent     |
 | psutil,       |   | (metrics,     |   | (optional)    |
 | sockets)      |   |  SSH)         |   |               |
 +-------+-------+   +-------+-------+   +-------+-------+
         |                   |                   ^
         |                   |                   |
         v                   v                   |
 +---------------+   +---------------+           |
 | Python        |   | Python        |           |
 | networking,   |   | metrics &     |-----------+
 | monitors,     |   | validation    |
 | build glue    |   +---------------+
 +-------+-------+
         |
         v
 +---------------+
 | Native        |
 | helpers       |
 | (compile      |
 |  on demand)   |
 +---------------+
```

Before the diagram, the key observation is that data originates from three distinct planes: the host operating system and its utilities, the microVM control plane and guest, and optional cloud telemetry endpoints. After the diagram, the consolidation point is the shared Python layer that normalizes those inputs and feeds assertions or remote agents.

## Control Flow

Execution is always triggered by test collection and individual test bodies. The critical path for guest interaction begins with namespace and tap setup, proceeds to virtual machine boot, then establishes the multiplexed secure shell session with retry loops, and only then runs remote commands or copies artifacts. Branching occurs when initial connection fails and triggers cleanup of a partially created control socket before retrying. Another branch occurs when optional error callbacks are registered to capture diagnostics on first failure.

Monitoring threads start when tests enter context managers or explicitly start threads. The central processing unit monitor loops until a stop flag is set, sleeping briefly between samples. The memory monitor loops similarly but exits early when thresholds are exceeded. The metrics export thread loops on a timer with one-second granularity for interruptibility, flushing accumulated snapshots on each wake until stopped.

The build helper runs synchronously in the test process except where file locking serializes concurrent compilation of the same helper binary.

Native helpers run as child processes with arguments supplied by tests. The seccomp helper branches on whether a filter path is the null device to skip installation. The virtio socket helper runs until end-of-file on standard input or an error in the transfer loop.

A small state-oriented view of the secure shell connection lifecycle appears below. Explanatory text before the figure: multiplexed sessions move from nonexistent to establishing, then to ready, and must pass through an explicit stop path to avoid orphaned daemons.

```
                    +-------------+
                    |  no socket  |
                    +------+------+
                           | establish (retry)
                           v
                    +-------------+
         error +---| connecting  |---> success
             |     +------+------+
             |            |
             v            v
      +-------------+  +-------------+
      | cleanup &   |  |   ready     |
      | retry       |  +------+------+
      +-------------+         | run / scp
                              |
                              v
                         +-------------+
                         |  closing    |
                         +-------------+
```

After the figure, the important condition is that readiness is proven by a successful remote no-op, and closure validates the control master process before sending stop and waiting for termination with a hard kill timeout.

## State & Lifecycle

Network namespace state exists in the kernel until explicitly deleted. Tap objects hold names and namespace references; taps are created eagerly on add. The secure shell helper holds paths to keys and control sockets and mutable callback hooks. Multiplexed connections leave kernel state unless stopped cleanly.

Memory monitors own a stop flag, current measurement cache, and optional reference to an offending process object when thresholds breach. Central processing unit monitors accumulate sample lists and a stop flag.

The metrics wrapper accumulates duplicate metric names in an in-memory dictionary for local persistence while optionally forwarding to the cloud logger. The long-running metrics monitor tracks an index into the list of snapshots already exported so each document is sent once.

Build artifacts on disk persist beyond a single test unless removed by broader harness cleanup; the compile helper skips rebuilding when the output already exists.

Native programs are stateless one-shot processes aside from kernel side effects such as installed filters or memory mappings held for the process duration.

Startup for monitored tests begins with thread creation and entry into polling loops. Steady state is characterized by periodic sampling or timer-driven export. Shutdown signals monitors to stop, joins threads, and for metrics export may issue a final flush action against the virtual machine before reading remaining snapshots. Recovery from partial failure relies on process exit and test teardown to reap resources; secure shell cleanup aggressively terminates stuck masters.

## Failure Modes

Secure shell establishment can fail due to guest boot delay, wrong keys, or namespace isolation issues. The design retries with fixed delay and surfaces the last exception. A race exists where multiplexing forks a daemon but the parent is killed early; the implementation avoids overly short timeouts on the establishing command to reduce orphaned sockets. Debug verbosity on the client is avoided because it interacts poorly with stream handling in the command runner.

Memory monitoring can fail to attach if the process vanishes; the monitor returns quietly in that case, which could mask a race if tests expect monitoring to always observe a live process. Guest memory sizing assumptions could misclassify regions if memory layout changes, leading to inflated or deflated overhead calculations.

Metrics validation is strict: schema mismatches raise validation errors, timestamp skew beyond one second fails assertions, and flattened failure-like keys with non-zero values fail export unless allowlisted. This strictness can produce false positives if legitimate tests expect non-zero failure counters; the cloud export path documents that limitation.

The embedded metrics library’s stateful dimensions can cause silent metric misattribution if callers forget to flush between dimension changes. The wrapper documents the required pattern.

Native helpers can fail with permission errors when raw memory or seccomp installation is not permitted. The user datagram script exits if socket options cannot be set. The virtio socket helper returns non-zero on read or write errors.

Silent corruption is most likely if memory region classification drifts without test updates, or if metrics schemas lag behind product changes but still validate superficially. The dynamic sections for named devices mitigate some drift by requiring presence of expected subtrees.

## Operational Characteristics

Resource consumption centers on additional threads per monitor, subprocesses for secure shell and copying, and periodic cloud metric emission. Network namespace and tap setup requires elevated capabilities typical of integration test environments. The metrics export thread sleeps most of the time but wakes every second to allow prompt shutdown.

Scaling bottlenecks include serialized compilation under file locks, cloud metrics API rate if snapshots are large, and sequential retry loops for secure shell readiness. Observability exists through standard logging in the metrics monitor on flush failure, structured assertion messages on metric validation, and optional local embedded metric format emission via environment variables for debugging.

## Design Rationale

The split between Python and small C programs reflects tradeoffs. Python integrates cleanly with the test framework, retry libraries, and JSON processing, while C provides minimal, explicit control over sockets, seccomp, memory mapping, and instruction sequences without interpreter overhead or feature gaps. Multiplexed secure shell connections amortize authentication and reduce latency for many short remote commands compared to spawning new sessions. Placing taps inside namespaces isolates tests and avoids interface name collisions on the host.

File-backed disk images default to temporary locations under a directory backed by a real disk because direct input-output in virtualized block backends expects block-aligned host files, which volatile memory-backed temporary filesystems may not satisfy. Monitoring resident set size with guest regions filtered targets virtual machine monitor overhead rather than allocated guest memory, which is often much larger and would dominate the signal.

Dynamic schema generation from expected metric trees keeps validation synchronized with nested structures and catches missing fields more robustly than ad hoc key checks. Flattening for cloud export aligns with dot-notation metric naming and enables uniform failure scanning. The allowlist for virtio-related read and write failure counters acknowledges known noisy shutdown behavior in external tools.

The metrics monitor uses incremental indexing of snapshots to preserve ordering and timestamps rather than collapsing everything at teardown, which aligns with time-series expectations. Stopping the monitor before destroying the virtual machine avoids races where the background thread wakes after teardown.

The undefined opcode helper encodes an explicit expectation about illegal instruction behavior for particular instruction extensions. The configuration space helper trades portability for power: it requires privileged access but enables tests that mutate emulated device state from the host in ways not exposed through standard management APIs.

Overall, the design favors explicitness, test determinism where possible, and strict validation of observability data, accepting that some tools require elevated privileges and that strict metrics assertions are inappropriate for negative testing scenarios.
