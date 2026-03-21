# Firecracker continuous-integration resource bundle

1. **Purpose and scope**

   1.1. The bundle supplies everything needed to **build and describe** the software artifacts that automated tests use when exercising the virtual machine monitor: compressed guest root filesystems, minimal early-boot images, and Linux kernels compiled from vendor sources with layered configuration. It also ships **syscall restriction policies** that constrain how different threads inside the monitor process may interact with the host kernel.

   1.2. The design optimizes for **repeatable CI**: the same inputs (base distribution, configuration fragments, overlay content, compiler flags) should yield inspectable outputs (package manifests, kernel configuration snapshots, binary images) so failures can be bisected to either the hypervisor, the guest image, or the policy layer.

   1.3. Two parallel concerns run through the bundle: **guest-side behavior** (what runs inside the microVM to make tests possible—network bring-up, timing signals to the host, memory stress tools) and **host-side behavior** (how the VMM’s threads are allowed to call into the kernel). Documentation below treats both, because they are often exercised together in integration tests.

2. **End-to-end build architecture**

   2.1. A single orchestrating script drives the pipeline. It installs host build dependencies (compiler toolchain, container runtime, squashfs tooling, initramfs helpers), selects the host CPU architecture, and writes all deliverables into an **architecture-named output directory** beside the resources tree so x86-64 and AArch64 builds never overwrite each other.

   2.2. **Root filesystem construction** uses a privileged container that bind-mounts a working directory. A seed tree is copied in first: it carries pre-authored unit files, small compiled helpers, and shell logic. Inside the container, a second script merges that seed into a minimal Ubuntu base, runs the package manager non-interactively, applies sysctl and systemd customizations, records an installed-package manifest on the root user’s home area, then **exports selected top-level directories** (userland binaries, libraries, configuration, home and root content) into a temporary root tree. Mountpoint directories are created empty so the final image is bootable when combined with kernel-provided virtual devices.

   2.3. **Command-line tooling for object storage** is installed into the guest tree from upstream bundles (not from distribution packages), placing binaries under a local prefix so tests can upload logs or artifacts when cloud credentials are available.

   2.4. The populated tree is compressed with **Zstandard-backed squashfs**, preserving root ownership semantics suitable for loop-mounting or direct kernel attachment as a read-only block device. The manifest is moved alongside the squashfs image for auditing “what was inside” a given CI run.

   2.5. **Small native utilities** are compiled on the host immediately before rootfs assembly, then removed from the seed tree after packaging so the source of truth remains the checked-in C sources while the build stays reproducible. One utility is only built on AArch64 where a specific instruction-level test needs it.

   2.6. **Initramfs** is a separate, much smaller artifact: a static shell binary provides `init`, a minimal `dev`/`proc`/`sys` layout is created, and a hand-written `init` script mounts pseudo-filesystems, writes a magic value to a **fixed guest-physical MMIO address** (architecture-dependent), then drops to an interactive shell on the console. That write implements a **boot-time handshake** with the VMM: the monitor exposes a synthetic device at a known address; early userspace maps physical memory and stores a byte there so the host can measure how long boot took. The same address constants appear in the main guest init wrapper and in the VMM’s architecture layout.

   2.7. **Kernel builds** clone a long-lived Linux fork maintained by the cloud vendor, discover the **newest tag** matching either a microvm-specific naming pattern or a generic kernel pattern for the requested baseline version, check out that tag, and concatenate **multiple configuration fragments** into one `.config`. Older values are overridden by newer fragments. The build target differs by architecture (ELF vmlinux on x86-64, PE Image on AArch64). Output is normalized to a version string derived from the kernel’s own release file, with optional **flavour suffixes** for variants such as “no ACPI” on x86-64.

   2.8. **Debug kernels** repeat selected builds with extra fragments enabling tracing subsystems and DWARF debug info, writing into a nested `debug` output directory. A post-step may **split debug symbols** from the main binary, strip the executable, attach a GNU debug link, and gzip the detached symbol file for storage efficiency while keeping GDB usable.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  Host build environment (toolchain, Docker, squashfs, cpio)    │
  └───────────────┬───────────────────────────────┬───────────────┘
                  │                               │
                  v                               v
┌─────────────────────────┐         ┌────────────────────────────┐
│  Seed overlay +         │         │  Vendor Linux fork +       │
│  chroot customization   │         │  layered Kconfig fragments │
│  -> squashfs + manifest │         │  -> vmlinux / Image        │
└─────────────────────────┘         └────────────────────────────┘
                  │                               │
                  └───────────┬───────────────────┘
                              v
                  ┌───────────────────────┐
                  │  CI artifacts per     │
                  │  architecture folder  │
                  └───────────────────────┘
```

3. **Guest root filesystem: policy and personality**

   3.1. The guest is intentionally **minimal but serviceable**: core userspace, init system, SSH server, networking utilities, lightweight scripting, performance tools, tracing, and hardware introspection where the architecture allows. Architecture-specific packages are added only on x86-64 (model-specific register and CPUID tooling).

   3.2. **Authentication** is relaxed for automation: the superuser password is cleared so non-interactive workflows can log in where needed, and the serial console is configured for **automatic login** to the superuser on the serial line that Firecracker typically wires to the microVM console—avoiding a login prompt that would block tests waiting for a shell.

   3.3. **Network time and resolver daemons** are disabled by unlinking their unit symlinks, reflecting an environment where **no real upstream DNS or NTP** is assumed; tests supply their own notion of time and naming.

   3.4. **Temporary storage** uses an in-memory filesystem backed by the kernel tmpfs driver, reducing wear on any backing disk and matching ephemeral workloads.

   3.5. **Documentation and locale trees** are deleted from the image after package installation to shrink size and attack surface.

   3.6. A sysctl knob **disables unprivileged BPF program attachment**, closing a known speculative-execution-related class of issues at the cost of forbidding unprivileged eBPF in the guest—which aligns with the threat model of short-lived, test-controlled workloads.

4. **Overlay: systemd integration and virtual networking bootstrap**

   4.1. A **oneshot service** runs early in boot and executes a shell script that enumerates every non-loopback network interface, discovers its Ethernet address, and derives an **IPv4 address** from the last four octets of a vendor-specific prefix baked into the address. The script assigns a `/30` address to each interface and brings the link up. The design is deterministic: given MAC allocation rules in tests, the guest always lands on predictable addresses without DHCP.

   4.2. The service is ordered so **secure shell** starts after addresses exist, so tests can immediately connect over the network namespace or tap setup the harness provides.

   4.3. A **mount unit** replaces the persistent directory used for runtime state under the init system’s state path with a **tmpfs** of bounded size and inode count, mode `1777`, with `nosuid` and `nodev`. That keeps the read-only root clean, avoids persistent fingerprinting across boots, and prevents setuid execution from that location.

5. **Guest userspace helpers (behavioral contract)**

   5.1. **Boot-completion notification.** The first userspace stage after the kernel hands off to PID 1 can be a tiny wrapper that memory-maps a single page at the **guest-physical MMIO address** reserved for the boot-timer device, writes a fixed byte value, asynchronously syncs the mapping, and immediately execs the real init. That signals the VMM that the guest reached userspace; the host can stop timers and assert latency budgets. The MMIO base differs between x86-64 and AArch64 to match each architecture’s memory map.

   5.2. **Memory pressure and OOM.** One helper forks a child that allocates anonymous memory in **one-megabyte chunks** until the parent’s child exits—either successfully or via OOM killer. The parent writes a short message to a temporary file describing whether the child was killed by signal or completed normally. Granular allocation avoids a single huge mapping that might fail early under pressure.

   5.3. **Balloon and confidentiality checks.** Another helper allocates a large anonymous region and scans it as an array of 32-bit integers, looking for **four consecutive occurrences** of a given value. Integration tests pair this with the memory filler: after a balloon deflates and returns memory to the host, the test searches for leftover data patterns to validate that **scrubbing or reuse** semantics match expectations.

   5.4. **Userfaultfd and snapshot restore.** A dedicated helper maps a large anonymous region (on the order of hundreds of megabytes), touches every page once, then blocks waiting for a user-defined signal. After a snapshot is taken and restored, the test sends that signal; the helper then rewrites every page and measures elapsed time using a monotonic boot clock, writing the duration in nanoseconds to a temporary file. The pattern forces **fast page faults** across the whole region after restore—exercising lazy memory and migration paths.

   5.5. **AArch64 instruction-level probe.** On that architecture only, a helper opens the physical memory device, maps a small window around the legacy BIOS area, and executes a **load instruction with post-increment** that reads past the mapping’s end. The intent is to surface **kernel-level** handling (e.g. `ENOSYS` or fault behavior) when user code performs a specific addressing pattern—useful for validating paravirtual or emulation quirks.

6. **Kernel configuration strategy**

   6.1. Baseline configurations are **imported wholesale** from Amazon Linux kernel builds for specific microvm-oriented versions. They encode a full Linux feature set appropriate for the vendor’s distribution, not a hand-minimized microkernel.

   6.2. A **CI overlay fragment** applies cross-cutting choices: exposing the running configuration via `/proc`, enabling **MS-DOS partition tables** and **Zstandard SquashFS** support, turning on **direct physical memory access** (`/dev/mem`) for tests that need MMIO-style probes, and enabling keyboard and PS/2-style input paths so **Ctrl+Alt+Del** handling exists where the architecture uses those drivers. Some errata toggles are explicitly disabled for AArch64 compatibility with the test matrix.

   6.3. **Debug variants** add ftrace tracers (function, graph, preempt, IRQ-off, scheduler, block I/O tracepoints, syscall tracing), function profiler support, and DWARF debug info with frame pointers—trading binary size and build time for observability.

   6.4. **Design tradeoffs** documented in the disclaimer: the kernel is tuned for **high density** (many small guests on one host). That implies aggressive virtual address space limits on AArch64, which can break software that assumes a larger user virtual address space (for example tooling that expects 48-bit virtual addresses). The tradeoff is intentional for ephemeral workloads and must be understood when porting guest workloads.

7. **Seccomp policy architecture**

   7.1. The monitor runs multiple classes of threads with different responsibilities. The policy bundle expresses **three separate profiles**—one for the main VMM thread group, one for the **API server** thread, and one for **vCPU worker** threads. Each profile is a **default-deny** list: any syscall not explicitly allowed causes the syscall to **trap** (fail) rather than silently allowing unknown behavior.

   7.2. Within each profile, a **positive filter** enumerates **allowed syscalls**. Many entries are unconditional; others attach **argument constraints** so that only specific `ioctl` command numbers, `fcntl` operations, or socket flags are permitted. This pattern is essential for `ioctl`, which multiplexes many unrelated operations through one syscall number.

   7.3. **vCPU profile** focuses on KVM and CPU affinity: `ioctl` allowances map to KVM run, register access, IRQ routing, TSC clock, KVM clock control after vCPU pause, and related controls. The intent is to let the KVM loop run while blocking unrelated device `ioctl`s.

   7.4. **API profile** covers networking and HTTP-style serving: socket creation, accept with `CLOEXEC`, TLS-related reads/writes, epoll, timerfd, and signal handling needed for graceful shutdowns.

   7.5. **Main VMM profile** is the broadest: memory mapping, file I/O, eventfd, io_uring (for storage path experiments), virtio-net tap I/O vectors, vsock, timers, futex, and signal masks for Rust runtime panics. Comments in the policy (not reproduced here) tie many syscalls to concrete subsystems—snapshot persistence, drive hotplug, RNG, metrics.

   7.6. A **stub profile** exists for platforms or build modes where the policy is not yet specialized: it **allows everything** but is structured so tooling can still attach filters—useful during bring-up or when syscall usage is still being enumerated.

   7.7. Operationally, **deny-by-default** reduces attack surface and makes accidental reliance on new libc or Rust runtime syscalls visible during development (the process crashes or logs a seccomp failure rather than silently expanding capability).

```
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  VMM threads │     │  API thread  │     │ vCPU threads │
  │  (devices,   │     │  (HTTP API,  │     │  (KVM run    │
  │   io_uring,  │     │   sockets)   │     │   loop)      │
  │   snapshot)  │     │              │     │              │
  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
         │                    │                    │
         v                    v                    v
   seccomp profile A      profile B            profile C
   (broad allow-list    (network + TLS +     (KVM ioctl
    with ioctl filter)  epoll + timers)      allow-list)
```

8. **Cross-cutting concerns**

   8.1. **Reproducibility:** manifests and saved kernel `.config` files make the artifact self-describing.

   8.2. **Separation of concerns:** guest images encode “what runs inside” tests; seccomp JSON encodes “what the host OS may do on behalf of the VMM.”

   8.3. **Testability:** MMIO boot signals, deterministic networking, memory stress tools, and balloon verification helpers are all **cooperative**—they assume a harness that can drive signals, snapshots, and device configurations in lockstep.

   8.4. **Security posture:** reduced services in the guest, no unprivileged BPF, syscall filtering on the host, and documented kernel limits for density all reinforce a **minimal privilege** stance appropriate for CI and microVM workloads rather than general-purpose servers.
