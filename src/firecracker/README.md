# Firecracker process shell and HTTP control plane

1. **Role in the overall system**

   1.1. The crate is the runnable surface of the product: it owns process startup, optional local HTTP over a Unix domain socket, and the glue that turns bytes on that socket into work for the virtual machine monitor while a separate scheduling loop runs the guest and devices.

   1.2. Heavy emulation, device models, snapshot formats, and the detailed semantics of each configuration knob live in library crates. Here the emphasis is orchestration—how threads divide responsibility, how syscall policies attach to those threads, how bounded queues and kernel notification primitives couple the socket side to the emulation side, and how periodic telemetry shares the same polling loop as devices without starving either path under normal conditions.

   1.3. A thin library surface re-exports only the HTTP server module so integration tests and embedders can exercise the same parsing and routing stack without linking the full binary entry point.

2. **Process and thread model**

   2.1. After launch there are two dominant roles whenever the socket API is enabled: a dedicated thread that owns all HTTP acceptance, parsing, and response serialization; and the thread that owns the epoll-driven event manager used to multiplex device activity, virtual CPU kick events, the API bridge subscriber, and the metrics timer. They exchange work through bounded channels plus a semaphore-style kernel counter used to wake the monitor side when new control-plane work is pending.

   2.2. The API thread’s workload is almost entirely I/O and JSON: accept connections on the Unix socket, drain readable data, map each logical HTTP exchange to a single synchronous round trip through the channels, then block on the return channel until the monitor thread has produced a typed outcome. That design keeps JSON and string formatting off the hot emulation path and allows a narrower seccomp profile on the socket thread than on the thread that runs timers and device callbacks.

   2.3. The monitor thread runs the event loop that advances guest time, services virtio and other backends, and applies control operations delivered from the API. Subscribers register their kernel handles with the same epoll instance; readiness on any handle schedules the corresponding callback. Order among ready handles is managed by the event manager implementation, but the important architectural guarantee is that long stretches of work on one subscriber delay all others—including the periodic metrics flush—unless the implementation explicitly yields.

   2.4. Virtual CPU threads are spawned by the monitor stack, not by this crate directly, but their syscall surface is still constrained by a third compiled BPF program distributed alongside the API and monitor programs. That separation reflects different workloads: the API rarely touches KVM ioctls, the monitor coordinates devices and KVM, and each hardware thread runs a tight guest entry loop.

3. **Startup sequence (API-enabled path)**

   3.1. Logging is initialized before almost everything else so early failures are visible. Host page size is queried once up front because memory layout code elsewhere assumes a stable answer for the lifetime of the process.

   3.2. Standard input is captured for a panic hook. Because the binary is built with abort-on-panic, the hook runs immediately before termination: it logs the panic, attempts to restore the terminal to canonical mode, increments a panic counter in the global metrics blob, and tries one last metrics write so operators still see a failure signal when the process dies abruptly.

   3.3. Command-line parsing establishes identity, logging sinks, optional metrics sinks, seccomp mode, payload limits, optional one-shot JSON configuration, optional guest metadata seeding, and whether the HTTP API is disabled entirely. Instance identity is validated and stored globally for log correlation.

   3.4. Signal disposition is registered through the monitor’s helper so graceful shutdown and diagnostics behave consistently regardless of API presence.

   3.5. On AArch64 hosts, speculative store bypass mitigation may be requested through the process-control interface. Failure is logged but not fatal—hardening is treated as best-effort on heterogeneous silicon.

   3.6. The file descriptor table may be expanded early to match the effective open-file limit (or a conservative default when the limit is unlimited). The technique duplicates a low-numbered descriptor into a high slot and immediately closes the duplicate, forcing the kernel to grow the table up front. The motivation is snapshot restore: complex guests register many event and timer descriptors with the polling loop; growing the kernel’s internal fd table while those descriptors are live showed multi‑tens‑of‑milliseconds stalls in production traces.

   3.7. Seccomp configuration is resolved to one of three outcomes: no filtering (empty programs for every role—explicitly discouraged), built-in advanced filters shipped as serialized BPF bytes inside the binary, or operator-supplied serialized maps read from disk. Custom maps must deserialize into exactly three labeled programs with no extra keys; missing or unknown labels fail closed at startup.

   3.8. When the API is on, the adapter path constructs two kernel event counters: one semaphore-style counter used to wake the monitor whenever at least one API message is enqueued, and one non-blocking counter wired into the HTTP server as a shutdown switch. Two bounded channels carry requests toward the monitor and responses back toward the socket thread. The API program is removed from the mutable filter map and handed to the socket thread; the remaining map is consumed when building the guest.

   3.9. The HTTP listener is created and the kill switch is registered before the socket thread is spawned named for observability. The monitor constructs its event manager, registers the periodic metrics helper, and then either boots straight from a JSON document or enters a pre-boot phase that blocks only on API traffic until configuration completes.

   3.10. After a successful build, the periodic metrics engine is started with a fixed period, then the adapter installs itself as another subscriber and the monitor loop runs until the guest reports a terminal shutdown code. Tearing down writes to the kill switch, joins the API thread, and propagates errors according to the exit-code mapping rules.

4. **HTTP server lifecycle on the API thread**

   4.1. Before any request is accepted, the socket server’s maximum body size is set from the configured limit so oversize frames are rejected at the transport layer without touching application parsers.

   4.2. The API-specific BPF program is installed on this thread. Failure to install is treated as fatal: running the socket without the intended syscall restriction is considered a security failure rather than a degraded mode. Operators who truly need an unrestricted socket thread must disable seccomp globally—not selectively skip installation.

   4.3. The in-process HTTP server implementation is started, after which optional wall-clock and CPU timestamps supplied on the command line are converted into startup metrics for orchestration layers that attribute process creation versus on-CPU time.

   4.4. The main loop repeatedly asks the server for a batch of ready requests. Each batch element is processed to completion before the next poll. Normal completion produces an HTTP response that is handed back to the server for write scheduling. A dedicated shutdown error from the polling layer flushes partial outgoing writes and ends the loop cleanly—this is how the monitor signals that the guest has finished and the socket should drain.

   4.5. Transient errors while polling for work are logged and the loop continues; the intent is resilience against ephemeral socket states without tearing down unrelated connections.

   4.6. For every successfully parsed request that maps to monitor work, the thread records a monotonic timestamp at the start of handling. After the synchronous round trip completes successfully, that timestamp seeds latency histograms for a small set of expensive operations (full and differential snapshot creation, snapshot load, pause, resume). The stopwatch spans the full user-visible duration from the beginning of API handling until the monitor acknowledges success, not merely time spent inside the guest.

   4.7. Deprecation metadata attached during parsing is logged as a warning and may mark the HTTP response as deprecated so transitional clients can detect legacy behavior without breaking strict parsers.

5. **Request routing and parsing**

   5.1. Each HTTP exchange is classified by verb, first path segment, optional further segments, and presence or absence of a body. The absolute path string is logged in a human-readable description built for operators; for CPU template uploads the raw template is omitted at default log levels and a hint points to verbose logging, because templates can be large and sensitive.

   5.2. Path decomposition trims leading slashes, splits on slash boundaries, and inspects the first token as the primary resource selector. Nested resources encode identifiers in subsequent segments (block devices, network interfaces, snapshot sub-operations, balloon statistics versus configuration, metadata document versus metadata network settings).

   5.3. Method constraints are enforced structurally: GET must not carry a body; PUT and PATCH without bodies are rejected with explicit messages; unknown verb-and-path combinations fall through to a structured “invalid route” style fault. Embedded identifiers must be non-empty and limited to alphanumeric characters and underscores so names remain safe for orchestrators and logs.

   5.4. JSON bodies are deserialized with strict schemas; many resources reject unknown fields to catch typos early. Handlers are thin: they either build a strongly typed monitor action or return a parsing error that never reaches the guest.

   5.5. Imperative operations are grouped under a single resource: the body discriminates among flushing metrics, starting the microVM after configuration, and—on x86 only—a legacy guest keyboard interrupt sequence. AArch64 rejects the keyboard sequence at parse time with a clear client error because the underlying signal has no portable meaning there.

   5.6. Successful responses use either 204 without a body when no payload exists, or 200 with JSON when data is returned. The monitor’s typed success variants drive which branch fires—empty success, structured configuration objects, metadata values, version strings, or full VM configuration snapshots for inspection endpoints.

6. **Error taxonomy from socket to client**

   6.1. **Parse-time faults** cover invalid methods, unsupported routes, malformed JSON, schema violations, empty or invalid resource identifiers, and verb/body mismatches. These are converted to HTTP 400 responses whose JSON body carries a single fault string; they do not imply the guest misbehaved.

   6.2. **Transport-level faults** include oversize bodies rejected by the HTTP stack before application code runs; those responses explain that prior unanswered requests may be dropped, reflecting how the server implementation recovers from pathological clients.

   6.3. **Monitor-side faults** stringify into the same JSON fault shape but use HTTP 400 by default. A specific overflow class tied to metadata size maps to 413 so clients can distinguish capacity limits from generic validation failures.

   6.4. **Success without content** uses 204; **success with data** uses 200 and JSON encoding. Logging on error paths includes the monitor’s string form for operator triage while keeping the wire format stable.

7. **Bridge between socket and monitor**

   7.1. The API thread enqueues a typed monitor action, increments the semaphore-style notifier so the polling wait set becomes readable even if multiple requests arrive before the monitor runs, then blocks on a blocking receive for the matching response. This yields strict per-request FIFO semantics from the client’s perspective: each HTTP transaction maps to exactly one completed monitor operation.

   7.2. During pre-boot, the monitor side may run a simpler blocking loop that only processes API traffic until the machine is constructed—there is no runtime controller yet, but the same channels and notifier preserve identical socket semantics.

   7.3. After boot, a runtime adapter subscribes to the notifier. On readability it drains the counter, attempts a non-blocking dequeue, and dispatches the action through the runtime controller that owns both live resource state and the shared monitor handle.

8. **Interaction with the monitor event loop**

   8.1. Under normal operation the event manager repeatedly waits for readiness, then dispatches subscribers. The API adapter is just another subscriber: its readiness means at least one control message should be available.

   8.2. If the notifier fires but the queue is empty, a warning is logged. That situation should be rare and points to coordination bugs or races between signals and queue state.

   8.3. Guest shutdown is observed by polling the monitor for an exit classification after each event-manager iteration. Clean success breaks the outer loop; non-clean codes surface as errors to the top-level binary; “not yet finished” continues polling.

   8.4. When the outer loop ends, the shutdown switch is written so the HTTP thread exits its accept loop and joins, ensuring log ordering and socket cleanup before process exit.

9. **Pause, resume, and signal-like actions**

   9.1. Pause is delivered as an ordinary synchronous action across the channels. The adapter handles the notifier read like any other request, but after servicing a pause it deliberately enters a nested loop that performs only blocking receives on the API queue until a matching resume arrives. While inside that nested loop the adapter never returns to the event manager.

   9.2. That nesting is the core quiescence guarantee: because the adapter does not yield back to epoll, other subscribers—including the metrics timer—do not run. Device emulation and timer-driven housekeeping on the monitor thread are effectively frozen at the control-plane’s pause boundary, which aligns with snapshot and diagnostic expectations: the thread that drives devices is not interleaving unrelated work while the guest is supposed to be stopped.

   9.3. Resume breaks the nested loop and restores normal epoll-driven processing. Spurious wakeups or additional pause/resume pairs during the nested phase are still handled sequentially because the loop processes every message until the resume marker appears.

   9.4. Guest-facing “signals” exposed through the imperative action resource are not Unix signals—they are monitor operations with architecture-specific backing. The x86-only keyboard interrupt path is rejected early on AArch64 so clients see a deterministic parse error instead of a runtime failure deep inside the monitor.

10. **Periodic metrics cadence**

    10.1. Metrics use a monotonic periodic timer armed with a fixed interval on the order of one minute. When started, the helper immediately writes the global metrics blob once—this captures startup-only counters before the first full period elapses.

    10.2. Each subsequent timer expiration acknowledges the tick at the kernel interface and writes metrics again. Failures increment a missed counter and log an error, but the periodic arm stays active so transient sink problems do not permanently silence telemetry.

    10.3. The timer subscriber registers with the same event manager as devices and the API bridge, so metric cadence competes fairly for turns with emulation work except during API-driven pause nesting, when the adapter intentionally starves the entire loop.

11. **Seccomp embedding and thread maps**

    11.1. Default advanced policies are produced at build time from architecture- and target-specific JSON descriptions compiled into serialized BPF bytes. Those bytes are embedded into the binary and deserialized at runtime into a map from string role labels to programs.

    11.2. Debug builds may substitute an empty policy for every role when no suitable default exists for the toolchain or target, so developers are not blocked; release builds use the full policy when available.

    11.3. Operator-supplied maps must deserialize with the same schema validator as defaults and pass the same role-name checks: exactly three roles—socket API, monitor, and per-vCPU—with no additional keys. The check partitions keys into allowed and disallowed buckets; extras are reported as a consolidated error listing the unexpected labels, and omissions are reported as missing roles.

    11.4. Installation happens per thread at the point where that thread’s workload is known: the socket thread installs the API program before listening; monitor and vCPU threads install their respective programs during monitor construction according to the shared map handed down from the entry point.

    11.5. When the API is compiled out at the process level, the API entry is removed from the map before building the guest so only monitor and vCPU programs remain—there is no socket thread to constrain.

12. **No-API mode**

    12.1. Selecting headless operation requires a JSON configuration on the command line and skips creating the socket thread entirely. Seccomp maps are filtered to drop the API program; the monitor and vCPU policies still apply.

    12.2. The event manager is created, the periodic metrics subscriber is registered, and the guest is built and booted directly from the JSON document with the same metadata and size limits as API mode.

    12.3. Metrics are started with the same fixed period, then the process enters the same outer polling loop used after boot in API mode, polling for shutdown classification after each dispatch round. There is no kill switch because there is no companion thread—termination is entirely guest-driven.

    12.4. This mode suits fixed appliances, CI scenarios, and minimal attack surface deployments where dynamic reconfiguration is unnecessary.

13. **End-to-end control plane (API mode)**

```
  External clients (CLI, automation)
              |
              |  HTTP/1.1 over Unix socket, keep-alive
              v
 +-----------------------------------------------+
 |  Socket thread                                |
 |  - payload cap enforced before parse        |
 |  - BPF profile for JSON + socket syscalls   |
 |  - parse -> enqueue action -> wake counter  |
 |  - block on response channel -> HTTP mapping  |
 +-------------------------+---------------------+
                           |
           semaphore counter + bounded channels
                           |
                           v
 +-----------------------------------------------+
 |  Monitor thread (shared epoll dispatcher)     |
 |  - devices, vCPUs, timers                     |
 |  - periodic metrics timerfd                   |
 |  - API adapter: drain action, run controller  |
 |    * normal: return to epoll                  |
 |    * pause: nested blocking loop until resume |
 +-----------------------------------------------+
                           |
                           v
              shutdown -> write kill switch
                           |
                           v
 +-----------------------------------------------+
 |  Socket thread observes kill, flushes, exits  |
 +-----------------------------------------------+
```

14. **Pause versus steady-state scheduling**

```
  Steady state                    Pause nesting
  -------------                   -------------

  epoll wait                      epoll wait
      |                               |
      +-- timer tick --> metrics      +-- (adapter not returned)
      +-- device ready --> work             |
      +-- API notify --> handle             v
              |                    blocking recv loop
              |                    (metrics & devices wait)
              v                               |
          optional nested <---------------------+
          pause branch
```

15. **Operational and security notes**

    15.1. Binding the Unix socket fails with a clear error when the address is already in use—operators should treat stale socket files as infrastructure problems rather than silent successes.

    15.2. Parser faults should never panic the socket thread; they become structured 4xx responses and the listener stays up. Monitor faults become structured HTTP errors and may influence process exit classification when the guest stops abnormally.

    15.3. Seccomp defaults are on; disabling filtering is explicit. Payload caps and default metadata limits are aligned so API and in-guest metadata services see coherent bounds.

    15.4. Latency histograms focus on expensive, user-visible operations so operators can correlate socket-level delays with snapshot and pause/resume activity rather than routine configuration tweaks.
