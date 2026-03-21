# Firecracker Binary Crate Architecture

This document describes the architecture of the Firecracker binary crate: how the runnable process is assembled from workspace components, how the control-plane HTTP server relates to the virtual machine monitor, and how execution is driven across threads and event loops. The focus is on responsibilities, contracts, and runtime behavior rather than implementation minutiae.

## Purpose & Boundaries

The binary crate is the outer shell of the Firecracker process. Its responsibility is to turn operating-system resources and command-line intent into a running micro virtual machine that can be managed through a stable control interface. Concretely, it owns process-level concerns: parsing startup parameters, configuring logging and metrics sinks, selecting and applying syscall restriction policies per execution context, choosing between interactive API-driven configuration and one-shot JSON configuration, starting the HTTP listener when enabled, and bridging remote control operations into the monitor library that actually constructs devices, vCPUs, and guest memory.

What this layer does not do is implement the emulated hardware, the KVM interactions, the snapshot format, or the guest boot protocol. Those belong to the virtual machine monitor library and its dependencies. If the binary layer fails early—for example because arguments are unusable, the control socket cannot be bound, or syscall filters cannot be installed—the process exits before any guest exists, and the larger system simply sees a failed launch. If the binary layer starts successfully but the monitor later stops with an error, the binary propagates that outcome as the process exit status so orchestrators can distinguish configuration mistakes from runtime failures. The binary is therefore a thin but critical control plane: it is not the physics of virtualization, yet it decides whether safe, observable, and reachable virtualization can begin at all.

## Interfaces & Contracts

**Outward-facing interfaces.** The primary external contract is the command line, which selects socket paths, identity strings, logging and metrics destinations, optional JSON configuration and optional metadata seeding, limits on HTTP body size and on metadata store size, timing inputs for startup observability, and modes that disable the API or syscall filtering for advanced scenarios. A second outward interface is the HTTP surface exposed over a Unix domain socket when the API is enabled: REST-shaped routes map to administrative actions such as configuring drives, network interfaces, machine parameters, snapshots, and lifecycle operations. Help, version printing, and snapshot format introspection are also exposed as early exit paths that do not start a long-lived server.

**Inward dependencies.** The binary integrates the workspace’s shared argument parser and validators, the core monitor library (including its logging and metrics facades, resource descriptions, RPC-style action model, builder and persistence components), a compact HTTP server implementation suitable for Unix sockets, and an event subscription framework used to multiplex I/O readiness across subscribers inside the monitor thread. Seccomp filter bytes are prepared at build time and deserialized at runtime into a map keyed by logical thread roles. Utility bindings provide readiness-based I/O helpers and timers.

**Contracts and invariants.** Callers must supply a valid instance identifier and consistent flag combinations; for example, running without the API requires a JSON configuration path. The monitor side expects synchronous request handling semantics for each administrative action issued from the API thread: the HTTP handler sends a work item, notifies the monitor thread, and blocks until a single response is returned. During a guest pause, the design temporarily switches the monitor-side handler to a mode that processes only control traffic so device emulation does not advance while the guest is frozen; callers relying on timely metric flushes should understand that this mode intentionally starves other event sources until resume. Seccomp maps must contain exactly the expected set of thread categories; missing or unknown categories are rejected at startup.

The following diagram situates the binary between operators, local I/O, and the monitor library.

```
                    +------------------+
                    |  Operator / CLI  |
                    +--------+---------+
                             |
              command line + optional JSON files
                             v
                    +------------------+
                    |  Binary process  |
                    |  (startup &      |
                    |   mode select)   |
                    +--+-----------+---+
                       |           |
           Unix API    |           |  build-from-JSON
           socket      |           |  (optional)
                       v           v
              +----------------+  +----------------+
              | HTTP server    |  | Direct JSON    |
              | thread         |  | bootstrap      |
              +--------+-------+  +--------+-------+
                       |                   |
                       |  sync actions     |
                       v                   v
              +------------------------------------+
              |  Monitor library (VMM core)      |
              |  devices, vCPUs, KVM, snapshots  |
              +------------------------------------+
```

Before the diagram, note that operators interact only through the CLI and the socket API; the binary never exposes a networked TCP listener by default. After the diagram, the takeaway is that whether configuration arrives over HTTP or from a file, the same monitor library performs the substantive work, preserving one semantic core behind two ingress shapes.

## Data Flow

**Ingress.** Data enters as Unicode command-line arguments, optional JSON strings read from disk for bulk configuration and optional metadata seeding, optional fifo or file paths for logs and metrics, and HTTP requests on the Unix socket carrying JSON bodies for fine-grained changes. Host page size is queried once early because downstream memory accounting depends on it.

**Transformation.** HTTP requests are parsed into abstract administrative actions. Parsing validates HTTP method and path combinations, rejects incoherent bodies, normalizes resource identifiers, and may attach deprecation notices without failing the call. Successful parses yield an owned action envelope that travels over a thread link to the monitor. The monitor executes the action and returns an outcome value that is translated into HTTP status codes and JSON bodies: empty successes map to no-content responses, structured data map to JSON payloads, and semantic errors map to client errors with a fault object. Snapshot and pause or resume style operations additionally feed latency metrics when they succeed.

**Egress.** Responses leave over the same HTTP connection for API mode. Logs and metrics leave via configured sinks managed inside the monitor’s logging subsystem; the binary also wires a periodic timer-driven flush so counters and histograms are written at a steady cadence once the virtual machine is live. Process timing inputs optionally enrich startup observability by recording wall and CPU time at the API thread.

A compact data-flow diagram follows.

```
CLI / env assumptions
        |
        v
+---------------------+
| Validate identity,  |
| configure logs &    |
| metrics, load       |
| syscall filter map  |
+---------------------+
        |
        +--------> [API path] -----> HTTP parse -----> action envelope
        |                                              |
        |                                              v
        |                                     channel to monitor
        |                                              |
        +--------> [no-API path] ---> JSON parse ------>+
                                                       |
                                                       v
                                              monitor execution
                                                       |
                                                       v
                                              typed result -> HTTP mapping
                                                       |
                                                       v
                                              client response / process exit
```

Prior to the diagram, recall that metadata size defaults to track the HTTP payload ceiling unless overridden, coupling guest-facing limits with control-plane limits. After the diagram, emphasize that snapshot describe and version printing short-circuit before any channel or monitor loop exists, so they never touch guest state.

## Control Flow

**Triggers.** Execution begins at process start. A small wrapper records failures and maps many error classes to stable exit codes for automation. The critical path initializes logging, queries host page size, installs a panic hook that resets terminal state and attempts to flush metrics, parses arguments, handles help and informational modes, validates the instance identifier, applies logger updates, registers signal handlers, optionally enables architecture-specific speculation mitigations, attempts a non-fatal file descriptor table resize, assembles instance metadata, initializes metrics if requested, builds the syscall filter map, and then chooses between API and non-API execution.

**API path.** The API path constructs synchronization primitives: a semaphore-style notification object for the monitor loop, a nonblocking shutdown trigger for the HTTP server, and channels connecting the API thread and the monitor thread. It binds an HTTP server to the configured socket path, arms a kill switch so the main thread can stop the server cleanly, and spawns a dedicated API thread that applies the API-specific syscall program, starts the HTTP server, reports process timing, and enters a loop of accepting and responding to requests. Meanwhile the main thread creates an event multiplexer, registers a periodic metrics subscriber, and either builds the microVM directly from JSON or drives an interactive pre-boot controller that reads actions from the channel until boot completes.

**Post-boot.** Once the guest is running, an adapter subscriber merges API traffic into the monitor’s event loop: when the notification object signals, the adapter receives the next action, dispatches it to a runtime controller holding live resources and the monitor, and replies synchronously to the API thread. If the action is a pause, the adapter enters a nested loop that blocks on the channel until a matching resume arrives, which intentionally prevents the event multiplexer from running and freezes emulation. When shutdown is clean, the main thread signals the HTTP shutdown object, joins the API thread, and returns success or propagates monitor errors.

**Non-API path.** The non-API path removes the API thread’s syscall entry, builds from mandatory JSON, starts periodic metrics, and runs the multiplexer until the monitor reports a shutdown code. The loop interprets the monitor’s exit hint to distinguish success from error termination.

The next diagram sketches the API control path across threads.

```
[main thread]                          [API thread]
     |                                       |
     | bind socket & spawn -----------------> starts HTTP server
     |                                       |
     | build guest (preboot or JSON)         |  await requests
     |                                       |
     | register adapter with event loop      |
     |                                       |
     | run event loop <----- notify -------- | parse HTTP -> enqueue action
     |        |                              |             |
     |        +--- process action ------------+             |
     |        |   (sync response)            |             |
     |        v                              |             |
     |   guest devices & vCPUs               |             |
     |                                       |
     | shutdown: signal kill switch -------> flush & exit
     | join API thread
```

Explanatory text before the diagram: the notify edge is what couples asynchronous readiness in the monitor thread with synchronous HTTP handling in the API thread. Explanatory text after: this design trades a little cross-thread complexity for a simple guarantee—each HTTP request maps to exactly one completed monitor action before the response is sent, which keeps client semantics predictable.

## State & Lifecycle

**Owned state.** The binary owns no guest memory itself; it owns startup configuration artifacts, the choice of ingress mode, the connected HTTP server handle in API mode, the periodic metrics subscriber state, and the adapter object that binds notification and channels to a runtime controller. The monitor owns the virtual machine object behind a shared exclusive lock that the adapter also holds when dispatching work.

**Initialization.** Initialization orders global logging first, then signal handling, then optional mitigations and fd table sizing, then metrics files, then syscall maps. In API mode, the HTTP server comes up before or in parallel with guest construction depending on the selected bootstrap, but always before the long-running event loop services API-driven actions for a fully built guest.

**Steady state.** Steady state is the event multiplexer running in the main thread with subscribers for metrics timers, the API adapter, and monitor-internal devices. The API thread spends its time in HTTP I/O and blocking waits for monitor replies.

**Shutdown and cleanup.** Shutdown begins when the monitor indicates an exit code. The API path then signals the HTTP server to observe the kill switch, drains or flushes as implemented by the HTTP layer, and joins the API thread. Non-API shutdown is entirely governed by the monitor’s exit hint inside the multiplexer loop.

A simple lifecycle state machine for API mode appears below.

```
        +-------------+
        |   process   |
        |   start     |
        +------+------+
               |
               v
        +-------------+
        |  configure |
        |  & filters |
        +------+------+
               |
         +-----+-----+
         |           |
         v           v
   +----------+  +----------+
   | preboot  |  |  JSON    |
   | channel  |  |  boot    |
   +----+-----+  +----+-----+
        |            |
        +-----+------+
              v
        +-------------+
        |  running    |
        |  (event     |
        |   loop)     |
        +------+------+
               |
               v
        +-------------+
        |  stopped    |
        |  (exit      |
        |   code)     |
        +-------------+
```

Before the state machine, interpret “configure” as the phase where logs, metrics, and syscall policy are fixed. After it, note that the split between preboot and JSON boot reflects product needs: automation often prefers a single file, while interactive orchestrators prefer incremental PUTs.

## Failure Modes

**Explicit failures.** Socket bind failures surface as clear errors when the address is already in use. HTTP server startup failures and microVM build failures propagate as structured errors. Monitor-request errors become HTTP client error responses with a fault message rather than panics. Snapshot format queries fail with readable errors if files cannot be read or versions are unknown.

**Panics and invariant breaks.** Disconnection on the request channel is treated as fatal inside the adapter because the system cannot reconcile API and monitor halves if the link disappears. Seccomp application failure on the API thread aborts that thread by design because running exposed without the intended syscall profile violates safety assumptions. Many helper paths terminate the process immediately for conditions that should be impossible after correct construction—those represent programming errors rather than user mistakes.

**Silent risk areas.** If callers disable syscall filtering globally, misconfiguration or compromise in the API thread has a wider kernel attack surface; that tradeoff is explicit. The fd table resize step tolerates partial failure to avoid hard failing launches on constrained hosts, at the potential cost of slower snapshot restore on large guests. Pause mode stops the multiplexer from servicing timer-driven metric flushes until resume, which can delay observability signals during debugging sessions—this is a liveness tradeoff, not silent corruption.

## Operational Characteristics

**Threads and scheduling.** API mode uses at least two threads: the main thread running the readiness-driven event multiplexer and a dedicated API thread for HTTP. Additional threads belong to the monitor for vCPUs and internal workers; the binary does not schedule those directly but inherits their footprint.

**Resource patterns.** Peak file descriptor usage grows with device count; pre-sizing the descriptor table avoids reallocations during snapshot restore on large configurations. HTTP payload limits protect memory use on the API thread; aligning metadata limits with that ceiling keeps dual policies coherent.

**Observability.** Logging uses the shared logger with instance-scoped identity. Metrics include latency buckets for expensive API operations when successful, plus periodic writes driven by a monotonic timer. Panic hooks attempt to emit metrics and restore terminal modes so consoles are left usable on developer machines.

**Bottlenecks.** The synchronous request path means long-running monitor actions block their originating HTTP connection; snapshot creation and loading are typical examples. The pause path intentionally halts unrelated event processing, which is powerful for inspection but can stall background housekeeping until resume.

## Design Rationale

**Why split the HTTP server onto its own thread.** Keeping HTTP parsing and response serialization off the hot path of device emulation isolates blocking client behavior from guest timekeeping. It also allows a distinct syscall profile for the API thread versus the monitor and vCPU threads, reducing kernel attack surface per role.

**Why channels plus notification objects.** A pure queue would still require the monitor thread to wake promptly; pairing a blocking queue with an event-driven notification integrates API work into the existing readiness model without polling. Synchronous request-reply on channels preserves HTTP semantics where each request expects exactly one response.

**Why JSON bootstrap exists alongside the API.** Large, generated configurations are easier to ship as a single artifact than as a scripted sequence of HTTP calls. Firecracker’s operational modes therefore mirror both interactive orchestration and immutable infrastructure patterns.

**Why optional syscall customization and opt-out.** Enterprise users occasionally need to extend syscall allow lists for plugins or experimental kernels; advanced users can supply custom serialized filters. Opting out entirely is discouraged but available for environments where outer containment replaces kernel filtering.

**Why fd table resizing appears in the binary.** This is a process-wide concern tied to how many event and timer objects register with the multiplexer; addressing it at launch avoids latency cliffs during restore that would be hard to attribute later.

**Workspace integration.** The binary crate composes sibling packages rather than reimplementing them: shared CLI parsing ensures consistency with other tools in the repository, the monitor library centralizes guest semantics, the HTTP stack stays small and auditable, and build-time generation keeps syscall policies versioned with the code that relies on them. The public library surface of this crate is intentionally minimal so most consumers use the executable, while tests and downstream crates can still reuse the HTTP parsing layer when needed.

Together, these choices produce a process that is easy to reason about: one monitor core, one optional API thread, one readiness-driven dispatch loop for emulation and administrative side effects, and explicit shutdown sequencing that keeps logs, metrics, and exit codes aligned with what operators expect from a production VMM control plane.
