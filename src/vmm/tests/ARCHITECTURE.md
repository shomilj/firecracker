# Architecture: Virtual Machine Monitor Integration Tests and Microbenchmarks

This document describes the architecture of the Rust-level integration tests and performance experiments that exercise the virtual machine monitor library. It covers what behaviors are validated, how tests are structured and triggered, and how microbenchmarks isolate hot paths. It does not describe production runtime components inside the library itself; it treats the test and benchmark harnesses as a system whose responsibility is to give fast, deterministic feedback on correctness and relative performance of selected subsystems.

## Purpose & Boundaries

The responsibility of this testing layer is to prove, at the crate boundary, that the monitor can be constructed, driven through representative lifecycle transitions, and checked against invariants that would be expensive or awkward to assert only through external process-level tests. The layer is explicitly not responsible for full guest workload fidelity, network correctness, or disk semantics end to end; comments in the sources acknowledge that deeper guest restoration after snapshot load is validated elsewhere. The layer is also not a substitute for unit tests that live beside implementation modules; instead it composes public and semi-public APIs the way a controller or embedding process would.

If this layer failed entirely, the project would lose early signal on regressions in boot orchestration, snapshot serialization contracts, pause-gated operations, asynchronous host I/O wrappers, and emulated device integration with the host event loop. External automation might still catch some issues, but iteration would slow and faults would surface later. The microbenchmark suite, treated as part of the same architectural picture, exists to quantify the cost of virtio queue manipulation, block request decoding, configuration template serialization, and guest memory backing behavior under page faults—not to establish service-level objectives for a full microVM.

## Interfaces & Contracts

Integration tests consume the library the way an embedder does: they build configured monitor instances, attach or simulate devices, route control through the preboot and runtime control surfaces, and observe outcomes through return values, state queries, and serialized artifacts on disk. They depend on shared test helpers in the library that fabricate minimal but valid kernel images and default machine configurations so that tests focus on orchestration rather than asset curation.

The asynchronous I/O ring tests speak directly to the ring abstraction and its mapping to host files and memory regions. They assume the kernel exposes the completion-queue semantics under test and that temporary files and anonymous mappings behave as documented on Linux.

The emulated serial tests integrate the legacy serial path with the same event-loop style dispatcher used in production, driving readiness and hang-up conditions through OS pipes instead of real TTYs. The contract here is that subscribers react to edge and level triggered semantics consistently when input is drained in small steps versus bulk.

Benchmark binaries are declared without the default test harness so they can use an external benchmarking framework. Each binary links against the library and constructs synthetic guest memory regions, virtio queue layouts, or configuration objects in isolation. Callers of these binaries are human operators or continuous integration jobs; the binaries promise stable relative measurements under controlled conditions rather than absolute nanosecond claims portable across machines.

## Data Flow

Configuration and ephemeral inputs enter the system from several directions. End-to-end scenarios start from implicit defaults augmented by test-only kernel blobs and optional toggles such as dirty page tracking. Snapshot flows produce paired artifacts: a structured state blob and a memory backing file. Those artifacts flow back into the preboot path as paths on the host filesystem, exercising deserialization and rebuild logic. Sanity checks consume deserialized state in memory and mutate copies to force validation failures, ensuring defensive checks trigger when invariants break.

The asynchronous I/O path moves byte-sized operations between anonymous mapped buffers and temporary files. When submission queues fill, the harness drains completions and retries, which models backpressure without involving a real block device. Event notification uses a counter-style synchronization primitive wired into readiness polling so that completion delivery is observable without busy waiting.

Serial tests move bytes through anonymous pipes: the writing end injects patterns larger than the emulated FIFO so that multiple scheduling generations of the event loop interact. Guest-side consumption is simulated by repeated narrow reads from the emulated bus, interleaved with dispatcher runs.

Benchmarks fabricate guest memory images in RAM, lay out virtio descriptor tables and available rings, or hold CPU template graphs in heap form. The memory access experiments allocate guest RAM through the same resource object used for real machines, then touch a single byte through a raw pointer so the measurement isolates one page fault and its associated mapping cost.

The following diagram summarizes how data moves through the primary integration strands.

```
                    +------------------+
                    | Test assets and  |
                    | default configs  |
                    +--------+---------+
                             |
         +-------------------+-------------------+
         |                   |                   |
         v                   v                   v
+----------------+  +---------------+  +------------------+
| Build and boot |  | Snapshot      |  | API-driven pause |
| paths          |  | create/load   |  | and inspect      |
+----------------+  +-------+-------+  +------------------+
                             |
                             v
                    +------------------+
                    | Files on disk:   |
                    | state + memory   |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Reload via       |
                    | preboot path     |
                    +------------------+


     Async I/O strand          Serial strand
     +-------------+           +-------------+
     | mmap buffer |           | pipe bytes  |
     +------+------+           +------+------+
            |                         |
            v                         v
     +-------------+           +-------------+
     | ring <->    |           | event loop|
     | host file   |           | <-> emulated|
     +-------------+           | serial     |
                               +-------------+
```

Before the diagram, inputs were described as originating from defaults, synthetic kernels, and host files. After the diagram, the split between disk-backed snapshot round-trips and in-memory ring or pipe traffic should be clear: snapshot data persists across process boundaries within a test, while ring and serial tests emphasize transient buffers and kernel file descriptors.

## Control Flow

Execution is triggered by the standard test runner for integration crates, which compiles each integration test target separately. There is no single entry binary; concerns are split so that failures localize to boot orchestration, host I/O, or device event handling. Benchmarks are triggered by invoking their standalone binaries, which register measurement loops with the external framework.

The critical path for lifecycle tests begins with constructing or retrieving a monitor handle, optionally starting execution, then applying control operations such as pause, resume, or stop with a normal exit code. Architecture-specific branches exist: on one architecture the guest test kernel is expected to finish and signal completion within a bounded wait; on another the harness stops the machine explicitly because the test kernel does not exit cleanly. Dirty page tracking tests gate some assertions similarly.

Snapshot creation follows a pattern: allow the guest to run briefly, pause, issue a snapshot request through the runtime control surface, then stop. Loading uses the preboot control surface with empty security profiles and default resource placeholders, then asserts the machine reports running before an orderly stop. Ordering tests configure boot-related resources first, then attempt snapshot load and expect refusal, protecting against ambiguous preboot state.

The asynchronous I/O suite walks setup errors first, then progressive stress on queue occupancy, then full read and write sweeps. The serial suite interleaves dispatcher runs with simulated guest reads to force specific orderings of readiness, hang-up, and buffer-full conditions.

For benchmarks, control flows are trivial: initialize synthetic state once per group, then repeat the operation under measurement with black-box hints to prevent unwanted optimization. Memory access benchmarks use batched allocation so each iteration can fault a fresh page where the harness intends.

A compact view of control branching in lifecycle tests:

```
             Start test
                 |
                 v
        +--------+--------+
        | Need running    |
        | guest?          |
        +--------+--------+
                 |
       +---------+---------+
       | yes                 | no
       v                     v
  Run / sleep            Stay paused
  bounded wait           or configure
       |                     |
       +---------+-----------+
                 |
                 v
        Apply API action
        (pause, snapshot, etc.)
                 |
                 v
        Assert result / state
                 |
                 v
             Stop / exit
```

The prose before the figure noted architecture-specific branches and pause gating. After the figure, the takeaway is that most tests converge on a single assert phase after optional running time, keeping the critical path short except where intentional sleeps allow races or dirty pages to accumulate.

## State & Lifecycle

Integration tests own short-lived monitor instances scoped to a single test function. State begins at construction defaults, moves through paused or running machine states depending on the scenario, and ends at explicit stop. Snapshot tests additionally own temporary files whose lifetime extends across sub-steps within one test: first creation and serialization, then reload in a fresh controller context. Deserialization sanity tests clone logical state in memory and clear regions to simulate corruption or omission.

The asynchronous I/O ring tests own ring instances tied to registered host file descriptors and optional notification handles. State includes pending submission counts, completion backlog, and registered fixed file indices. Tests assert monotonic relationships between pushed, submitted, and completed work, and drain queues to reach a quiescent baseline.

Serial tests own mutex-protected emulated devices registered with the dispatcher. State spans FIFO occupancy, whether standard input remains registered, and whether hang-up was observable before the guest drained data. Two major scenarios differ by whether back-pressure logic unregistered the input before the peer closed the pipe, which changes which kernel events the dispatcher observes later.

Benchmarks minimize retained state between iterations except where the framework resets indices on the virtio queue wrapper. CPU template benchmarks print approximate payload sizes once to stderr for human context; the measured objects stay immutable during timing loops.

A simplified lifecycle for snapshot round-trip tests:

```
  [Configured VM] --> run briefly --> pause
       |                                 |
       |                                 v
       |                          write state + memory
       |                                 |
       v                                 v
    stop old VM                   new preboot controller
                                         |
                                         v
                                   load from disk
                                         |
                                         v
                                   running --> stop
```

Before this state machine, snapshots were described as paired artifacts and controllers as disposable. After it, the important invariant is that the old monitor must be stopped before a new one assumes responsibility, mirroring operational practice.

## Failure Modes

Assertions encode expected failures: missing boot configuration, illegal snapshot while running, illegal CPU configuration export while running, invalid ring parameters, disallowed snapshot load after conflicting preboot actions, and sanity check failures when deserialized state omits memory regions. These are explicit and loud; the tests do not generally catch silent corruption except where deserialized structures would diverge from expected counts of virtual CPUs or devices.

The asynchronous I/O harness distinguishes retryable queue pressure from programmer errors such as unregistered files or invalid indices. Completion entries carry success or failure per operation; restriction tests expect disallowed operations to fail at completion time even if submission succeeded.

Serial tests guard against deadlocks by using short timeouts in places where the dispatcher must go idle. A failure to observe the expected number of events usually indicates a regression in registration logic rather than flaky timing, because the scenarios are deterministic once pipe and FIFO sizes are fixed.

Benchmarks suffer from external noise: CPU frequency scaling, background processes, and memory pressure. The harness mitigates variance with configurable sample counts and noise thresholds per group. Misinterpretation remains a failure mode: a regression in unrelated system software can shift numbers without indicating a library change.

## Operational Characteristics

Integration tests exercise real kernel interfaces: asynchronous I/O rings, epoll-style dispatch, pipes, memory mapping, and file I/O. They are heavier than pure unit tests and may sleep to allow races or dirty memory to settle. Snapshot tests allocate disk space in temporary directories and can be sensitive to filesystem speed but not to network conditions.

Resource consumption scales with the number of virtual CPUs and memory size configured; most scenarios stay minimal. Parallel test execution at the cargo level may contend for CPU; the tests themselves rarely spawn extra threads beyond what the monitor already uses internally for virtual CPUs and workers.

Observability is primarily through assertion messages and optional prints in benchmarks. There is no structured tracing requirement inside this layer; failures are reproduced by rerunning a single test target.

Microbenchmarks emphasize repeatability: virtio queue benchmarks use a modest queue depth and prewired descriptor chains; block request parsing reuses a popped descriptor; template serialization uses fixed fixture text; memory benchmarks fault one page per iteration inside batched allocation to separate mapping cost from allocator noise.

## Design Rationale

Splitting integration tests by concern keeps compile and iteration times manageable and makes ownership clearer when a subsystem changes. Duplication between scenarios is acceptable where it keeps each test readable without indirection-heavy frameworks.

Using the same resource and control APIs as production embedders maximizes the value of each assertion; mocks are limited to serial input and test kernels. For snapshots, the design accepts that guest memory contents correctness is out of scope here, prioritizing wire format and control-flow invariants that Rust integration can cheaply verify.

The asynchronous I/O tests lean on small operations and explicit queue filling because they aim to exercise the ring state machine, not disk throughput. Serial tests use pipes and non-blocking descriptors to make event ordering deterministic compared with TTY-based interaction.

Benchmarks isolate hot paths identified in virtio and configuration handling without standing up full devices. Choosing synthetic memory layouts avoids KVM or userfaultfd variability where not relevant. The memory access benchmark compares default page size configuration against huge-page oriented configuration to give engineers a direct before-and-after signal when changing guest memory allocation policies.

The pairing of integration tests in the library’s integration-test area with separate benchmark binaries reflects a separation between correctness gates that run in ordinary test jobs and longer-running performance measurements that may run on dedicated hosts. Together they form a complementary envelope: behavioral coverage at the API edge and targeted measurement of internal algorithms and allocation paths that dominate overhead in production microVMs.

Benchmarks do not chain dependencies; each binary stands alone and links only the monitor library plus shared test helpers. The following topology clarifies how the experiments relate to the same codebase without sharing runtime state.

```
  +------------------+     +------------------+     +------------------+
  | Virtio queue and |     | Block virtio     |     | CPU template     |
  | descriptor walks |     | request decode   |     | JSON round-trip  |
  +---------+--------+     +---------+--------+     +---------+--------+
            |                        |                        |
            v                        v                        v
  +---------------------------------------------------------------+
  |                     Library under test                         |
  +---------------------------------------------------------------+
            ^
            |
  +---------+--------+
  | Guest memory     |
  | page fault cost  |
  +------------------+
```

Before this topology diagram, benchmarks were characterized as independent executables with synthetic inputs. After it, the reader should see that virtio-related benches share themes around descriptor handling while the memory bench routes through resource configuration to approximate real allocation paths, and template serialization measures an orthogonal configuration pipeline.

Together, these strands implement a test architecture that mirrors how operators and embeddings drive the monitor: configure, run, pause, snapshot, reload, and inspect. The asynchronous and serial suites deepen coverage where OS integration and event ordering matter. Microbenchmarks close the loop by quantifying the cost of frequent inner-loop operations, giving teams a way to relate code changes to predictable performance deltas without conflating them with full stack integration noise.
