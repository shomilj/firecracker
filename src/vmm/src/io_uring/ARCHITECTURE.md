# Async I/O Ring Architecture

This document explains the asynchronous block-oriented I/O engine that sits between the virtual machine monitor and the host Linux kernel’s completion-based I/O interface. The description is intentionally free of source identifiers so it can be read as a design narrative rather than an index into the implementation.

## Purpose & Boundaries

The subsystem’s responsibility is to expose a safe, opinionated façade over the host kernel’s shared-memory submission and completion rings. Callers obtain a single handle that can enqueue read, write, and durability operations against files that were registered up front, correlate completions with caller-owned context, and optionally integrate with an event notification channel so higher layers can block or poll until work finishes.

What this layer does not do is implement virtio queue semantics, guest memory translation policy, or request scheduling beyond the capacity of the rings themselves. It does not choose which host files back a disk image, does not interpret partition tables, and does not perform caching above what the kernel already provides. The virtio block device selects this engine when asynchronous I/O is desired; that upper layer wraps each guest request in its own tracking object, pins guest buffers, and translates completion results back into virtio responses.

If this component failed entirely—initialization refused, every enqueue returned a resource error, or completions could not be consumed—the block device would either fall back to a synchronous path where one exists or fail open operations that depend on async throughput. The virtual machine would still run, but disk latency would spike and parallel I/O would collapse to serialized system calls, which is a meaningful regression for workloads that rely on deep request queues.

## Interfaces & Contracts

**Outward surface.** Consumers receive a generic ring handle parameterized by the type of caller context stored with each operation. They build high-level operations describing opcode, byte length, file offset, buffer address, and an opaque user payload. They may push those operations, submit batches to the kernel, wait until all outstanding work has produced completions, and pop completions one at a time. Helpers expose how many operations are logically outstanding and how many entries are waiting in the submission side before a submit call flushes them.

**Kernel and OS contracts.** The implementation opens the kernel ring in a deliberately disabled state so that security restrictions and file registration can be applied before any I/O is accepted. It maps three regions shared with the kernel: the submission ring metadata and index array, the dense array of submission entries, and the completion ring with its completion records. It registers optional notification endpoints and a list of file descriptors that are translated into small integer indices for every subsequent operation. Only after registration succeeds does it enable the ring for actual execution.

**Invariants callers must uphold.** Buffer addresses must remain valid until a matching completion is observed; the engine stores only numeric identifiers in shared memory and cannot keep Rust lifetimes for guest mappings. The user payload type must be consistent between enqueue and dequeue. When using fixed file indices, every operation must reference an index that was actually registered; the ring counts how many files are live and rejects out-of-range indices. Because completions reclaim internal storage keyed by opaque handles, callers should eventually drain completions for every successful enqueue or risk leaking that storage.

**Guarantees offered in return.** The implementation refuses to start unless the kernel advertises a feature that prevents silent completion loss on queue overflow, and it probes the ring at startup to ensure read and write opcodes are supported. It caps logical outstanding work so it never exceeds the completion ring capacity, which pairs with the kernel feature to avoid dropping completion events. When an enqueue fails after internal bookkeeping was partially updated, the error path returns the original user payload so callers can retry or abandon deterministically.

The following picture situates the abstraction relative to the virtio block stack and the host kernel.

```
  +------------------+       +---------------------------+
  | Virtio block     |       | Ring façade (this layer) |
  | request path     |------>|  - enqueue / submit      |
  | guest memory     |       |  - pop completions       |
  +------------------+       +------------+-------------+
                                          |
                     mmap shared rings      |  register files,
                     +---------------------+------------------+
                     |        Host kernel async I/O engine     |
                     |  submission ring | SQEs | completion CQ |
                     +-----------------------------------------+
```

Before the diagram, the intent was to show that virtio remains the policy owner while this layer is strictly a transport and lifecycle wrapper. After the diagram, the important takeaway is that all fast-path data movement happens in mapped memory and fixed file tables; the virtual machine monitor never issues per-request open calls on the hot path.

## Data Flow

**Ingress.** An operation enters as a structured description: opcode, fixed file index, optional buffer pointer and length, optional offset, and a caller-defined context object. The layer copies kernel-facing fields into a zero-initialized submission record, sets flags so the kernel always treats the file argument as a registered index rather than a raw descriptor, and inserts the context into an internal slab allocator. The slab index is written into the submission record’s user-data field so the completion path can retrieve the original context without interpreting buffer pointers.

**Inside the rings.** The submission side maintains a monotonically advancing tail, modulo the ring mask, writing each new record into the dense SQE array at the slot chosen by that tail. A separate kernel-visible tail counter is updated with release semantics so the host can observe new entries. The submission layer also initializes the index array to an identity mapping so each slot points to its corresponding record offset, which matches how this implementation uses the rings.

**Kernel execution.** A submit call invokes the enter syscall with the number of pending submission entries and optional flags asking the kernel to wait until a minimum number of completions have been posted. The kernel executes operations asynchronously with respect to the caller’s userspace thread, reading file data into or out of the supplied addresses for read and write operations.

**Egress.** Completion records appear in the completion ring. The consumer maintains its own head pointer, reads the kernel result code and the user-data field, advances the head with release semantics, and uses the user-data value as an index into the slab to recover the original context object. The result code is surfaced as either a byte count or a negative errno-style value wrapped as a standard I/O error.

**Throttling path.** Before accepting another enqueue, the façade compares the total number of operations already submitted or not yet popped against the completion ring capacity. If that count would exceed capacity, enqueue fails even if the submission ring still has free slots, preserving the invariant that completions will always have space.

A second diagram focuses purely on the data objects moving through the pipeline.

```
  Operation + context
        |
        v
  [ Slab insert ]---- user-data key written into SQE
        |
        v
  Submission ring / SQE array  --->  kernel  --->  Completion ring
                                                      |
                                                      v
                                               [ Slab remove ]
                                                      |
                                                      v
                                            Result + original context
```

Narratively before the figure, the flow emphasizes that the only persistent bridge between submission and completion is the integer key, not the guest buffer address. Narratively after the figure, the consequence is clear: if the slab insert succeeds but submission fails, the implementation must return the recovered context on the error path to avoid stranding slab entries.

## Control Flow

**Initialization trigger.** Construction begins when the virtio block device (or tests) request an async engine. The critical path allocates kernel parameters with the ring-created-disabled flag, invokes the setup syscall, validates advertised features, maps memory for both rings, allocates the internal slab, probes opcode support, optionally registers an event notification descriptor, applies restriction records, registers backing files, and finally enables the ring.

**Steady-state operation.** Each guest I/O splits into enqueue, periodic submit, and pop loops driven by the upper layer. The block device typically pushes one operation per virtio request, then explicitly kicks submission so the kernel begins work. Completion processing may be polled in a tight loop or triggered after the notification descriptor fires, depending on integration choices outside this module.

**Branching decisions.** Enqueue fails immediately if no files were registered, if the fixed index is out of range, if the completion backlog would exceed capacity, or if the submission ring is full. Submit with zero pending items and no wait request is a fast no-op. Pop returns an empty option when no completion is ready yet rather than blocking inside this layer.

**Shutdown and replacement.** When the block device swaps backing files, it may construct a fresh ring wired to the same notification endpoint and drop the old handle. Destruction order is significant: mapped ring memory must be unmapped before the kernel ring file descriptor is closed, which the structure ordering enforces implicitly.

## State & Lifecycle

**Owned state.** The façade holds the kernel file descriptor, two mapped regions for submission metadata and submission records, one mapped region for completion records, the slab backing storage for user contexts, a count of registered files, a running total of logically outstanding operations, and cached ring parameters such as masks and sizes.

**Initialization phases.** The lifecycle begins in a disabled ring state where registration syscalls are permitted. Restrictions, if any, are frozen for the lifetime of the instance once applied. File registration expands the fixed table; empty registration is allowed but makes every enqueue fail until corrected. Enabling transitions the ring to accepting kernel execution.

**Steady modification.** Each successful enqueue increments the outstanding-operation counter. Each successful completion pop decrements it with saturating arithmetic as a defensive measure. Submission bookkeeping tracks how many entries have been written to the ring but not yet flushed via the enter syscall; partial submission results after the syscall decrement that shadow counter.

**Cleanup.** On drop, completion and submission queues unmap their regions. The slab should be empty if all completions were processed; otherwise entries may leak as unreachable allocations. The kernel file descriptor closes last so shared memory stays valid through unmap.

A small state-oriented view captures the enablement story without enumerating implementation details.

```
     create (disabled)
           |
           | register files, restrictions, optional notifier
           v
      enable rings
           |
           v
     active (enqueue / submit / pop)
           |
           v
     teardown (unmap, close)
```

The prose before this figure noted that the disabled state exists specifically to make security policy atomic. The prose after reinforces that once active, policy cannot be relaxed without constructing a new instance.

## Failure Modes

**Explicit errors.** Setup may fail if the host kernel is too old, if parameters are invalid, or if memory mapping fails. Feature and opcode probes may fail or report missing capabilities. Registration steps may fail if file lists are too large, if restriction vectors cannot be installed, or if the optional notifier cannot be bound. Enqueue and submit paths surface queue fullness, mapping errors, and syscall failures distinctly.

**Propagation versus containment.** Many errors abort construction entirely, which is appropriate because a half-initialized ring is unsafe to expose. Runtime errors on enqueue return the caller’s context alongside the error so virtio can complete the guest request with failure status. Completion pop errors propagate upward; a failure to remove from the slab indicates serious internal inconsistency rather than a routine I/O problem.

**Silent corruption risks.** Violating buffer lifetime invariants corrupts guest memory or reads garbage without this layer detecting it, because addresses are opaque integers. Mis-counting completions relative to submissions could theoretically desynchronize the outstanding-operation counter, though the implementation guards the enqueue path with capacity checks tied to completion ring size. Assuming opcode support without probing would allow subtle runtime failures; startup probing mitigates that.

**Throttling classification.** Certain errors indicate back pressure: submission queue full or completion backlog at capacity. Callers treat these differently from hard failures because retrying after draining or submitting is valid.

## Operational Characteristics

**Resource use.** Memory footprint scales with the chosen ring sizes plus slab capacity sized similarly to combined queue depths. Each registered file consumes a kernel slot but avoids per-I/O descriptor table churn. Mapped regions use shared mappings with populate-on-map to reduce fault overhead during steady operation.

**Bottlenecks.** Throughput is limited by ring depth, the cost of the enter syscall batching, and host storage latency. Deep queues amortize syscall overhead but increase the memory pinned for in-flight contexts. The completion notification path adds a file descriptor wakeup but reduces busy polling.

**Observability.** The implementation itself is light on metrics; debugging relies on error enums propagated to the block layer, which already logs unexpected completion failures. Property tests in the module compare asynchronous results against synchronous pread and pwrite baselines for randomized sequences, which acts as an architectural regression shield rather than production telemetry.

## Design Rationale

**Why a thin ring façade.** Firecracker-style virtual machine monitors need deterministic control over which syscalls run in the hot path. Registering files once and using fixed indices trims reference counting and permission checks inside the kernel for each operation. The disabled-then-enabled pattern lets restrictions be applied atomically, approximating seccomp-like policy for opcode and file-handle use without trusting future code paths to behave.

**Why slab-backed user data.** Kernel completion records are fixed layout and only carry a 64-bit user-data field. Storing pointers to Rust objects would be unsound across arbitrary reordering of completions. A slab provides stable integer keys and lets the type parameter vary for virtio’s pending-request wrappers versus test harness integers.

**Why require non-dropping completions.** Older kernels could drop completion events under overload. The feature gate ensures the monitor never silently loses track of guest I/O completion, which would otherwise violate virtio correctness and could hang guests waiting for responses.

**Tradeoffs accepted.** Only read, write, and durability operations are modeled in the high-level operation type even though the kernel supports many more opcodes; narrowing the surface reduces attack and misuse surface for the block use case. The submission queue initializes its index array to the identity permutation, which is simple but assumes this mapping strategy; alternative indexing schemes are not supported here. Blocking waits are expressed as submit-and-wait helpers that delegate to the kernel rather than implementing a userspace futex, keeping policy in one place at the cost of syscall coupling.

**How virtio block uses the abstraction.** The asynchronous file engine wraps each guest request in a structure that optionally tracks a guest address for write-back dirty marking. It configures the ring with a restriction that requires fixed file indices and an allowlist covering read, write, and durability operations—the three opcodes the block device issues. It registers a single backing file at index zero and wires an event notifier so the device model can sleep until completions arrive. Reads pin guest memory and enqueue with dirty tracking; writes enqueue without it; durability requests enqueue a flush-style operation. Submission is explicitly kicked after enqueue bursts, and completion handling maps the wrapped context back into virtio-level pending structures. When the backing file changes, the engine rebuilds the ring while preserving the notifier, trading a small setup cost for clean security state.

Together, these choices produce an async I/O engine that is deliberately smaller than the kernel’s full capability, but aligned with the monitor’s threat model, virtio’s needs, and the operational requirement that every guest request eventually sees an explicit completion or a structured error that the block stack can translate into a device-level response.
