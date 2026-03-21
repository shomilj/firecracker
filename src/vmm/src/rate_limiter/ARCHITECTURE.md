# Rate Limiter Architecture

## Purpose & Boundaries

This subsystem implements virtual-machine I/O throttling using paired token-bucket models so that upper layers can cap sustained throughput and burst separately for byte-oriented work and for operation-count-oriented work. It is intended to be embedded wherever guest traffic must be shaped, including block storage paths and network transmit and receive paths. Each embedding typically owns one composite limiter instance; network interfaces often attach one instance per direction, while a block device attaches a single instance shared by that device’s request processing.

What lives outside this boundary is the policy that decides numeric limits, the scheduling of guest work, and any retry or queueing strategy when throttling occurs. The limiter does not allocate memory for I/O buffers, does not parse protocols, and does not choose which device queue or request runs next. It only answers whether a proposed amount of work may proceed under current budgets, transitions into a throttled state when it may not, and signals when that throttled state should clear.

If this component failed in a way that always denied work, guest I/O would stall or appear hung until higher layers timed out or were reset. If it failed open by always granting tokens without accounting, configured bandwidth or operations-per-second caps would be ineffective, risking host resource exhaustion or unfair sharing. If its time-based refill logic drifted or double-counted, guests could see systematic under- or over-delivery relative to the configured rates.

## Interfaces & Contracts

Callers interact with a composite limiter that exposes two logical buckets: one measured in bytes and one measured in discrete operations. Configuration supplies, for each bucket, a maximum capacity, an optional one-time burst allowance on top of capacity that does not replenish, and the wall-clock duration required to refill an empty bucket back to full capacity. A zero capacity or zero refill interval for a given dimension disables limiting in that dimension while leaving the other dimension active if configured.

The outward-facing consumption operation takes a token count and selects which bucket applies. It returns a boolean outcome: permission granted or denied. Denial does not block the calling thread; it is the caller’s responsibility to defer work and wait for readiness via the asynchronous notification channel described below. A complementary operation exists to add tokens back manually, which supports rollback when an operation was tentatively accounted for but did not complete.

The limiter also exports a file descriptor suitable for integration with an event loop. When the limiter enters its throttled state, the descriptor becomes eligible for readability in the usual polling sense; the embedding must run the designated unblock path when that happens, or spurious wakeups and stuck throttled states can result. A query operation reports whether the limiter is currently throttled.

Embedded instances may receive live updates that replace bucket parameters, disable a dimension entirely, or leave settings unchanged. After replacement, new bucket instances are typically full; this is a documented operational characteristic for hot reconfiguration rather than a hidden invariant.

The limiter consumes monotonic clock readings for refill math and depends on the host to provide a timer facility for the active unblock path. It does not require network access or persistent storage for steady-state operation.

Callers must not invoke the consumption operation while the limiter reports throttled, except where they intentionally expect denial; the contract is that all consumption attempts fail fast until the timer-driven unblock path runs. They must integrate the descriptor with their readiness-based event loop and must invoke the unblock path after readiness. They should treat manual token return as paired with earlier consumption when using rollback semantics, or budgets will drift.

## Data Flow

Traffic originates from guest-driven activity: a block read or write with a known length, or a network packet or batch with a byte count and an operation count. Before performing the associated host work, the embedding translates that work into token deductions. Byte-heavy work draws from the byte bucket; each discrete operation draws from the operations bucket, possibly in addition to bytes when both limits apply.

The data that enters the limiter is therefore numeric: how many bytes and how many operations the next step will consume. The limiter combines that with internal state—current budgets, burst remainder, and timestamps—to decide grant or deny. On grant, internal budgets decrease. On deny for insufficient regular budget, the passive refill path may run inside the bucket logic, advancing simulated tokens based on elapsed monotonic time. If still insufficient, the composite limiter arms a one-shot timer and marks itself throttled.

The output of the limiter to its embedder is binary permission plus the side effect of throttled state and timer arming. Downstream of permission, the embedding performs DMA, copy, or tap I/O. Downstream of denial, the embedding typically leaves a request pending and registers interest in the limiter’s descriptor.

The following diagram situates the limiter between policy configuration and physical I/O without naming implementation artifacts.

```
  +------------------+     numeric limits      +------------------+
  |  Control plane   | ----------------------> |  Token buckets   |
  |  (caps, bursts)  |                         |  (bytes + ops)   |
  +------------------+                         +--------+---------+
                                                        |
  +------------------+     proposed work         |        v
  |  Guest I/O path  | ----------------------> |   Grant / deny   |
  |  (block / net)   | <---------------------- |   + throttle   |
  +------------------+                         +--------+---------+
         |                                              |
         | on grant                                     | descriptor
         v                                              v
  +------------------+                         +------------------+
  |  Host I/O        |                         |  Event loop      |
  |  (disk / NIC)    |                         |  (unblock path)  |
  +------------------+                         +------------------+
```

After the diagram: configuration flows in once or on update; steady-state flow is a stream of proposed byte and operation costs evaluated against bucket state; successful proposals release traffic to hardware or backend services, while failures feed back into scheduling via the event integration.

## Control Flow

Execution is driven by guest-visible I/O attempts in the embedding. The critical path is: translate work to tokens, attempt consumption for each relevant dimension, and proceed only if both dimensions that are enabled succeed. For block devices this often means charging both bytes transferred and one operation per request when both limits are configured.

The first branching point is whether the composite limiter is already throttled. If so, every consumption attempt fails immediately without examining buckets. This global gate ensures that a single timer coordinates backoff for both dimensions and prevents reordering bugs where one bucket might advance while the other is starved.

If not throttled, the enabled bucket for the requested dimension processes the deduction. One-time burst tokens, when present, satisfy deductions first without invoking time-based refill, which preserves the intended burst semantics for cold start or migration scenarios.

When regular budget suffices after optional passive refill, the operation succeeds. When regular budget does not suffice after passive refill, the failure branch arms the timer. A fixed short interval is used for the common case of waiting for partial refill; an extended interval is used when a single request exceeds total bucket capacity so that the effective penalty scales with how far the request overshoots capacity.

The unblock path is triggered only from the event loop when the descriptor signals readiness. It reads the timer counter, clears the throttled flag, and allows the embedding to retry pending work. If the unblock path runs without a pending timer expiration, the embedding receives an error-style outcome so it can detect misuse.

Updates from the control plane replace bucket objects wholesale. That path does not interact with the timer directly; however, a fresh bucket begins with full regular budget, which can immediately relieve pressure after reconfiguration.

A compact view of the throttling state machine follows.

```
                    +----------------+
                    |  Not throttled |
                    +--------+-------+
                             |
               consume succeeds (all relevant dims)
                             |
         +-------------------+-------------------+
         |                                       |
         v                                       v
   +-----------+                         +----------------+
   | Proceed   |                         |  Throttled     |
   | with I/O  |                         |  (timer armed) |
   +-----------+                         +--------+-------+
                                                 |
                                    timer fires, unblock path runs
                                                 |
                                                 v
                                        +----------------+
                                        |  Not throttled |
                                        +----------------+
```

Before the diagram, the steady state is the top node where consumption attempts are evaluated freely. After the diagram, re-entry to the not-throttled state always goes through the timer and the unblock path when throttling was timer-driven; manual configuration updates can also change budgets without traversing this loop.

## State & Lifecycle

Owned state splits into per-bucket state and composite state. Each bucket stores configured capacity, refill duration, remaining one-time burst, current regular budget, and a timestamp of last activity used for passive refill. To keep arithmetic stable across large capacities and refill windows, the implementation precomputes reduced fractions equivalent to the ideal continuous refill rate.

The composite structure holds optional references to each bucket, a monotonic timer object, and a boolean indicating whether throttling is active. Creation always allocates the timer object even when both dimensions are disabled, so later reconfiguration can enable limiting without requiring new capabilities that might be restricted after sandboxing.

At startup of a limiter instance, enabled buckets begin full on the regular budget line; burst allowances sit in their separate counters. Steady-state operation mutates budgets on successful consumption, advances timestamps on burst use, and adjusts timestamps during passive refill to preserve fractional token accumulation across calls.

Shutdown is implicit when the embedding drops the limiter; there is no cooperative teardown protocol. Snapshot save captures serializable bucket fields plus an elapsed-time summary derived from the monotonic clock since last update, not the raw timestamp value, because monotonic instants are not portable across processes. Snapshot restore reconstructs an equivalent last-update instant by subtracting the saved elapsed interval from the current monotonic time at restore, reapplies saved budgets and burst remainders, and builds a fresh timer handle in the disarmed state. Thus a restored guest always begins with the limiter logically unthrottled even if the saved guest had been mid-backoff; the next consumption will re-derive correctness from bucket budgets and time adjustment.

## Failure Modes

Explicit failure surfaces include consumption denial when throttled or when budgets are insufficient, and error returns from the unblock path when invoked without an expiring timer. These are expected control-flow outcomes rather than catastrophic faults.

Propagation happens when embeddings ignore denial and perform I/O anyway; that violates the rate contract and can overrun host resources. Assumptions that, if violated, cause subtle harm include calling the unblock path without integrating the descriptor, which can leave the limiter permanently throttled, and double-consuming burst tokens through manual return paths that are not paired with failed operations, which can inflate apparent throughput.

Silent mis-accounting is unlikely if passive refill invariants hold, but integer rounding is handled deliberately: time credited toward token generation is adjusted upward when remainder nanoseconds would otherwise be lost, preventing systematic extra tokens. Conversely, over-sized single requests that exceed bucket capacity are allowed to complete while imposing a proportional cooldown through a longer timer, which avoids deadlock but means instantaneous traffic can still spike for one quantum.

Snapshot restore clamps impossible time adjustments if monotonic time moves backward or stored intervals exceed representable ranges, falling back to “now” to avoid panic; that edge case slightly perturbs refill fairness but preserves liveness.

## Operational Characteristics

Resource use is small and fixed: two bucket structures plus one timer descriptor per composite limiter instance. The dominant cost is periodic timer events when traffic persistently hits limits, not per-byte work on the hot path. Scaling bottlenecks appear when thousands of devices each thrash at their caps, because each throttled episode arms a timer and generates an event-loop wakeup; policy therefore benefits from setting limits coarsely enough that guests are not constantly edge-triggering timers.

Observability is indirect: embeddings are expected to increment metrics when consumption fails, when the unblock path runs, and when spurious unblock attempts occur. The limiter itself does not emit structured logs on success paths.

CPU overhead concentrates in consumption and passive refill, which perform a few integer multiplications and divisions using widened arithmetic to avoid overflow. Memory traffic is minimal because no allocations occur per operation.

## Design Rationale

The problem is to enforce two independent dimensions—bytes per second and operations per second—with a single coordination point so paravirtualized I/O devices and network backends do not duplicate divergent token-bucket code. Packing both dimensions into one composite object lets block devices charge a request as both data volume and one operation, while network stacks can emphasize bytes on transmit and receive separately by instantiating one composite limiter per direction.

Passive refill on demand avoids a background thread: tokens accrue when the guest tries to work, which matches event-driven VMs. Active timer refill exists because when budgets hit zero the guest might stop producing events; without a wake-up, partially refilled tokens might never become visible. The fixed short timer interval trades a small amount of timer churn for predictable latency to resume after exhaustion. The alternate long cooldown for oversize requests encodes the intuition that borrowing beyond the bucket size must be paid back with silence proportional to the overrun.

The global throttled flag plus single timer simplifies reasoning: there is never more than one outstanding backoff for the pair of buckets, which matches how an embedding usually serializes retries for a single queue. The tradeoff is that exhaustion on one dimension blocks consumption attempts for the other dimension even if that other dimension still has budget; embeddings that need finer-grained independence would require separate composite limiters.

Snapshot semantics favor determinism of numeric budgets over perfect preservation of in-flight backoff, because the timer file descriptor cannot be migrated as a live handle. Resetting to unthrottled with time-adjusted buckets approximates the amount of credit the guest had earned while avoiding stalled I/O after resume.

## Shared Use Across Network and Disk Paths

Network interfaces and block devices both wrap the same composite limiter abstraction but attach it at different granularities and with different natural token mappings. A network endpoint typically installs one composite limiter on the receive path and another on the transmit path so that inbound and outbound guests traffic can be capped independently. A block backend installs one composite limiter per device, shared across read and write queues that belong to that device, so all storage traffic for that disk competes for the same byte and operation pools.

The sharing story is therefore “shared implementation, distinct instances per attachment point,” not a single global limiter across the whole VM. Each instance still internally shares the byte bucket and operations bucket for the workloads that pass through that instance, which is what enforces coupled limits when both dimensions are configured.

Conceptual topology:

```
        +-----------+                +-----------+
        |  Guest    |                |  Guest    |
        |  network  |                |  storage  |
        +-----+-----+                +-----+-----+
              |                            |
              v                            v
     +--------+---------+          +--------+---------+
     | RX limiter       |          | Device limiter   |
     | (bytes + ops)    |          | (bytes + ops)    |
     +--------+---------+          +--------+---------+
              |                            |
     +--------+---------+                  |
     | TX limiter       |                  |
     | (bytes + ops)    |                  |
     +--------+---------+                  |
              |                            |
              v                            v
        host networking              host block I/O
```

Before the diagram: multiple device attachments mean multiple independent composite limiters, each obeying the same rules. After the diagram: network separation by direction avoids coupling ingress policing to egress policing; disk separation by device isolates tenants or volumes when each device carries its own limit.

## Refill Mechanics

Refill has two modes that cooperate. Passive refill runs whenever a deduction attempts to draw from regular budget and finds it short. It compares monotonic time against the last activity timestamp, converts elapsed time to tokens using the pre-simplified refill rate, caps tokens at capacity, and carries fractional time forward so rapid successive calls do not systematically discard partial tokens.

Active refill is not a separate counter; instead, the timer ensures the embedding wakes up and retries consumption, which triggers passive refill again. When elapsed time exceeds the full refill interval in one jump, the passive path can short-circuit to a full bucket without stepping through partial increments, which keeps behavior stable after long idle periods.

One-time burst sits outside the refill cycle. While burst remains, deductions decrement burst first and refresh the activity clock without touching regular budget, which models promotional credits that never regenerate.

## Blocking Versus Non-Blocking Semantics

The design is non-blocking in the thread-sense: the consumption operation returns immediately. Blocking appears only as a logical stall: the limiter refuses forward progress until the timer and unblock path run, so the embedding must retain queued work. This matches typical event-driven virtual machine monitors where the core thread should not sleep inside the device model.

There is an intentional asymmetry. While throttled, every consumption fails fast, including for dimensions that still have budget. That global stall is simpler for single-queue devices but means “blocking” is a property of the whole composite limiter instance, not of an individual bucket in isolation.

## Snapshot Behavior

Serializable state for each bucket includes structural parameters, remaining burst, remaining regular budget, and elapsed nanoseconds since the last activity timestamp interpreted at save time. The composite wrapper stores optional saved bucket snapshots for each dimension. The timer’s armed or disarmed status is not serialized; restore always creates a disarmed timer.

Equivalence after restore is defined in tests to compare capacities, refill intervals, burst remainders, and regular budgets, while treating the reconstructed last-activity instant as distinct if it differs by benign clock skew. That acknowledges migration across hosts with different monotonic bases while preserving the economic meaning of how many tokens had accrued.

If a guest was throttled at snapshot, restore does not continue the backoff; the guest receives full regular budgets relative to the reconstructed time axis. Operators should expect a possible temporary throughput bump immediately after migration, followed by re-enforcement as new traffic arrives. Combined with hot-update semantics that refill buckets on parameter replacement, the system prefers simple predictable state over perfect continuity of timer debt across snapshots.

## Closing Perspective

This subsystem concentrates the mathematics of token buckets and the integration pattern for timer-driven resumption so that heterogeneous I/O backends remain policy-consistent. Understanding passive versus timer-driven refill, the global throttled gate, and snapshot tradeoffs is essential for correct embedding: the limiter is a small state machine with clear contracts, but its leverage on guest-visible performance is large when limits are tight.
