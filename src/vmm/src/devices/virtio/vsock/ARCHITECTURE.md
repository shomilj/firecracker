# Paravirtual Socket Path Architecture

## Purpose & Boundaries

This subsystem implements the paravirtual socket device that sits between a guest virtual machine and the host process. Its responsibility is to present a standards-aligned virtio vsock device to guest software while translating traffic to and from host-side Unix domain sockets. The design deliberately avoids routing guest traffic through kernel vhost paths on the host; the virtual machine monitor owns the full device model and mediates every packet.

What belongs here includes virtio queue handling for the vsock device, parsing and composing wire-format packets, connection lifecycle for stream-oriented flows, host socket multiplexing, and event-driven scheduling so that guest buffers, host sockets, and internal queues make progress together. What does not belong here includes generic virtio infrastructure elsewhere in the stack, guest operating system socket implementations, and policy decisions about which host services are exposed at which paths—that configuration remains outside this layer.

If this component fails, guest workloads that rely on paravirtual sockets lose connectivity to host services that were reached through the Unix socket bridge. The rest of the virtual machine may continue running, but any automation, agents, or APIs that depended on that channel would stall or error until the device is restored or the workload is reconfigured. Snapshot restore recreates the virtio frontend state and reconnects the backend using saved path information, but live host sockets are not preserved across migration in the same way as pure memory state; operators should expect connection-oriented flows to reset when the surrounding lifecycle events are disruptive.

## Interfaces & Contracts

The device exposes the virtio surface: feature negotiation aligned with a modern virtio revision, a small configuration space that publishes the guest context identifier, and three queues used for inbound packets toward the guest, outbound packets from the guest, and a single-slot event channel used for transport-level notifications. It consumes guest memory only through validated descriptor chains, and it consumes host resources through non-blocking Unix sockets and a nested readiness mechanism described later.

Callers of the abstract channel interface must supply well-formed descriptor chains. The device guarantees that it will not synthesize guest memory references beyond what the chains describe, and it will reject or zero-length complete malformed inbound work rather than leaving queues inconsistent when parsing fails. The backend side of the contract is that any implementation must be able to report whether it has work ready for the guest, must accept outbound packets in order when the virtio transmit path runs, and must surface readiness in a way that composes with the monitor’s central polling loop.

The Unix-oriented backend additionally binds a listening socket on the host filesystem, accepts new host-initiated connections, parses a minimal text command to learn a destination port, and connects outbound Unix streams for guest-initiated requests using a deterministic path pattern derived from the listening path and the destination port. Those behaviors are part of the public contract for deployments that rely on host agents speaking that command protocol.

## Data Flow

Traffic from the guest enters as virtio transmit queue buffers. Each buffer encodes at most one vsock packet: a fixed-size header followed optionally by payload bytes for data operations. The device copies or maps the header into an internal working copy, validates lengths and addressing, then forwards the packet to the backend multiplexer. The multiplexer routes by the pair of ports carried in the header. Connection requests that do not yet have state cause the backend to create a new host stream and attach a per-connection state machine. Established flows forward data and control operations into that state machine.

Traffic toward the guest is assembled when the receive path runs. The backend first decides which logical source should produce the next packet: a forced reset, or a particular live connection. A bounded queue holds hints for that scheduling; when it cannot represent every connection that has something to say, the structure goes out of synchronization and the implementation rebuilds from a walk of active connections, trading a rare linear cost for bounded memory.

For each chosen connection, the state machine fills a guest receive buffer: it may emit handshake packets, credit traffic, payload reads from the host stream straight into guest memory, graceful shutdown notices, or resets. Header fields carry flow-control accounting so peers stay consistent about how much data is in flight. When the guest side has not posted receive buffers, work stays pending inside the backend until a later receive queue event drains it.

The following diagram summarizes the major data paths without naming implementation artifacts.

```
  +------------------+     virtio TX      +-------------------+
  | Guest vsock      | -----------------> | Device: parse TX  |
  | driver + memory  |                    | chains -> packet |
  +------------------+                    +---------+---------+
                                                    |
                                                    v
                                         +----------+----------+
                                         | Backend multiplexer  |
                                         | route + connections  |
                                         +----------+----------+
                                                    |
                           host Unix streams <------+ +------> guest memory
                                                    |
  +------------------+     virtio RX      +---------v---------+
  | Guest buffers    | <----------------- | Device: fill RX   |
  | posted by driver |                    | chains from pkt   |
  +------------------+                    +-------------------+
```

Before the diagram, the intent is to show that guest-originated bytes always pass through virtio transmit descriptors, then through the backend, while host-originated bytes move from Unix streams into virtio receive descriptors. After the diagram, it is important to add that the event queue is a narrow side channel used for transport reset signaling to the driver rather than for bulk payload, so most application bytes never touch that queue.

## Control Flow

Execution is event-driven. A registration phase wires an activation notification separately from steady-state sources so that backend listeners are not spun before the virtio device is fully live. After activation, the steady-state sources are the receive queue kick, the transmit queue kick, the event queue kick, and readiness on the backend’s aggregate polling handle.

When the transmit queue signals, the device drains guest-provided chains, sends each parsed packet to the backend, completes successfully processed chains, and then opportunistically runs the receive side if the backend indicates pending inbound work. When the receive queue signals, the device only moves data guest-ward if the backend still has pending work; otherwise the notification is consumed without touching queues, avoiding useless scanning.

When the backend signals readiness, the device first delivers the readiness to the backend implementation, which may flush partially written host paths or mark inbound data as fetchable. Then the device retries transmit processing so that a previously stalled queue can advance, and finally it tries to fill receive buffers if inbound packets are waiting.

The control path branches on packet type and connection state inside the backend. Unknown connection keys with only certain operation types create new host sockets; others elicit a reset reply. Duplicate shutdown handling merges flags over time. Kill timers move connections toward forced teardown if handshakes or graceful shutdowns linger too long.

```
                    +-------------+
                    | Activation |
                    +------+------+
                           |
           +---------------+---------------+
           |                               |
           v                               v
    +-------------+                 +---------------+
    | Queue kicks |                 | Backend ready |
    +------+------+                 +-------+-------+
           |                                 |
           v                                 v
    +-------------+                 +---------------+
    | TX process  |                 | Notify backend|
    +------+------+                 +-------+-------+
           |                                 |
           |         +----------------+        |
           +-------->| RX opportunistic|<-------+
                     +--------+-------+
                              |
                              v
                     +--------+--------+
                     | IRQ if used     |
                     | ring advanced   |
                     +-----------------+
```

The prose before this figure notes that activation is the gate for all real work, and that transmit and backend notifications both funnel into optional receive work and interrupt signaling. The prose after emphasizes that interrupt coalescing is not described here at the virtio layer; the device raises an interrupt when it has moved the used ring forward for either path.

## State & Lifecycle

The virtio device holds queue state, working packet buffers, feature bits, and the activation marker that ties guest memory to queue bases. The backend holds a pool of connections keyed by host-local and guest-peer port pairs, a listener for host-initiated attachments, internal hints queues for inbound scheduling and timed teardown, a nested readiness set for all host file descriptors, and bookkeeping for which local ports are in use.

Connections move through distinct phases: awaiting peer confirmation, awaiting host confirmation, established data transfer, graceful shutdown with paired directions, and forced termination. Timers arm for connection requests that are not answered in time and for shutdown sequences that need a bounded completion window. When timers fire, entries are collected through a small queue that may fall out of synchronization under pressure; the implementation rebuilds it by scanning connections when needed.

At startup, the listening host path is bound and the multiplexer registers for accepts. Guest initiation creates streams connected to per-port paths. Host initiation accepts a connection, switches to reading a textual port selection, then promotes the stream into a tracked connection with an allocated local port in a high range. Shutdown removes listeners and frees ports.

Persistence captures the guest identifier, virtio queue snapshots, and the host socket path for the listener. Restore rebuilds queues and reconnects the listener; existing host connections are not magically resurrected, so application-level sessions should assume a reset around snapshot operations.

## Failure Modes

Transmit processing stops consuming new chains if the backend reports that it cannot accept further work at the moment. The device leaves the current chain uncompleted so a later kick retries. This avoids dropping guest data while still allowing the virtio path to remain consistent.

Receive processing that cannot fill a buffer rewinds the queue iterator so the buffer remains available. Parse errors on receive chains complete with zero used bytes where appropriate and log problems, preventing stuck rings at the cost of dropping a buffer’s worth of progress for that step.

Malformed guest headers, impossible lengths, or unsupported packet types are handled at the edges: unsupported types trigger reset responses; unknown destinations with non-handshake operations trigger resets; cross-context identifiers that are not the host role are dropped rather than forwarded.

Host socket errors during established data transfer can cause immediate termination of a connection. Reset packets are preferred over silent hangs. The nested readiness dispatcher logs failures when waiting for events fails, incrementing error counters without crashing the whole device.

A documented risk remains if the guest stops consuming receive buffers while the host keeps sending: the backend could remain ready while unable to drain host-side input indefinitely. The code carries a reminder that ideal behavior would unregister read interest in that scenario. Operators should monitor resource usage on the host side of the sockets if guests misbehave.

Silent corruption is most likely if guest memory were accessed without the bounds checks that descriptor parsing provides; the design treats guest-supplied pointers as hostile and validates lengths against maximum packet sizes and memory limits enforced by the virtio helpers.

## Operational Characteristics

Work is proportional to queued virtio descriptors, the number of active connections, and the depth of internal hint queues. Rebuilding a full receive hint list or kill schedule is linear in the number of connections and is triggered when bounded queues overflow their synchronization guarantees, so very chatty workloads with many simultaneous connections may pay occasional extra scans.

Each connection carries a fixed maximum transmit-side staging buffer for guest-to-host data when the host socket blocks, plus protocol-driven credit windows for host-to-guest data. The multiplexer caps the number of simultaneous established flows; attempts beyond that refuse new host accepts cheaply and fail guest-side opens with connection-level errors expressed as resets.

Observability is primarily through aggregated counters: activation and configuration failures, queue event failures, backend dispatch failures, connection churn, byte and packet totals, flush and read failures, and resynchronizations of the termination schedule queue. These metrics flush in structured form when the monitor’s metrics writer runs. Logging captures warnings for unexpected readiness types, parse issues, and dropped resets when hint queues are full.

## Design Rationale

The architecture exists to give guests a standard paravirtual socket without forcing the host kernel’s vhost vsock into the data path. That choice keeps the virtual machine monitor in control of scheduling, simplifies reasoning about non-blocking behavior in userspace, and aligns with a minimal device model philosophy.

Unix domain sockets on the host provide a familiar, permission-friendly rendezvous point for services co-located with the process. Mapping guest port numbers to filesystem paths makes integration predictable for tooling, at the expense of requiring disciplined host layout and trusting the host side of the connection.

Nested readiness polling consolidates many host file descriptors behind one handle for the outer event loop, reducing registration churn at the cost of an extra dispatch step when readiness fires. Credit-based flow control mirrors the vsock specification so peers do not overrun each other’s buffers; proactive credit updates reduce reliance on explicit credit requests during steady streaming.

Bounded hint queues trade perfect scheduling for fixed memory. When queues desynchronize, correctness is preserved by scanning connection sets, favoring stable behavior under overload over strictly optimal latency.

The activation indirection prevents backend listeners from firing in a tight loop before virtio queues are wired to guest memory, which was an explicit product decision called out in the implementation comments. That small startup dance avoids pathological event storms during construction and restore.

---

The following state-oriented diagram illustrates connection progression at a conceptual level. It is not exhaustive of every edge case, but it captures the handshake and teardown intent.

```
        [ Created ]
             |
      +------+------+
      |             |
      v             v
 [ Host-led     [ Guest-led
   handshake ]    handshake ]
      |             |
      +------+------+
             |
             v
      [ Established ]
             |
     +-------+-------+
     |               |
     v               v
[ Graceful        [ Forced
  shutdown ]        reset ]
     |               |
     v               v
     [ Removed from active set ]
```

Before the diagram, the point is that both directions of initiation converge on a single established phase where payload and credit packets matter most. After the diagram, note that graceful shutdown can still pass through intermediate substates where one direction has stopped while the other drains, and the kill timer ensures those substates cannot last indefinitely without a forced reset.

Together, these mechanisms implement a complete paravirtual socket path: virtio queues and memory safety at the boundary, protocol-faithful connection machines in the middle, and Unix sockets on the host with explicit backpressure, bounded queues, and clear security separation between guest-supplied identifiers and host filesystem layout.
