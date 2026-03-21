# GDB Remote Debugging Subsystem Architecture

## Purpose & Boundaries

This subsystem implements the host side of a remote GDB debugging session for a hardware-virtualized guest. Its responsibility is to translate between the GDB remote serial protocol (as implemented by an external stub library) and the virtual machine monitor’s execution model: KVM-backed virtual CPUs, guest memory, and the existing pause and resume machinery used elsewhere in the monitor. It configures the hypervisor for guest debugging, surfaces stop reasons when a virtual CPU hits a debug event, and answers debugger requests for registers, memory, breakpoints, single stepping, and multithreaded views of the guest.

What lies explicitly outside this boundary includes the virtio vsock device implementation, network or socket policy on the host, and any client-side GDB configuration. The debugger connection endpoint is presented as a host-local stream socket path supplied at boot; in typical deployments that path is the far end of a virtio-vsock bridge so that a GDB process running in the guest (or another namespaced environment) can reach the stub without traditional network routing. The subsystem does not implement guest operating system policies, symbol resolution, or source-level mapping; those remain the client debugger’s concern.

If this component failed to start or crashed during operation, interactive kernel debugging through GDB would be unavailable, but ordinary virtual machine boot and execution would continue unless the failure path intentionally tears down the whole machine—which the design does only after the debugging session ends. If the component mishandled guest memory or register updates, the guest could corrupt itself or become inconsistent with the debugger’s view; correctness therefore depends on disciplined translation of virtual addresses and on only mutating state when the relevant virtual CPU is stopped under the subsystem’s own coordination.

## Interfaces & Contracts

The subsystem consumes several stable contracts from the surrounding monitor. It holds a mutex-protected handle to the top-level virtual machine object so it can reach guest memory and per-virtual-CPU control handles. Each virtual CPU exposes an operating-system-level handle into KVM for register access, single instruction stepping flags, hardware breakpoint programming, and address translation where the kernel provides it. A unidirectional queue carries compact notifications from virtual CPU threads to the debugging thread: each notification identifies which virtual CPU index encountered a debug exit from KVM.

Toward the debugger, the subsystem exposes a stream-oriented connection after binding a configurable local path and accepting exactly one inbound connection. That design assumes a single controlling GDB session. The protocol layer expects multithreaded stop and resume semantics: the debugger identifies threads using small positive integers that the subsystem maps deterministically to virtual CPU indices. Callers of the boot-time configuration must supply a valid socket path and enable the feature consistently with the rest of the build; the monitor wires the notification channel before virtual CPUs begin running so that early debug exits are not lost.

Invariants maintained by the subsystem include the following. Hardware breakpoint slots are bounded by the architecture and KVM (for example, four hardware addresses on the common 64-bit x86 configuration). Software breakpoints are tracked by guest physical address after translation, with original instruction bytes saved so removal can restore the guest image. Updates to KVM guest debug state are applied only while the affected virtual CPU is considered paused in the subsystem’s own bookkeeping, avoiding races with live execution except where the hypervisor API explicitly allows it. The monitor lock must be taken for operations that touch shared guest state; poisoning or deadlock at that lock surfaces as fatal errors to the protocol layer rather than silent continuation.

## Data Flow

Debugger traffic arrives as a byte stream of remote protocol packets. The blocking protocol engine reads commands and dispatches them to adapter callbacks that read or write register images, translate guest virtual addresses to physical addresses, and copy bytes to or from the guest memory object in page-aware chunks. Responses are framed and written back on the same stream. Independently, virtual CPU threads push integer identifiers onto the notification queue when KVM returns from execution because of a debug-related exit.

Guest virtual addresses for memory access pass through architecture-specific translation. On one architecture, translation is delegated to a single hypervisor ioctl per lookup. On another, when paging is not yet established, the implementation may treat the address as already physical; otherwise it walks page tables using values read from system registers and guest memory to derive a guest physical address. Translated physical addresses are then used with the monitor’s guest memory abstraction, respecting page boundaries so that reads and writes never cross a page in a single guest memory operation without iteration.

Stop classification inspects the stopped virtual CPU’s instruction pointer, compares it against the recorded software breakpoint set (keyed by physical address), the hardware breakpoint list, and the configured kernel entry address used for the initial stop. When the stop does not correspond to any breakpoint owned by this subsystem, the flow may treat it as a guest-internal breakpoint, optionally asking the hypervisor to reinject the faulting instruction’s trap semantics and immediately resuming that virtual CPU without notifying the remote debugger—so that self-tests inside the guest kernel can proceed.

The following diagram summarizes major data paths without naming internal modules.

```
  +------------------+          stream I/O           +------------------------+
  | GDB client       | <-------------------------> | Host local socket      |
  | (guest or host)  |                               | (often paired with     |
  +------------------+                               |  virtio-vsock bridge)  |
                                                           |
                                                           v
                                                    +------------------+
                                                    | Protocol engine  |
                                                    | (decode/encode)  |
                                                    +--------+---------+
                                                             |
                         register / memory / control        |
                                                             v
                                                    +------------------+
                                                    | Debug target     |
                                                    | adapter          |
                                                    +--+-----------+---+
                                                       |           |
                                    KVM ioctl /        |           | guest memory
                                    register IO        |           | read/write
                                                       v           v
                                              +------------+  +-----------+
                                              | Per-vCPU   |  | Guest RAM |
                                              | KVM state  |  | mapping   |
                                              +------------+  +-----------+

  +------------------+     integer index        +------------------+
  | vCPU thread      | --- notification ----> | Debug thread     |
  | (KVM run loop)   |                          | (poll + wait)  |
  +------------------+                          +------------------+
```

Before the diagram, it is useful to state plainly that two independent sources of input drive the debug thread: asynchronous notifications from virtual CPUs and synchronous bytes from the debugger. After the diagram, the important consequence is that ordering between those sources is merged in a single polling loop so that neither starvation nor protocol deadlock is introduced by solely blocking on the socket.

## Control Flow

Startup begins after the virtual machine image is prepared and the entry address for the guest kernel is known. The subsystem programs an initial hardware breakpoint on the boot virtual CPU at that entry address so the guest is already stopped at a well-defined place when the debugger attaches—this also allows setting breakpoints before guest code runs. Other virtual CPUs are placed in a debug-enabled configuration without pending addresses so they can run or idle without immediately tripping stops.

A dedicated thread binds the configured local path, blocks on accept, and spawns the long-lived handler that owns the protocol engine and the debug target state. The parent returns so boot can continue; the debugging thread first waits on the notification channel for the initial stop that corresponds to hitting the entry breakpoint, ensuring protocol processing does not start before the guest has reached the expected pre-kernel halt.

During steady operation, a custom wait primitive alternates between non-blocking reads on the notification queue and peek-based reads on the debugger connection. When a virtual CPU reports a debug exit, the adapter records which thread is paused, derives a stop reason, and may either return that reason to the protocol engine or transparently reinject and resume as described earlier. When data is available from the debugger, the engine consumes it. User interrupt handling maps to pausing a designated primary virtual CPU and reporting an interrupt signal class stop so the human operator can break a running guest.

Resume and stepping are staged: the protocol layer first records per-thread intent (continue versus single instruction), then a batched resume operation pushes updated KVM debug parameters to every virtual CPU that is paused, clears the central notion of which thread was last stopped, and issues resume events through the same channels the monitor uses for ordinary pause and resume—each resume waits for an acknowledgment so the subsystem knows the virtual CPU left the paused state.

The next diagram sketches the high-level decision tree when a notification arrives.

```
            KVM returns to userspace
            with a debug-class exit
                        |
                        v
              +-------------------+
              | Map notifier id   |
              | to debugger       |
              | thread identity   |
              +---------+---------+
                        |
                        v
              +-------------------+
              | Classify stop     |
              | (step / sw / hw / |
              |  entry / other)   |
              +---------+---------+
                        |
           +------------+------------+
           |                         |
           v                         v
    [Owned breakpoint]        [Foreign trap]
    notify GDB               reinject if supported,
    with reason               else synthesize;
                             resume vCPU
```

Before this figure, recall that classification depends on both architectural state (instruction pointer) and the subsystem’s own breakpoint tables. After it, note that the foreign-trap branch exists specifically to avoid stealing breakpoints that belong to guest self-tests or firmware.

## State & Lifecycle

Owned state splits into three layers: per-virtual-CPU shadow flags (paused versus running, single-step arm), global breakpoint tables (hardware list with a fixed upper bound, software map from translated addresses to saved instruction bytes), and bookkeeping for which debugger thread identifier corresponds to the last stopped virtual CPU so that commands omitting an explicit thread still target sensible state.

Initialization sets the first virtual CPU as paused in shadow state to reflect the entry breakpoint fiction, seeds the last-stopped identifier to that CPU, and prepares empty breakpoint containers. Runtime mutation occurs on debugger commands (insert or remove breakpoints, read or write memory or registers) and on virtual CPU notifications. Teardown is triggered when the remote disconnects or the protocol engine observes target exit; the handler then instructs the monitor to stop cleanly, which tears down the machine from the perspective of the rest of the system.

Recovery from partial failures is not expansive: channel errors between virtual CPUs and the debug thread are treated as fatal to the session, and mutex poisoning when touching the monitor is also fatal. There is no automatic retry of breakpoint programming beyond what the remote operator initiates.

A small state machine for virtual CPU visibility from the debugger’s perspective can be drawn as follows.

```
       +----------+
       | running  |
       +----+-----+
            | debug exit or
            | explicit pause
            v
       +----------+       resume batch       +----------+
       | paused   | -----------------------> | running  |
       +----------+                           +----------+
            ^                                     |
            |         single-step may             |
            +----------- re-enter paused --------+
```

Prose before: each virtual CPU is either executing guest code under KVM or held out of the run loop with emulation paused while the debugger holds the system. Prose after: transitions back to running always go through the consolidated resume path so hardware debug registers and single-step flags stay coherent across the population of CPUs.

## Failure Modes

Explicit failures include inability to bind or accept the local socket, failure to spawn the service thread, errors when sending pause or resume requests to a virtual CPU, KVM errors when applying guest debug state, translation failures for guest virtual addresses, and guest memory faults. Many of these surface as non-fatal protocol errors so the debugger can retry or adjust commands; mutex failure and a few internal fault classes are treated as fatal because continuing would risk inconsistent machine state.

Silent corruption risks stem from incorrect address translation (reading or writing the wrong physical page), mishandling of software breakpoint patching if the underlying bytes change unexpectedly, and divergent views between virtual CPUs if breakpoints were programmed while not all relevant CPUs were paused—mitigations include applying debug register updates only on paused CPUs and centralizing resume.

When the guest hits a trap that is not tracked as a subsystem breakpoint, ignoring it would hang the guest; the reinject-and-resume path attempts to hand control back to guest handlers. On one architecture reinjection is fully supported; on another the hypervisor lacks an equivalent injection path, so the no-operation reinject path implies foreign traps may not be maskable the same way—operators should be aware that self-inflicted traps during early boot may behave differently across architectures.

## Operational Characteristics

Resource use centers on one long-lived thread plus the existing virtual CPU threads; memory overhead is dominated by breakpoint maps and modest buffering for protocol I/O. The notification queue depth is not unbounded in the custom wait loop, so bursts of debug events rely on virtual CPUs blocking until the debug thread consumes prior messages—typically acceptable because only one virtual CPU should hit a breakpoint at a time in many debugging scenarios, though concurrent stops on multiple CPUs can serialize behind the single handler.

Scaling limits include the hardware breakpoint cap, the single connected debugger session, and the sequential processing of stop reasons. Observability is mostly indirect: logging traces notable decisions such as reinjection, pause and resume anomalies, and classification shortcuts when instruction pointer reads fail (where a software breakpoint stop is synthesized as a safe fallback). There is no dedicated metrics layer inside this folder; operators rely on general monitor logging and GDB’s own packet traces when diagnosing protocol issues.

Concurrency is asymmetric: virtual CPUs run in parallel, but the debug thread mutates shared breakpoint state and issues pause and resume operations sequentially. Locking the monitor for memory and translation operations can contend with other management actions; heavy debugger memory scanning could interact visibly with API latency on the control plane.

## Design Rationale

The architecture solves the problem of making a minimal virtual machine monitor debuggable without embedding a full in-guest agent. By leaning on KVM guest debugging and a standard remote protocol, the monitor reuses familiar tooling while keeping the trusted computing base smaller than an in-guest stub would require. Using a host-local socket endpoint integrates naturally with virtio-vsock-backed workflows: the vsock device connects guest-initiated streams to host Unix sockets, so exposing the stub on such a path yields “GDB over vsock” operationally even though this package itself speaks only POSIX sockets.

Multithreaded GDB support maps each virtual CPU to a distinct thread identifier so operators can inspect and step per CPU, but the implementation favors an “all resume” model when continuing after a stop, updating every paused CPU’s debug state before running—reducing stray single-step flags and hardware register drift across CPUs. The initial entry breakpoint trades a small amount of startup complexity for a deterministic first stop before guest code executes.

Software breakpoints are implemented by patching guest memory with an architecture-specific trap instruction and saving overwritten bytes keyed by physical address so the same location remains consistent even if multiple virtual CPUs share address space. Implicit software breakpoint assistance from the library is disabled so the monitor retains full control over reinjection and foreign traps.

Tradeoffs accepted include reliance on a correct translation path for memory and breakpoints (especially on the architecture that walks page tables manually), the use of a single primary virtual CPU for user interrupt handling rather than uniformly stopping every CPU, and shutdown of the entire virtual machine when the debugger disconnects—appropriate for a development-oriented feature but potentially surprising if a management stack expected the guest to keep running unattended after a debug session.

Taken together, the design prioritizes protocol fidelity, predictable stop and resume behavior, and tight coupling to KVM’s debugging facilities, while deferring transport policy to the surrounding deployment (direct host access, forwarded file descriptors, or virtio vsock bridging) and deferring symbolic debugging entirely to the client.
