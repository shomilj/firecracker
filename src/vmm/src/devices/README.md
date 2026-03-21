# Virtual device layer

## 1. Role in the microVM

1.1. The virtual device layer sits between the guest’s programmed I/O (MMIO traps, and on x86 also port I/O for legacy hardware) and the host: it translates guest-visible register semantics into memory operations, queue walks, host syscalls (network TAP, block files, sockets), and interrupt injection into the virtual machine.

1.2. Design goals are strict separation of concerns: a single **address-space router** maps guest physical addresses to device handlers without knowing device semantics; **virtio** splits a **transport** (negotiation, configuration, interrupts) from **device backends** (block, net, entropy, balloon, vsock); **legacy** devices provide minimum platform glue (PC keyboard controller, UART, optional RTC); **pseudo** devices exist only for observability; **ACPI-related** helpers expose a small set of guest-visible state (e.g. VM generation identity) outside the virtio stack.

1.3. Everything that must run when the guest is not executing in a tight loop is driven by the VMM’s event loop: queue kick **eventfds**, device interrupt **eventfds**, TAP and socket fds, and timers. The device code does not own the loop; it exposes fds and reacts to readiness.

---

## 2. Address routing and the MMIO bus

2.1. The MMIO bus is a non-overlapping map from **base address + length** to a **mutex-protected device object**. Lookup is “find the range whose start is the greatest start ≤ address,” then verify the address falls before the range end. Inserts reject zero-length regions and any overlap with existing regions.

2.2. Reads and writes take a **guest physical address** and a byte slice. The bus resolves the containing device and the **offset within that device’s window**, then dispatches to the device’s handler. Missing addresses return failure to the caller (typically the memory emulation path) so unmapped MMIO can be modeled as RAZ/WI or fault depending on the architecture layer.

2.3. **BusDevice** is a closed sum type: legacy keyboard controller, optional ARM real-time clock, virtio MMIO transport wrapper, a boot-time pseudo device, serial console, and test doubles. Only the serial path participates in the **event subscriber** trait so the UART can be registered with the poller; other devices rely on the transport’s fds or on separate registration paths.

2.4. **Concurrency model**: each device is behind one mutex. The bus is `Clone` by cloning the shared `Arc<Mutex<...>>` map—snapshot or diagnostic code can hold a second handle to the same bus without duplicating device state.

---

## 3. Virtio: transport split from devices

3.1. Virtio in this codebase uses the **virtio 1.x MMIO transport** only (no legacy PCI virtio). The transport exposes a single **4 KiB** register window: generic registers in the first 256 bytes, device-specific configuration space above that.

3.2. **Magic, version, device ID, vendor ID** are readable for discovery. **Feature negotiation** uses two pages (32 bits each) for the 64-bit feature mask; when the driver reads the high page, the transport ORs in the virtio 1.0 capability bit so the guest can negotiate modern virtio.

3.3. **Queue selection** is indirect: the driver writes a queue index, then reads/writes queue metadata (max size, selected size, ready bit, descriptor/avail/used GPA halves). Queue updates are gated on device status: once the driver has moved past `FEATURES_OK` toward `DRIVER_OK`, queue fields are mutable; otherwise writes are logged and ignored.

3.4. **Device status state machine** follows the virtio spec: `INIT` → `ACKNOWLEDGE` → `DRIVER` → `FEATURES_OK` → `DRIVER_OK`. Transitions are **monotonic**; the driver must not clear bits. Setting `DRIVER_OK` triggers **activation**: the transport attaches guest memory to the device and calls the backend’s activation hook. If activation fails, the transport sets `DEVICE_NEEDS_RESET`, fires a **configuration-change interrupt**, and logs the error.

3.5. **Reset** is initiated by writing status `0`. If the backend supports reset, it returns ownership of interrupt and queue eventfds; the transport reinitializes queue objects (keeping max sizes), clears interrupt status, resets transport registers except **config generation** (monotonic), and leaves eventfds in place so pending counts may cause spurious wakeups but no incorrect state.

3.6. **Interrupt status** uses two bits: “used buffers” (vring) and “configuration changed.” The transport shares an atomic word with the interrupt helper so MMIO reads of the status register reflect the last signaled reasons. The **IRQ path** sets bits in the atomic word and **writes** to a Linux `eventfd`; the KVM layer maps that fd to a guest IRQ line so **one write** completes injection.

3.7. **Vhost-user nuance**: for backends that run outside the process, the remote side cannot always mirror interrupt status bits. When the transport is marked as vhost-backed, reads of the interrupt status register **collapse** “unknown nonzero” to the vring bit so the guest can make progress; only the explicit config bit is preserved when set, so configuration-change interrupts remain distinguishable.

3.8. **Guest notification (kicks)**: virtio MMIO places a **queue notification doorbell** at a fixed offset from the device base. The VMM registers each queue’s kick `eventfd` with KVM as an **ioeventfd** targeting that MMIO address with a **data value equal to the queue index**. When the guest writes the index to the doorbell, KVM increments the corresponding fd without necessarily exiting to userspace—reducing cost for high-I/O workloads.

---

## 4. Virtio device abstraction and queues

4.1. **VirtioDevice** binds the transport to backend behavior: **feature words**, **queue count and max sizes**, **per-queue eventfds** (one per queue, used for kicks), **interrupt trigger** (shared atomic + irq eventfd), **config space** read/write, **activation** with guest memory, optional **reset**, and **dirty tracking** of queue memory for migration.

4.2. **Feature acknowledgment** is defensive: the driver’s bits are ANDed with the device-offered mask; unknown bits are warned and stripped so the invariant “acked ⊆ available” always holds.

4.3. **Queue** represents a split virtqueue after the driver programs addresses. On activation, the queue **initializes** against guest memory: it validates power-of-two size, alignment of descriptor table (16 bytes), avail ring (2-byte), used ring (4-byte), obtains host-mapped pointers, and marks those regions **dirty** in the bitmap for tracking. Internal state tracks **next avail / next used** indices, optional **event index** notification suppression, and **used elements** bookkeeping.

4.4. **Descriptor chains** are walked from the descriptor table with TTL to prevent cycles. Chains are iterated for **read-only** vs **write-only** descriptors; network and block paths build **scatter/gather** views (`iovec`-style helpers) to map guest buffers into host operations without a single contiguous copy.

4.5. **Availability** is consumed by comparing `avail.idx` with the device’s cursor; **malformed** availability (avail index implying more entries than the queue can hold) is treated as a **fatal consistency error** in the networking layer: logging and metrics are updated, and the process may terminate rather than looping on a hostile guest.

---

## 5. Block device

5.1. Block is split into **in-process virtio-blk** and **vhost-user block**. The former performs full descriptor processing in the VMM: read/write/flush, optional read-only enforcement, **rate limiting** on bytes and ops, and **async or sync** host file engines depending on configuration.

5.2. **Vhost-user block** hands off virtqueues to a separate process after **feature and protocol negotiation** with the `vhost` frontend. The VMM still owns the MMIO transport and configuration space, but **queue processing** is not driven by the same code path as in-process virtio—kicks go to the backend via the shared protocol. **Config updates** from the backend may require a **config interrupt** and interrupt status handling as described for vhost-user.

5.3. **Persistence** snapshots queue state, negotiated features, transport registers, and backend-specific data (file path, cache mode, etc.). Before save, backends run a **prepare** hook so in-flight state is quiesced where possible.

---

## 6. Network device

6.1. The virtio-net model uses a **TAP** interface on the host, **RX and TX queues**, and a **config space** with MAC address. **Offload feature bits** (checksum, TSO, UFO, mergeable buffers) are negotiated and shape how **virtio-net headers** prefix L2 frames.

6.2. **RX path**: the guest posts buffers; the device fills them from TAP frames, respecting **vnet header** and **rate limits**. **TX path**: the guest submits frames; the device validates length, parses headers, and writes to TAP. **MMDS** (metadata service) can intercept or synthesize traffic when integrated, so the net device is both a NIC and a policy point for guest HTTP-like metadata access.

6.3. **Buffers** aggregate multiple descriptor chains into a single **mutable iovec** for scatter/gather RX, with careful rules about **non-overlapping** guest memory. **Metrics** are per-device for failures and drops.

6.4. **Event handling** ties TAP readability, queue kicks, and the **rate limiter** together; failures in signaling interrupts increment **event failure** metrics.

---

## 7. Balloon device

7.1. The balloon uses **three virtqueues**: inflate, deflate, and optional **stats**. The virtio balloon config records **target pages** and **actual pages**; feature bits enable **deflate-on-OOM** and **stats**.

7.2. **Inflate/deflate** paths translate guest-supplied page frame numbers into guest memory operations: **removing** pages from the guest’s view (balloon inflation) or **returning** them. The implementation uses **compaction** of PFN lists and batching to avoid huge allocations per request.

7.3. **Stats** use a **timerfd** (or periodic timer) when stats polling is enabled: on expiry the device injects work on the stats queue to collect guest-reported memory statistics (swap, faults, free/total memory, etc.).

7.4. **Activation** wires memory, queues, and timers; **metrics** track guest behavior and host-side errors.

---

## 8. Entropy (virtio-rng)

8.1. A **single queue** receives buffer chains; the device fills them with **cryptographically strong** random bytes from the host crypto provider, subject to **rate limiting** on both operation count and byte volume.

8.2. **Back-pressure**: if the rate limiter denies a request, the device replenishes tokens consistently so accounting stays balanced; empty guest buffers complete with zero length.

8.3. **Interrupts** signal the guest when buffers are consumed; the model matches other single-queue devices.

---

## 9. Vsock

9.1. Vsock implements **connection-oriented** virtio sockets with **RX, TX, and event** queues. The **guest** is identified by a **context ID**; the **host backend** (Unix domain socket muxer in the common case) maps vsock packets to host connections.

9.2. **Packet format** parses and builds headers matching the Linux vsock UAPI layout: **operation types** (connection, payload, credit, shutdown, etc.); **credit-based flow control** prevents unbounded buffering.

9.3. **Mux** for Unix sockets maintains **connection state machines**, **rx/tx queues** of packets, and **kill** queues for teardown. **Epoll-driven** readiness is integrated with virtio queue kicks: when the backend has data, the RX path fills guest buffers; when the guest posts TX, the muxer forwards to the socket.

9.4. **Activation ordering** uses an extra **activation eventfd** so the device is not registered for backend fds until after virtio activation completes—avoiding **spurious events** before queues are valid.

9.5. **Transport reset** events propagate through the event queue so the guest can resynchronize after reconfiguration or migration.

---

## 10. Legacy devices

10.1. **x86 keyboard controller (i8042)** emulates just enough of the PS/2 controller to support **Ctrl+Alt+Del** and a **CPU reset** command path. Port I/O is **8**-byte aligned within a small window; **status** and **data** ports behave differently; **interrupts** and **command state machines** follow simplified hardware.

10.2. **UART serial** is a **wrapped** `vm_superio` serial with **eventfd-driven** RX when connected to stdin in interactive mode. **IRQ** registration uses the same **irqfd** pattern as virtio devices.

10.3. **ARM PL031 RTC** (when present) exposes the **real-time clock** MMIO register set for time reads and programmable alarms.

---

## 11. Pseudo devices

11.1. The **boot timer** is a minimal MMIO device: a **single-byte write** at offset zero with a known magic value logs **guest boot time** (wall and CPU time) relative to VMM start. Reads are no-ops. It is not a virtio device and carries no IRQ.

---

## 12. ACPI-adjacent guest state (VM generation ID)

12.1. **VMGenID** exposes a **128-bit random** value in guest memory, **aligned** to 8 bytes, distinct from virtio. It exists so Windows guests can detect **VM generation changes** when restoring snapshots or cloning instances.

12.2. **Generation** is written at construction; **persistence** saves/restores the value and **bump** semantics on restore can change the ID and **notify** the guest via an **interrupt** on a dedicated **GSI** using the same **irqfd** mechanism as other devices.

---

## 13. KVM: irqfd and ioeventfd interaction

13.1. The diagram below summarizes how virtio devices connect to KVM without naming implementation symbols:

```
                    +-------------------+
                    | Guest writes MMIO |
                    | (queue notify)    |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | KVM ioeventfd     |
                    | (addr + data = q) |
                    +---------+---------+
                              | increments
                              v
                    +-------------------+
                    | queue kick fd     |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | VMM event loop    |
                    | queue handler     |
                    +-------------------+


                    +-------------------+
                    | Device completes  |
                    | work + signals    |
                    +---------+---------+
                              | write(1)
                              v
                    +-------------------+
                    | KVM irqfd         |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | Guest IRQ line    |
                    +-------------------+
```

13.2. **ioeventfd** binds **guest physical MMIO writes** at the virtio notify address to **queue-specific** events. The **data** field distinguishes which queue index was written—critical for multi-queue devices.

13.3. **irqfd** binds the **interrupt eventfd** to a **Global System Interrupt** number allocated by the resource allocator. **Virtio** uses a **single IRQ per device** in the current design; all reasons (vring vs config) share the line, with the interrupt status register disambiguating.

13.4. **Serial** on ARM uses the same irqfd mapping for its UART IRQ. **Legacy i8042** uses separate irqfds for keyboard-related lines. **VMGenID** registers its own irqfd for the **generation-change** notification.

---

## 14. Snapshot persistence and restore

14.1. **Queue state** is serialized as **sizes, addresses, ready flag,** and **internal indices** (`next_avail`, `next_used`, `num_added`). Restore reconstructs queues and, when the device was activated, **re-initializes** pointers to guest memory so **avail/used** rings match the saved cursor positions.

14.2. **Virtio device state** bundles **feature words**, **per-queue snapshots**, **interrupt status word**, and **activated flag**. Restore checks **device type**, **queue count**, **max queue size**, and **acked ⊆ avail** before rebuilding queues. **Event index** suppression is re-enabled when the negotiated feature bit is present.

14.3. **MMIO transport state** saves **feature selectors**, **queue selector**, **device status**, and **config generation**—not the guest’s live memory, but the **transport’s** view of negotiation. **Interrupt status** is restored from the device snapshot.

14.4. **Backend-specific** state (block file, vhost socket, vsock CID, balloon targets, TAP MAC, etc.) lives in separate snapshot sections; **device-specific** hooks run after generic virtio state is restored.

14.5. **Dirty memory**: queue initialization marks descriptor and ring pages dirty for migration; **mark_queue_memory_dirty** during save ensures migration bitmaps include **queue structures** the device will touch after resume.

---

## 15. Cross-cutting concerns

15.1. **Errors** propagate as typed enums (`DeviceError`, backend-specific errors). **IRQ signaling** failures surface as I/O errors; **queue** errors surface as `QueueError` or `InvalidAvailIdx` depending on severity.

15.2. **Metrics** are pervasive: per-device counters for network, block, balloon, vsock, vhost-user, and legacy paths. **Shared** metrics aggregate where multiple instances exist.

15.3. **Logging** uses warnings for **invalid** transitions (e.g., queue updates in wrong driver state) rather than failing the VM—favoring **observability** over **hard failure** except where security requires (e.g., inconsistent avail indices).

---

## 16. Mental model summary

16.1. **MMIO bus** = address decode → **mutex** → **device**.

16.2. **Virtio** = **MMIO transport** (spec + lifecycle) + **VirtioDevice** (queues + fds + config + activation) + **backend** (block/net/rng/balloon/vsock semantics).

16.3. **KVM** = **ioeventfd** accelerates **queue kicks**; **irqfd** accelerates **interrupt injection**; both are **eventfd**-based bridges between guest execution and the VMM event loop.

16.4. **Snapshots** = **transport + device + queue** state + **backend-specific** resources, revalidated on restore.

This architecture keeps the **guest-visible protocol** (virtio MMIO, descriptor rings) **stable** while allowing **backend diversity** (in-process emulation, vhost-user, TAP, sockets), and keeps **interrupt and kick paths** uniform across devices.
