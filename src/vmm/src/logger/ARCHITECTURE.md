# Logger Subsystem Architecture

This document describes the architecture of the logging and metrics subsystem that serves the virtual machine monitor. It explains how human-readable diagnostic output and machine-readable operational statistics are produced, stored, and related to one another. The description is conceptual; it avoids implementation identifiers so that readers can reason about behavior and tradeoffs without reading source code.

## Purpose & Boundaries

The subsystem has two primary responsibilities. First, it provides a single, process-wide channel for human-readable diagnostic messages that originate from many concurrent threads across the monitor, the API surface, and device emulation. Those messages are ultimately written as newline-terminated text lines to either standard output or to an administrator-supplied destination such as a regular file or a named pipe. Second, it maintains a large, hierarchical collection of numeric operational metrics—counters, latencies, and a few gauge-like values—that summarize behavior over time and can be serialized as structured data for external collection.

What this layer does not do is equally important. It is not a distributed tracing system, a structured logging framework with arbitrary key-value fields per event, or a log aggregation pipeline. It does not perform sampling-based tracing, correlation identifiers across services, or retention management. Metrics are not pushed to a remote backend from within this module; they are written to whatever byte sink the host process configured at initialization, and any periodic scheduling or network forwarding happens outside this boundary. The subsystem also does not implement security policy beyond what the host operating system enforces on the chosen output paths.

If this component failed catastrophically—if every write path errored silently or if the global metric state were corrupted—the rest of the monitor would lose visibility: operators would not see diagnostic lines during incidents, and automation would not receive fresh counters. The guest might continue to run if the failure were confined to observability, but operational response would degrade because the control plane could not distinguish normal operation from subtle failure modes. Conversely, loss of metrics does not by itself stop virtualization, but it undermines capacity planning, regression detection, and incident triage.

## Interfaces & Contracts

Externally, the logging half of the subsystem behaves like a standard logging facade integration: callers emit messages at various severity levels through shared macros, and a global maximum severity threshold determines which messages are evaluated before they reach the custom back end. The back end itself applies an optional module-path filter so that only messages whose logical origin matches a configured prefix are formatted and written. Administrators configure the destination path, severity threshold, whether severity and source location appear in each line, and the optional module prefix through a configuration structure that can be deserialized from the product’s API. A stable default severity aligns with the product’s published specification. Instance identity for log lines is supplied once per process via a lazily initialized string; until set, a well-known placeholder identity is used.

The metrics half exposes a single global aggregate object that dereferences to the full metric tree. Callers across the codebase increment counters, store elapsed times, and update aggregates through small wrapper types that use atomic operations for hot paths. Initialization accepts a buffered writer wrapping a file descriptor; this is expected to happen once during startup on a dedicated code path. A write operation serializes the entire tree and appends a record terminator. Incremental counters are defined to reset their visible period totals when serialized, so the contract for “counter” semantics is delta-based per flush rather than monotonic forever.

Callers must respect several invariants. Metrics initialization must not be attempted twice. Logging configuration updates should not be assumed lock-free; they take an exclusive lock around internal state. The optional module filter uses prefix matching on the logical module path; messages without path metadata are dropped when a filter is active. Signal handlers and other constrained contexts are documented in the implementation to avoid unsafe interactions with the metric serialization path; deadly signal counters use a different storage shape than high-frequency incremental counters specifically to reduce races when multiple threads might serialize concurrently.

The guarantees in return are: log lines follow a single textual format when emitted; failed writes to the log destination are counted without recursive logging; metric serialization emits a self-describing structured record with a leading wall-clock timestamp in milliseconds; and incremental metrics report the change since the previous successful serialization for each counter, not the absolute total since process start.

## Data Flow

Human-readable logging begins when application code formats a message and submits it through the logging facade. The facade applies the global severity filter. If the message passes, the custom back end receives a record containing the formatted arguments, severity, optional source file and line, optional module path, and the emitting thread’s identity. The back end acquires exclusive access to its configuration, evaluates the optional module prefix rule, and either returns without output or builds a single line. That line starts with a local timestamp, then a bracketed segment that always includes the instance identifier and thread name, optionally augmented with severity and source location, followed by the message body and a newline. The bytes are written either to the configured file descriptor or to standard output. When the write fails, a dedicated counter in the metrics tree is incremented; no secondary error line is written to avoid feedback loops.

The following diagram summarizes the logging data path from multiple producers to a single sink.

```
  +----------------+     +------------------+
  |  Many threads  |     |  API / config    |
  |  (devices,     |     |  (level, path,   |
  |   vCPU, etc.)  |     |   module filter) |
  +-------+--------+     +---------+--------+
          |                        |
          v                        v
  +---------------+    +---------------------+
  | Logging       |    | Logger configuration |
  | facade        |--->| (max severity,       |
  | (severity     |    |  target, format,    |
  |  gating)      |    |  module prefix)     |
  +-------+-------+    +----------+----------+
          |                       |
          +-----------+-----------+
                      v
            +-------------------+
            | Line formatter    |
            | (time, id, thread,|
            |  optional meta)   |
            +---------+---------+
                      |
          +-----------+-----------+
          v                       v
   +-------------+       +-------------+
   | Standard    |       | Configured  |
   | output      |       | file / pipe |
   +-------------+       +-------------+
```

Before this diagram, the key idea is that all threads funnel through one formatter and one configuration gate, which enforces a consistent line shape and optional filtering. After the diagram, note that the exclusive lock around configuration means bursts of logging from many threads serialize at the formatter; this is a deliberate tradeoff for simplicity and a single output ordering.

Metrics data flow is different. Hot paths only touch atomics: workers increment shared counters or overwrite stored values with minimal contention. Periodically or on explicit request, a cold path serializes the entire metric aggregate to the initialized writer. Serialization walks a tree that includes first-class sections for the API, virtual CPUs, signals, security filtering, latency buckets, and logging health, plus flattened sections that delegate to device-specific metric bundles so that block, network, socket, entropy, balloon, and related devices contribute their own nested objects without the core tree hard-coding every field. The serializer injects a UTC millisecond timestamp as the first field of the output record. Incremental counters implement their serialization by computing the difference between a running total and a remembered baseline, then advancing the baseline on success so the next emission reports only new activity since the last flush.

The metrics sink path is illustrated below.

```
  +------------------+       atomic inc/store
  | All subsystems   | ------------------------+
  | (API, devices,   |                       |
  |  VCPU, signals)  |                       v
  +------------------+              +------------------+
                                    | Global metric    |
                                    | aggregate        |
                                    | (tree + device   |
                                    |  proxies)        |
                                    +--------+---------+
                                             |
                         +-------------------+-------------------+
                         |                                       |
                         v                                       v
                  +--------------+                      +---------------+
                  | Serialize to |                      | On failure:   |
                  | JSON + newline                      | count miss,   |
                  | + timestamp                         | optional log  |
                  +------+-------+                      +---------------+
                         |
                         v
                  +--------------+
                  | Buffered     |
                  | writer ->    |
                  | configured   |
                  | file         |
                  +--------------+
```

The diagram shows that producers never format JSON themselves; they only mutate counters. The expensive work is batched in one serialization pass, which keeps instrumentation call sites small and uniform.

## Control Flow

Logging is entirely event-driven from the perspective of the subsystem: every log line is triggered by some thread calling into the logging macros or equivalent APIs. There is no internal timer or polling loop for logs. The critical path is: acquire configuration lock, evaluate module filter, build the line string, write bytes, optionally increment a missed-log counter on error, release lock. Configuration updates follow a separate path invoked when the control plane applies new logger settings; that path adjusts global severity, opens or switches the output descriptor when a path is provided, and toggles format flags.

Metrics have two control paths. The hot path is invoked constantly from normal execution: increments, stores, and scoped latency recorders that finalize on scope exit. The cold path is invoked when something calls the explicit write operation: serialization runs, the writer receives one complete record, and incremental baselines advance inside the serializer for compatible counters. In a full product deployment, a timer in the host process typically drives this write on a fixed interval on the order of one minute, and an API action can also request an immediate flush. Startup code may invoke an initial write so that early lifecycle metrics appear without waiting for the first timer tick. Process termination and signal handling may attempt a final write so that counters survive abrupt exits when safe.

Branching inside the logger back end is straightforward. If a module filter is configured and the incoming record lacks a module path, the message is dropped. If the filter is configured and the path does not begin with the configured prefix, the message is dropped. If no destination file is configured, standard output is used. If the write fails, control returns after bumping the missed counter; there is no alternate sink.

## State & Lifecycle

The logging subsystem owns the current output target (if any), the format flags for optional severity and source location, the optional module prefix string, and the lock that serializes concurrent access to those fields. It also relies on a process-wide slot for the instance identifier string, set once when the product establishes context. The global maximum log level is owned by the logging facade and is updated when configuration is applied.

The metrics subsystem owns the entire metric tree, the optional destination writer wrapped for interior mutability, and all atomic cells inside the metric primitives. Incremental counters maintain two machine words: one accumulates live updates, the other stores the last serialized snapshot. Store-style metrics keep a single word. Latency aggregate groups combine store-backed min and max with an incremental sum so that averages can be reconstructed by consumers.

At startup, the logger is registered with the facade and default severity is applied. Detailed configuration may arrive later via the API. The metrics writer is attached exactly once; until then, explicit write calls succeed but report that nothing was written, which allows early code to probe without error. During steady operation, threads contend on the logger lock only when emitting lines, and on individual atomics when recording metrics; they do not contend on the metrics writer except during serialization. On shutdown, the host process may flush metrics a final time; logging may continue until the process exits. Recovery from partial failure is minimal: if metrics were never initialized, writes are no-ops; if the logger was never pointed at a file, output goes to standard output.

A simple state-oriented view of logger configuration is shown next.

```
                    +----------------+
                    | Unregistered   |
                    | (pre-init)     |
                    +-------+--------+
                            | register facade
                            v
                    +----------------+
                    | Active with    |
                    | defaults       |
                    +-------+--------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
     +----------------+          +----------------+
     | Stdout only    |          | File/pipe      |
     | (no path set)  |          | configured     |
     +----------------+          +----------------+
              \                       /
               \                     /
                v                   v
                    +----------------+
                    | Reconfigured   |
                    | via API        |
                    +----------------+
```

The prose before the figure noted startup and late-bound configuration; the figure afterward emphasizes that the process moves from default behavior to operator-chosen targets and may move between them over the process lifetime as configuration is reapplied.

## Failure Modes

Log write failures are deliberately silent at the text level: the subsystem does not print a secondary error to standard error when the primary destination fails, because that could amplify I/O problems or recurse into logging. Instead, a counter records how many lines failed to land. This design favors stability and predictable overhead over maximal diagnosability of sink outages.

Metrics write failures return errors to the caller of the write operation. The host integration is expected to log that failure through the logging path and increment a missed-metrics counter, which creates a feedback path: persistent sink problems surface both as log messages and as rising miss counts. Serialization errors are treated as hard failures for that flush attempt. Double initialization of the metrics destination is rejected so that the process cannot accidentally redirect the writer to two different backends without a clear error.

The incremental counter serialization strategy has a subtle integrity property: if serialization fails after computing a snapshot but before advancing the baseline, the implementation is written so that unsuccessful attempts should not advance the baseline, preventing silent loss of counts. Conversely, any successful serialization advances the baseline even if downstream consumers drop the data, which means external loss after a successful write is not detectable here.

Violated assumptions that could cause silent corruption include: concurrent unsynchronized serialization of the full tree from multiple threads without the intended locking (the implementation assumes a mostly single-threaded flush path); misuse of incremental counters where a non-atomic print or debug dump triggers serialization side effects and resets counters unintentionally; and signal handlers racing with a full-tree write. The design mitigates the last case for a subset of critical signal counters by using store-style metrics that serialize without the same reset pattern as high-churn counters.

## Operational Characteristics

Resource use is dominated by three factors: contention on the single logger lock under high log volume, atomic traffic on hot counters, and the size and cost of periodic JSON serialization. Logging has no rate limiting or sampling in this layer; if the system is configured for verbose severity and many subsystems emit at once, the formatter and output descriptor become the bottleneck. The optional module prefix filter reduces volume only when operators constrain the emitting namespace; it is not a throttle.

The metrics system is designed for lock-light hot paths: increments use relaxed atomic fetch-add on contiguous words, avoiding mutexes in the steady state. Serialization is necessarily heavier: it walks the entire tree, invokes nested serializers for device modules, and allocates through the serializer. That work is amortized across the flush interval when a timer drives it. The buffered writer type used with metric output batches writes to reduce syscall overhead.

Observability of the observability layer itself includes dedicated metrics for missed log lines, missed metric flushes, and failures in logging and metrics handling. Logging lines include timestamps and optional origin metadata for grep-friendly forensics. There is no first-class trace or span identifier in this subsystem; correlation is limited to thread name, instance identity, and time.

## Design Rationale

The split between a logging facade and a custom back end exists to align with ecosystem conventions while preserving a product-specific line format and integration with named pipes for containerized deployments. Using standard severity levels and macros minimizes friction for contributors and keeps static analysis and tooling familiar. The custom back end exists because a single process-wide policy for formatting, filtering, and destination selection is easier to reason about than ad hoc prints scattered across modules.

The metrics design favors process-wide aggregation with atomic primitives so that instrumentation can live on hot paths in the emulator without introducing mutexes for every counter. The two-word incremental pattern avoids a separate reset write from the flushing thread: the serializer thread reads both words and advances the baseline during serialization, which reduces coordination compared to resetting counters after the fact. The tradeoff is semantic: published counters represent deltas over flush intervals (or whatever period elapses between writes), not lifetime totals, which matches periodic reporting and rollups better than monotonic ever-growing numbers that require external differencing.

Flattening device metrics through proxy serialization keeps the central tree stable while allowing device crates to own their field definitions. That modularity reduces merge conflicts and keeps device teams responsible for their own observability surface.

Choosing optional inclusion of severity and file location in log lines reflects dual audiences: operators who want compact journal-style lines versus developers who need pinpoint context during bring-up. Defaulting those off preserves stable, shorter lines in production while allowing verbose diagnostics on demand.

The module prefix filter is a coarse tool compared to per-logger hierarchical filters in larger frameworks, but it maps well to Rust’s module tree and gives operators a single knob to focus on a subtree during investigations without recompiling.

Non-blocking open flags on the configured log path interact with how administrators wire FIFOs and pipes: the process can avoid blocking indefinitely on open when the other end is not yet connected, at the cost of surfacing errors through write behavior and missed counters rather than at startup. That behavior is a deployment-facing tradeoff between startup progress and strict fail-fast semantics.

Finally, keeping metrics as in-memory structures written to a local file aligns with the embedded nature of the monitor: no network dependency, predictable failure modes, and straightforward testing by pointing the writer at a temporary file. External systems poll or tail the output; scaling concerns shift to how often the process flushes and how large each JSON document grows as features add counters, not to distributed back pressure within this crate.

---

Together, the logging pipeline and metrics sink form a thin but critical plane: they translate the monitor’s internal behavior into operator-visible text and machine-readable statistics, with explicit tradeoffs around locking, atomicity, delta semantics, and failure visibility that favor predictable overhead in the hot paths and batched work on the cold paths.
