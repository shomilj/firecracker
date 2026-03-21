# Firecracker binary and HTTP control plane

1. **Purpose and boundaries**

   1.1. The crate is the user-facing process that hosts the virtual machine monitor: it parses command-line configuration, optionally exposes a local HTTP API over a Unix domain socket, and drives an event loop that runs the guest and devices.

   1.2. Core emulation, device models, snapshot formats, and RPC semantics live in separate library crates. Here, the focus is orchestration: wiring logging and metrics, applying host-level policies, bridging HTTP to typed virtual-machine actions, and coordinating threads so the API remains responsive while the main loop services devices and virtual CPUs.

   1.3. A minimal library surface re-exports only the HTTP server module so tests and embedders can reuse the same parsing and routing stack without pulling in the full binary entry point.

2. **Process architecture**

   2.1. **Single process, two dominant roles after boot.** One thread owns the HTTP server when the API is enabled; another owns the event manager that multiplexes timers, device events, and virtual CPU activity. They communicate through bounded channels and a notification file descriptor so the API never blocks inside the emulation core except during intentional pause handling.

   2.2. **API thread.** Listens on a Unix socket, accepts HTTP/1.1, enforces a maximum request body size, applies a dedicated seccomp program, and turns each request into a strongly typed action destined for the monitor. For each synchronous action it blocks until the monitor returns a result, then maps that result to HTTP status codes and JSON bodies.

   2.3. **Monitor thread (event loop).** Runs the epoll-driven dispatcher that advances the virtual machine. It registers subscribers for the metrics timer, the API bridge, and everything the monitor attaches. When the API signals work, this thread drains the request, dispatches through a runtime controller, and sends the response back to the API thread.

   2.4. **Why split threads.** Network I/O and JSON parsing stay off the hot path of device emulation. The syscall surface differs per thread, which is why seccomp policies are issued per logical category rather than one global filter.

3. **Startup and configuration**

   3.1. **Early initialization.** Logging is initialized first so subsequent failures are visible. Host page size is queried once because it influences memory layout assumptions elsewhere. A panic hook runs before abort-on-panic termination: it logs the panic, restores terminal mode on standard input, records a panic metric, and attempts a final metrics flush so operators see failure signals even on crash.

   3.2. **Identity and limits.** The instance identifier is validated and stored globally for log correlation. Optional log and metrics sinks are configured from paths; metrics initialization is separate from the periodic flush machinery and may target a fifo or file.

   3.3. **Signal handling.** OS signals are registered through the monitor’s helper so shutdown and diagnostics behave consistently whether or not the API is present.

   3.4. **aarch64 hardening.** On that architecture, speculative store bypass disablement may be requested through a process-control syscall. Failure is logged but not fatal, matching the principle that hardening is best-effort on heterogeneous hosts.

   3.5. **File descriptor table sizing.** Before heavy work, the process may expand its descriptor table to the effective `RLIMIT_NOFILE` (or a conservative default when unlimited). The technique duplicates a low-numbered descriptor to a high slot then closes it, forcing upfront allocation. The goal is to avoid kernel reallocations of the fd table during snapshot restore when many eventfds and timerfds are live—those reallocations showed tens of milliseconds of latency in production traces.

   3.6. **Seccomp selection.** Configuration is trichotomous: no filtering (empty programs, discouraged), built-in advanced filters embedded at compile time, or user-supplied serialized BPF maps. Custom maps must deserialize into exactly three labeled programs; missing or unknown labels are rejected at startup rather than silently ignored.

   3.7. **Bootstrapping the guest.** Two paths exist. With a JSON configuration file, the monitor is built and booted directly from that document, optionally after seeding metadata for the guest metadata service. Without a file, the process enters a pre-boot phase where only configuration actions are accepted until the guest is explicitly started via an action. An alternate mode disables the API entirely and requires a configuration file, yielding a headless appliance.

   3.8. **Payload and metadata limits.** HTTP bodies are capped with a configurable maximum; the same value defaults the metadata store size limit unless overridden, keeping API and in-guest service constraints aligned.

4. **HTTP control plane**

   4.1. **Transport.** The server uses HTTP/1.1 over a Unix domain socket with keep-alive. Multiple connections may be multiplexed on the API thread via epoll; each accepted connection is processed to produce one or more requests per scheduling iteration.

   4.2. **Request lifecycle.** Incoming bytes become parsed requests. Each request is classified by HTTP method and path shape. Path parsing trims leading slashes and tokenizes segments so resources with identifiers embed the id in the path (drives, network interfaces, etc.).

   4.3. **Parsing philosophy.** Bodies are deserialized with Serde where JSON is expected. Many resources use `deny_unknown_fields` to catch typos early. Validation errors become HTTP 400 responses with a JSON object carrying a single fault string; some monitor errors map to 413 when a size limit is exceeded. Successful operations return either 204 without a body or 200 with JSON, depending on whether data is returned.

   4.4. **Deprecation.** Parsers may attach deprecation messages when legacy shapes are accepted. Those messages surface as warnings in logs and may mark responses as deprecated at the HTTP layer so clients can detect transitional behavior.

   4.5. **Logging discipline.** Request lines describe method and path; for most bodies the raw payload is logged at info level. CPU template payloads are special: at default log levels the template content is omitted and a hint points operators to debug logging, because templates can be large and sensitive.

   4.6. **Metrics on the API thread.** Certain endpoints increment counters for attempts and failures (for example actions). Latency histograms are updated for long-running monitor operations after success: creating snapshots, loading snapshots, pausing, and resuming. The stopwatch starts when request processing begins on the API thread and ends when the monitor acknowledges success, so the metric reflects end-to-end user-perceived latency for those operations.

   4.7. **Seccomp on the API thread.** Before serving traffic, the API program is installed. Failure to install is fatal because running without the intended syscall restrictions violates the security model.

   4.8. **Process time reporting.** Optional timestamps supplied on the command line are converted into startup metrics so orchestration layers can attribute process creation vs CPU time accurately.

   4.9. **Shutdown.** A separate non-blocking eventfd acts as a kill switch. When the monitor finishes, it signals this fd; the HTTP server observes shutdown, flushes partial writes where applicable, and exits the accept loop cleanly.

5. **Routing map (conceptual)**

   5.1. **Read-only inspection.** Root path returns instance information. Version and machine configuration have dedicated endpoints. Balloon configuration and statistics are exposed via distinct GET shapes. The metadata service supports GET for the document and GET for configuration when applicable.

   5.2. **Configuration writes.** PUT handlers exist for boot source, machine configuration, drives, network interfaces, vsock, entropy device, logger and metrics sinks, balloon, CPU configuration, metadata content and metadata network configuration, and snapshots (create/load sub-operations encoded in the path).

   5.3. **Partial updates.** PATCH handlers allow fine-grained updates where the monitor supports merging: balloon fields, drive paths, machine configuration fields, network interface fields, metadata patches, and virtual machine state (pause/resume via PATCH in addition to action-based flows where implemented).

   5.4. **Imperative actions.** A PUT on the actions resource accepts JSON discriminated by `action_type` to flush metrics, start the microVM after configuration, or send guest signals where the architecture permits.

   5.5. **Method/body rules.** GET requests must not carry bodies; PUT/PATCH without bodies are rejected; unknown method/path pairs yield a structured invalid-route error. Resource identifiers embedded in paths must be non-empty and alphanumeric with underscores—mirroring orchestrator-friendly naming rules.

6. **Bridge between HTTP and monitor**

   6.1. **Channels.** The API thread sends boxed actions on a queue to the monitor thread. A semaphore-style eventfd is incremented for each enqueue so the epoll loop wakes even if multiple requests batch before the monitor runs.

   6.2. **Synchronous semantic.** Each HTTP request that maps to an action is processed as one round trip: send, wake, block on the response channel. This keeps client semantics simple—responses reflect completion or failure of the monitor operation.

   6.3. **Response mapping.** Monitor success variants serialize to JSON or empty responses. Errors stringify into fault bodies with appropriate status codes. The bridge does not reinterpret business logic; it translates typed errors into HTTP.

7. **Runtime integration after boot**

   7.1. **Adapter subscriber.** Once the guest is running, a dedicated subscriber holds the runtime controller and the receive side of the API queue. On eventfd read, it tries a non-blocking receive first; if a pause action arrives, it enters a nested loop that blocks on the queue until resume, intentionally starving the event manager of turns. That design freezes device emulation and timer handling while the guest is paused through the API, making pause/resume a true quiescence point for the thread that drives devices.

   7.2. **Spurious wakeups.** If the eventfd fires but no message is ready, a warning is logged—this should be rare and points to coordination bugs or signal races.

   7.3. **Termination.** The event loop polls the monitor for shutdown exit codes. Clean shutdown with success ends the loop; other codes propagate as errors to the top-level binary. After the loop, the kill switch fires so the API thread drains and joins.

8. **No-API mode**

   8.1. **Seccomp subset.** The API filter is dropped from the map so only monitor and vCPU programs apply—there is no API thread to constrain.

   8.2. **Event loop only.** The guest is built from JSON, periodic metrics start, and the process runs until the monitor reports a normal or abnormal shutdown. This mode suits fixed appliances and tests that do not need dynamic reconfiguration.

9. **Periodic metrics**

   9.1. **Timer-driven flushes.** A monotonic timerfd is armed with a fixed period (on the order of tens of seconds). On startup, an immediate flush captures early process metrics; each subsequent timer tick writes the global metrics blob.

   9.2. **Failure handling.** Write failures increment a missed-metrics counter and log an error; the timer continues to fire so transient sink issues do not permanently silence telemetry.

   9.3. **Integration.** The metrics subscriber is registered with the same event manager as devices, so metric cadence and emulation share the epoll instance in monitor-thread modes.

10. **Seccomp pipeline**

    10.1. **Default filters.** At release build time, architecture- and target-specific JSON policies compile to serialized BPF bytes that ship inside the binary. Debug builds may substitute empty policies so developers on unsupported toolchains still compile.

    10.2. **Custom filters.** Operators can supply their own serialized map; deserialization uses the same schema validation as defaults.

    10.3. **Thread labels.** Programs are keyed by logical roles—API, VMM, vCPU—so each thread installs the minimal syscall allowlist for its workload. The crate validates that all three exist and that no extraneous keys appear, preventing partially effective policies.

11. **Auxiliary artifacts**

    11.1. **Machine-readable API description.** A YAML document elsewhere in the tree describes resources and schemas for tooling; it is not consulted at runtime but aligns with the routing and parsing behavior.

    11.2. **Examples.** Small programs demonstrate seccomp filter generation and userfaultfd handlers; they illustrate integration patterns rather than production paths.

    11.3. **Policy tests.** One integration test scans workspace manifests to forbid non-caret dependency version constraints, keeping builds reproducible across the workspace.

12. **End-to-end control flow (API mode)**

```
  Client processes (CLI, orchestrator)
           |
           |  HTTP/1.1 over Unix socket
           v
 +---------------------+
 |   API thread        |
 | - parse & validate |
 | - seccomp (API)    |
 | - send action +    |
 |   wake eventfd     |
 +----------+----------+
            |
            |  channel + eventfd notify
            v
 +---------------------+
 | Event loop thread   |
 | - epoll dispatcher  |
 | - runtime controller|
 | - devices & vCPUs   |
 | - metrics timer     |
 +----------+----------+
            |
            |  responses on channel
            v
 +---------------------+
 |   API thread        |
 | - map to HTTP       |
 +---------------------+
```

13. **Operational characteristics**

    13.1. **Failure modes.** Socket bind failures surface clearly when the path is already owned. Parser errors never crash the server thread—they return 4xx responses and keep listening. Monitor failures bubble as structured HTTP errors and may carry non-zero process exit classification when the guest stops abnormally.

    13.2. **Security stance.** Seccomp is on by default; disabling it is explicit. API payloads are bounded. Metadata size defaults track API limits. Pause handling stops the event loop progression, reducing race windows for snapshot operations initiated through the API.

    13.3. **Performance considerations.** Separating I/O parsing from emulation reduces contention. Upfront fd table sizing avoids rare kernel reallocations during restore. Latency metrics focus on expensive monitor calls so operators can see snapshot and pause/resume costs directly.
