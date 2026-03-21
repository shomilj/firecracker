# ARM64 Virtual Machine Architecture

This document describes how the AArch64-specific layer of the virtual machine monitor interacts with the host kernel’s virtualization facilities. It focuses on KVM concepts, processor register handling, interrupt delivery through the emulated interrupt controller, and timer behavior as exposed to guests. The description is conceptual; it does not map to individual identifiers in the source tree.

## Purpose & Boundaries

The ARM64-specific subsystem is responsible for translating the generic machine model (boot protocol, memory layout, device models, and snapshot semantics) into concrete actions against the host kernel’s ARM virtual machine interface. That includes choosing and configuring a virtual interrupt controller, establishing the guest’s initial processor state for Linux boot, publishing a machine description the guest kernel can parse, and coordinating save and restore of virtual CPU and interrupt-controller state.

This layer does not implement the generic device manager, virtio transports, or guest memory allocation policy. It does not define the overall VMM lifecycle policy beyond what is required to wire the architecture correctly. If this component failed or misconfigured the host interface, guests would not boot, interrupts would not reach devices or timers, snapshots would be inconsistent across processors, or the guest would observe incorrect time and affinity information.

The following diagram situates the ARM64 layer between the portable VMM core and the host kernel.

```
+------------------------------------------------------------------+
|  Portable VMM (devices, memory, execution loop, API)            |
+------------------------------------------------------------------+
                              |
                              v
+------------------------------------------------------------------+
|  ARM64 adaptation layer (this subsystem)                         |
|  - Virtual machine / vCPU configuration                        |
|  - Register images, MP state, optional paravirtual time          |
|  - Virtual GIC device, MMIO layout, device tree content        |
+------------------------------------------------------------------+
                              |
                              v
+------------------------------------------------------------------+
|  Host kernel (KVM on ARM64)                                    |
+------------------------------------------------------------------+
```

The portable core depends on this layer to provide correct ordering and semantics for ARM-specific operations; the host kernel enforces additional constraints that are not visible in the portable code alone.

## Interfaces & Contracts

**Upstream inputs.** The architecture layer receives the KVM file descriptor, the virtual machine file descriptor, guest memory views, boot configuration (kernel entry, command line, optional initial ramdisk), CPU templates that may override or filter register values, and the machine’s desired processor count. It consumes optional host capability probes (for example whether the guest physical counter can be offset or reset in a controlled way) and, when building the guest firmware blob, information about MMIO devices and their interrupt numbers.

**Downstream outputs to the host.** Operations include creating virtual processors, initializing them with a target CPU type and feature bitmap, reading and writing register values through the single-register access path, setting multiprocessor run state, attaching a paravirtual steal-time region when supported, and driving a virtual interrupt controller device: creation, address assignment, sizing of the interrupt space, and final activation. For migration and snapshots, the layer exports the guest memory description together with serialized interrupt-controller state and per-vCPU register images.

**Guest-visible contract.** The guest must see a coherent ARMv8 environment: Linux boot protocol expectations (entry point, first general-purpose register holding the flattened device tree address, processor state suitable for exception level one), a valid interrupt controller tree with maintenance interrupt wired for the hypervisor, generic timer interrupts described as private peripheral interrupts, and power-state coordination so secondary processors start only when the operating system requests it. The device tree must list CPU affinity using the same affinity values the hypervisor reports for each virtual processor.

**Invariants.** Callers must create virtual processors before the virtual interrupt controller is fully initialized, because the host implementation ties interrupt controller setup to an already-created processor set. Register save and restore must respect initialization order: certain wide vector extensions require finalization before the rest of the register file is restored, and the list of registers must be obtained only after the virtual processor has been initialized. The guest interrupt numbering scheme must stay within the configured maximum shared peripheral interrupt index so that device assignment and the controller configuration agree.

The virtual interrupt controller exposes three broad classes of interrupt to software. Software-generated interrupts are used for low-level CPU-to-CPU signaling. Private peripheral interrupts are per-CPU lines such as the architecture timers and the maintenance interrupt used by the hypervisor side of the controller. Shared peripheral interrupts are the shared lines used for virtio and other platform devices. The microVM caps how high shared peripheral numbering can go so that device assignment stays within a predictable range.

```
        +----------------------+
        |  Virtual GIC model |
        +----------------------+
        | SGI (low range)      |  CPU-to-CPU signaling
        | PPI (mid range)      |  Per-CPU (timers, maintenance)
        | SPI (high range)     |  virtio, UART, and other devices
        +----------------------+
                 ^
                 |
   Device tree: SPIs for devices; fixed PPIs for arch timer;
   maintenance PPI depends on controller variant.
```

The diagram summarizes how interrupt classes partition the space. In the flattened device tree, each interrupt specifier for this controller typically uses three cells: interrupt class, number within that class, and trigger or polarity flags. Virtio devices reference shared peripheral lines; the timer node wires the four standard timer interrupts as private peripheral interrupts.

## Data Flow

**Boot path.** Kernel bytes are loaded into guest RAM at a fixed offset from the start of DRAM. The flattened device tree is built in host memory and written near the end of guest RAM (or at the start when RAM is small), subject to alignment and size limits from the architecture boot specification. Each virtual processor’s affinity register value is read from the hypervisor and embedded in the device tree so CPU nodes match hardware-reported IDs. Optional cache topology information is read from the host’s sysfs and folded into CPU nodes so that guests see a hierarchy consistent with the host where intended.

**Runtime register path.** During configuration, template-driven register values are applied first. Boot registers then set the program counter, the processor state mask for a faulting kernel entry, and the first argument register to the device tree guest physical address on the primary processor. Secondary processors are marked powered off until the guest uses the power-state interface. When optional host support exists, the physical counter register exposed to the guest is reset near boot so the guest does not inherit the host’s raw count value.

**Interrupt path.** Devices obtain guest-side interrupt indices (global system interrupt numbers) that map cleanly onto shared peripheral interrupts in the virtual controller. The device tree connects virtio and other devices to the interrupt controller node with interrupt type, number, and trigger semantics. The ARM architecture timer is described with a fixed set of private peripheral interrupt numbers for secure, non-secure, virtual, and hypervisor timers, matching the usual ARMv8 timer binding.

**Snapshot path.** Saving collects multiprocessor state, the full set of register identifiers returned by the host, and per-line state for the distributor and per-CPU redistributor or CPU interface regions, plus the CPU interface system register banks keyed by processor affinity. Pending interrupt tables may be flushed to guest RAM before readout. Restoring replays virtual processor initialization, applies special-case ordering for vector extension state, then restores remaining registers and multiprocessor state, and reattaches paravirtual time if a guest physical address was recorded.

The next diagram summarizes data movement at boot and snapshot time.

```
Boot:
  Host files (kernel, initrd) --> Guest RAM
  CPU template + KVM registers --> vCPU state
  Device inventory + GIC layout --> FDT blob --> Guest RAM

Steady state / snapshot:
  vCPU host run loop <--> Guest RAM
  irqfd / virtio --> GIC SPI -> vCPU

Save:
  vCPU registers + MP state --> snapshot blob
  virtual GIC device state (dist / redist or CPU iface) --> snapshot blob
  Guest memory --> external memory image
```

## Control Flow

**Initialization.** Virtual machine creation allocates the generic VM state. Virtual processors are created next. Only after that does the layer instantiate the virtual interrupt controller: create the in-kernel device, program distributor and CPU interface or redistributor base addresses in the high MMIO region below one gigabyte, declare how many interrupts exist up to the configured ceiling, and issue the finalize control operation. The primary processor is initialized with the preferred CPU target from the host, feature bits for the standard power-state interface, and optionally powered-off secondaries for SMP.

**Boot configuration.** Before the guest runs, the architecture layer applies CPU templates, sets boot registers, writes the device tree, and relies on the portable loader to have placed the kernel. The boot vCPU runs first.

**Execution.** Virtual processors enter the host run loop. Unexpected exits from the hypervisor are treated as errors in the architecture-specific emulation hook; the design expects normal operation to be MMIO and interrupt driven through established mechanisms.

**Power and secondary CPUs.** The device tree advertises power-state coordination via the hypervisor call conduit. The guest brings additional processors online through that interface; the hypervisor reflects the powered-off initial state for non-boot CPUs.

**Migration.** Save is triggered by the upper layer; the ARM64 code gathers affinity values for each processor, then saves GIC state using those values as keys. Restore recreates the controller state before resuming execution.

The host enforces a strict order: the virtual machine exists first, then every virtual processor is created, and only then is the in-kernel interrupt controller device created and finalized. Attempting to finalize the controller before processors exist violates kernel expectations. The sequence below is typical for bringing the system to a first guest entry on the boot processor.

```
[Create VM]
     |
     v
[Create virtual CPUs 0..N-1]  (secondaries start in powered-off state)
     |
     v
[Create virtual interrupt controller device]
     |
     v
[Program MMIO bases: distributor, redistributors or CPU interface]
     |
     v
[Declare interrupt ceiling and finalize controller init]
     |
     v
[Per CPU: init, optional scalable vector extension finalize, boot regs]
     |
     v
[Run boot vCPU]
```

Snapshot restore must mirror compatible steps: virtual processor initialization and optional wide-vector finalization before bulk register replay, then multiprocessor state, so that the kernel sees a topology consistent with the saved image.

## State & Lifecycle

**Owned state.** The architecture layer owns the virtual interrupt controller handle, the virtual processor initialization descriptor (target type and feature words), per-vCPU paravirtual time guest physical addresses when enabled, and the optional capability flags discovered at runtime. It does not own guest RAM contents except where it writes the device tree and participates in snapshot memory description.

**Lifecycle phases.** Startup proceeds: VM creation, vCPU creation, IRQ controller setup, vCPU init and finalize, boot register programming, FDT write. Steady operation is dominated by vCPU runs and device injection. Shutdown is largely handled by the parent; the ARM64 layer does not define a lengthy teardown beyond dropping resources.

**Recovery.** Snapshot restore must supply consistent memory, GIC, and vCPU images. Mismatched processor counts between saved GIC state and the current machine are rejected. Invalid or partial GIC register state is treated as an error rather than silently continuing.

## Failure Modes

**Host errors.** ioctl failures surface as structured errors: register get/set failures, device attribute failures, failure to create the GIC device or finalize it, or failure to query the register list. A bad file descriptor from a closed resource propagates as the same class of error.

**Ordering violations.** Initializing the interrupt controller before vCPUs exist can fail at the host. Reading the register list before the vCPU is initialized fails. Restoring registers in the wrong order can fail when wide vector state requires special handling.

**Logical inconsistency.** If the device tree and the actual GIC version or MMIO addresses disagree, the guest kernel will not boot correctly; the layer keeps FDT generation and GIC setup aligned. If SPI indices exceed the configured maximum, interrupt assignment could silently overlap; the platform caps the maximum SPI and derives usable GSIs from that.

**Assumption violations.** Assuming the guest can run without a finalized GIC leads to missing interrupts. Assuming secondary CPUs are runnable without going through the power-state interface breaks SMP boot. Assuming the physical counter can be reset without probing host capability leads to best-effort behavior: the reset is skipped when the host does not advertise the capability.

## Operational Characteristics

**Resource use.** Each vCPU has an open file descriptor and kernel-side state. The GIC device adds another descriptor and kernel memory for distributor and redistributor or CPU interface tables. Register save can walk hundreds of registers per vCPU; the implementation pre-allocates a reasonable list size and grows if the host reports a larger set.

**Bottlenecks.** Large register images dominate snapshot size and time. GIC save walks MMIO ranges through device attributes. The single-register API is flexible but not batched for bulk save.

**Observability.** Errors are logged when unexpected KVM exit reasons occur. Metrics may count such failures. Debug output can dump register sets for troubleshooting.

## Design Rationale

**KVM-first model.** ARM64 guests in this stack are built around the Linux KVM user API: one-register access for AArch64 state, a device abstraction for the virtual GIC, and the MP state ioctl for run/pause semantics. This matches the kernel’s reference implementation and avoids duplicating low-level emulation in userspace.

**GIC version selection.** Preferring the newer interrupt controller version reduces legacy limitations and matches modern ARM servers; when the host cannot create that device, falling back to the older version preserves compatibility.

**MMIO placement.** Placing the distributor and redistributors immediately below the one-gigabyte boundary keeps the high MMIO window for devices while satisfying alignment and size rules for the KVM virtual GIC device.

**Boot register choices.** The Linux kernel for ARM64 expects a specific entry state: exception level one, appropriate flags in the processor state register, the program counter at the kernel entry, and the first argument register pointing at the device tree. Secondary processors start in a powered-off state so the kernel’s own bring-up sequence runs.

**Physical counter handling.** The guest’s view of physical time can be anchored near zero when the host supports counter offset capability, reducing leakage of host uptime into the guest and making behavior more predictable for tests and migration; the implementation is conditional because older kernels lack the required support.

**PSCI and HVC.** The power-state interface is advertised with the hypervisor call method because KVM on ARM uses the HVC conduit for privileged calls from the guest, not the secure monitor call path used on bare metal secure firmware.

**Paravirtual steal time.** When supported, registering a guest physical page for steal time allows the guest scheduler to account for time stolen by the hypervisor; the layer records the address for restore.

Taken together, these choices bind the portable monitor to the Linux user API on AArch64: template-driven virtual processors, Linux boot conventions, a virtual interrupt controller with explicit MMIO layout and snapshot support, and a device tree that describes processors, timers, power control, and devices. Correctness under migration and performance tuning both depend on respecting host ordering rules, the single-register access model, and the interrupt topology described above.
