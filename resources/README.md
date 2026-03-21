# Firecracker continuous-integration resource bundle

1. **Purpose and scope**

   1.1. The bundle supplies everything needed to **build and describe** the software artifacts that automated tests use when exercising the virtual machine monitor: compressed guest root filesystems, minimal early-boot images, and Linux kernels compiled from vendor sources with layered configuration. It also ships **syscall restriction policies** that constrain how different threads inside the monitor process may interact with the host kernel.

   1.2. The design optimizes for **repeatable CI**: the same inputs (base distribution, configuration fragments, overlay content, compiler flags) should yield inspectable outputs (package manifests, kernel configuration snapshots, binary images) so failures can be bisected to either the hypervisor, the guest image, or the policy layer.

   1.3. Two parallel concerns run through the bundle: **guest-side behavior** (what runs inside the microVM to make tests possible—network bring-up, timing signals to the host, memory stress tools) and **host-side behavior** (how the VMM’s threads are allowed to call into the kernel). Both are often exercised together in integration tests.

   1.4. **Downstream consumption**: host orchestration downloads version-scoped guest binaries from object storage into a build-scoped image cache, then post-processes them for SSH keys and writable disks. Those steps assume the artifacts described here remain semantically stable across minor versions unless intentionally version-bumped.

2. **Guest root filesystem pipeline (deep view)**

   The rootfs path is a **multi-stage factory**: native helper compilation, overlay fusion, privileged base-distribution customization, filesystem export, compression, manifest capture, and cleanup. Kernels and initramfs follow related but separate tracks.

   2.1. **Host preparation before any guest work**

      2.1.1. The rebuild driver ensures Debian packages exist for compilers, flex and bison, libssl, squashfs, static busybox, cpio, curl, patch, and a container runtime. This is intentionally broad because later stages compile kernels, run nested containers, and assemble initramfs cpio archives.

      2.1.2. An **architecture-named output directory** is created as a dedicated sibling of the working tree so x86-64 and AArch64 artifact trees never collide on multi-arch builders.

   2.2. **Native helper compilation stage**

      2.2.1. Small C sources living under the overlay tree are compiled with the host GCC into fixed destination names (boot notification, memory filling, userfaultfd timing, optional AArch64 physical-memory probe). This happens **before** the main rootfs assembly so the overlay carries fresh binaries.

      2.2.2. One helper is gated on AArch64 only because a specific instruction-level test needs it; other helpers build on every architecture.

   2.3. **Overlay seeding**

      2.3.1. The overlay subtree is copied wholesale into a working directory. It carries pre-authored systemd units, network bootstrap scripts, mount units for tmpfs-backed runtime state, and the freshly compiled helpers.

   2.4. **Privileged base-image stage (nested container)**

      2.4.1. A **container daemon** is started in-process for the rebuild environment when object-store–minimal base tarballs are not used directly; the script waits until the daemon socket responds.

      2.4.2. A **minimal Ubuntu cloud image** base is pulled from a public registry as the inner root. The working tree is bind-mounted read-write with a working directory set to the resource tree.

      2.4.3. Inside that container, a **chroot customization script** runs as the main non-interactive entry. It installs a curated package set (udev, systemd, OpenSSH server, iproute2, curl, socat, Python minimal, iperf3, ping, fio, kmod, tmux, hwloc, editor, trace-cmd, linuxptp, strace, boto3, plus x86-only MSR and CPUID tools), sets hostname, clears the root password, configures serial getty for **root autologin** on the console Firecracker attaches, enables the custom networking unit, disables resolver and time-sync daemons, enables tmpfs-backed temporary directories and the systemd state tmpfs mount, strips documentation and locale trees, and appends a sysctl line to disable unprivileged BPF. It emits a **dpkg manifest** into root’s home for traceability.

      2.4.4. After the inner script, selected top-level directories are **tar-streamed** from the ephemeral container root into the working rootfs directory: userland binaries, libraries, configuration, home and root content. Empty mountpoint directories are created for dev, proc, sys, run, tmp, and package database state so the tree is bootable when combined with kernel devices.

      2.4.5. **Cloud command-line tooling** is installed from an upstream bundle into a local prefix inside the tree so tests can upload logs when credentials exist in the environment.

      2.4.6. **Resolver configuration** is replaced with a minimal stub because tests do not assume working upstream DNS.

      2.4.7. The manifest is moved out to the output directory with a basename matching the rootfs flavor; the populated tree is compressed with **Zstandard-backed squashfs** preserving root ownership semantics.

      2.4.8. **Cleanup** removes compiled helper binaries from the overlay source so the next run recompiles from pristine sources; ephemeral daemon logs are removed.

   2.5. **Initramfs track (parallel small artifact)**

      2.5.1. A minimal directory layout is created with busybox providing `sh` and `mount`.

      2.5.2. An `init` shell script mounts devtmpfs, proc, and sysfs, writes a magic byte to a **fixed guest-physical MMIO address** (one constant on x86-64, another on AArch64), rewires console file descriptors, prints uptime, and drops to an interactive shell. The MMIO write implements the **boot handshake** with the VMM’s synthetic boot-timer device.

      2.5.3. The tree is packed into a **newc cpio** archive written beside the squashfs outputs.

   2.6. **Kernel track**

      2.6.1. A long-lived vendor Linux fork is cloned shallowly if missing.

      2.6.2. **Tag selection** prefers microvm-specific kernel tags for the requested baseline minor version; if none match, it falls back to generic kernel tags for that minor line. The newest matching tag wins.

      2.6.3. **Configuration concatenation** merges multiple fragments in order; later fragments override earlier keys. A cross-cutting CI fragment enables squashfs with Zstd, partition table support, direct physical memory access where tests probe hardware, and input paths for legacy reboot signaling; architecture-specific errata toggles appear in AArch64-focused fragments.

      2.6.4. **Build targets** differ: ELF `vmlinux` on x86-64, PE `Image` on AArch64. Outputs are renamed to include a normalized version string plus optional **flavour suffixes** (for example variants without ACPI on x86-64).

      2.6.5. **Debug builds** add ftrace and DWARF fragments, output into a nested debug directory, then may split debug information, strip the executable, add a GNU debug link, and gzip the detached symbols for storage efficiency.

   2.7. **Orchestration modes**

      2.7.1. A default mode builds rootfs plus the standard kernel matrix.

      2.7.2. A rootfs-only mode skips kernel cloning.

      2.7.3. A kernel-only mode accepts optional version selectors to reduce iteration time when tuning Kconfig fragments.

3. **Guest root filesystem pipeline diagram**

```
  +------------------+
  | install host     |
  | deps + mkdir out |
  +--------+---------+
           |
           v
  +------------------+      +---------------------+
  | compile overlay  |      | start nested docker |
  | C helpers        |      | (if using base img) |
  +--------+---------+      +----------+----------+
           |                             |
           v                             v
  +------------------+
  | copy overlay     |
  | into workdir     |
  +--------+---------+
           |
           v
  +------------------+
  | privileged inner|
  | image: chroot   |
  | script installs |
  | packages + cfg  |
  +--------+---------+
           |
           v
  +------------------+
  | tar export dirs  |
  | + AWS CLI bundle |
  | + mountpoints    |
  +--------+---------+
           |
           v
  +------------------+      +------------------+
  | mksquashfs zstd  |      | build initramfs  |
  | + move manifest  |      | cpio + MMIO init |
  +------------------+      +------------------+
           |
           v
  +------------------+
  | clone + tag      |
  | vendor kernel    |
  | cat Kconfigs     |
  | make + rename    |
  +------------------+
```

4. **Guest root filesystem: policy and personality**

   4.1. The guest is intentionally **minimal but serviceable**: core userspace, init system, SSH server, networking utilities, lightweight scripting, performance tools, tracing, and hardware introspection where the architecture allows. Architecture-specific packages are added only on x86-64 (model-specific register and CPUID tooling).

   4.2. **Authentication** is relaxed for automation: the superuser password is cleared where appropriate, and serial console autologin avoids blocking tests that wait for a shell.

   4.3. **Network time and resolver daemons** are disabled by unlinking unit symlinks, reflecting an environment where **no real upstream DNS or NTP** is assumed; tests supply their own notion of time and naming.

   4.4. **Temporary storage** uses tmpfs for `/tmp` and for systemd’s mutable state path, reducing wear on backing disks and matching ephemeral workloads.

   4.5. **Documentation and locale trees** are deleted after package installation to shrink size and attack surface.

   4.6. A sysctl knob **disables unprivileged BPF program attachment**, closing a speculative-execution-related class of issues at the cost of forbidding unprivileged eBPF in the guest.

5. **Overlay: systemd integration and virtual networking bootstrap**

   5.1. A **oneshot service** runs early in boot and executes a shell script that enumerates every non-loopback network interface, discovers its Ethernet address, and derives an **IPv4 address** from the last four octets of a vendor-specific prefix baked into the address. The script assigns a `/30` address to each interface and brings the link up. Given MAC allocation rules in tests, the guest lands on predictable addresses without DHCP.

   5.2. Ordering ensures **secure shell** starts after addresses exist so tests can connect immediately over the tap or namespace the harness provides.

   5.3. A **mount unit** replaces persistent runtime state under the init system with a bounded tmpfs (`nosuid`, `nodev`, sticky world-writable mode) so the read-only root stays clean and setuid execution from that location is avoided.

6. **Guest userspace helpers (behavioral contract)**

   6.1. **Boot-completion notification.** The first userspace stage after the kernel hands off to PID 1 may memory-map a single page at the guest-physical MMIO address reserved for the boot-timer device, write a fixed byte value, asynchronously sync the mapping, and exec the real init—signaling the VMM that the guest reached userspace for latency assertions.

   6.2. **Memory pressure and OOM.** One helper forks a child that allocates anonymous memory in **one-megabyte chunks** until the child exits—successfully or via OOM. The parent records the outcome in a small temporary file.

   6.3. **Balloon and confidentiality checks.** Another helper scans memory for repeated patterns after balloon operations to validate scrubbing semantics.

   6.4. **Userfaultfd and snapshot restore.** A helper maps a large anonymous region, touches every page, blocks on a signal, then after snapshot restore rewrites every page and records elapsed time—exercising fast page faults across the region.

   6.5. **AArch64 instruction-level probe.** On that architecture only, a helper maps a window around legacy BIOS space and executes a load with post-increment past the mapping end to surface kernel handling of edge addressing patterns.

7. **Kernel configuration strategy**

   7.1. Baseline configurations are **imported wholesale** from Amazon Linux kernel builds for microvm-oriented versions—a full distribution feature set, not a hand-minimized microkernel.

   7.2. A **CI overlay fragment** applies cross-cutting choices: exposing configuration via procfs, enabling MS-DOS partition tables and Zstandard squashfs, exposing `/dev/mem` for MMIO-style probes, and keyboard or PS/2 paths for legacy reboot signaling where applicable.

   7.3. **Debug variants** add ftrace tracers, profilers, and DWARF with frame pointers.

   7.4. **Density tradeoff**: the kernel may enforce aggressive virtual address space limits on AArch64 to favor many small guests; software assuming a larger user virtual address space may break—an intentional tradeoff for ephemeral CI workloads.

8. **Seccomp policy architecture (triple-model)**

   Host syscall policy is not a single global allow-list. The monitor maps **three independent profiles** to three thread families, each serialized as its own JSON object inside a target-triple document. A separate **unimplemented** document provides a permissive stub for bring-up when specialization is incomplete.

   8.1. **Why three profiles instead of one**

      8.1.1. Threads that run the KVM vCPU loop perform a tight sequence of KVM ioctl-driven operations and should not need socket or HTTP stack syscalls. Threads that serve the control API must create sockets, accept connections, and drive TLS without ever needing the full KVM ioctl surface. Threads that coordinate devices, snapshots, virtio, and metrics combine file I/O, memory mapping, io_uring, vsock, and signal handling at a much broader level than either of the other two.

      8.1.2. Splitting policies **shrinks each allow-list** to the minimal surface for that role, reducing accidental syscall exposure if a library pulls in a new interface on only one thread type.

   8.2. **Per-profile JSON shape**

      8.2.1. Each profile object carries a **default action** of trap (syscall denied causes a synchronous fault the process can observe) and a **filter action** of allow for matching rules—classic seccomp-bpf “deny by default, allow explicitly.”

      8.2.2. The **filter** array lists syscall names. Many entries are unconditional allows. Others attach **argument comparators** restricting ioctl command numbers, fcntl commands, socket families, or flag combinations—essential because ioctl multiplexes unrelated driver operations behind one syscall number.

      8.2.3. **Comments** inline document engineering rationale (which subsystem triggered the need, which Rust std path exercised the syscall). Comments are not loaded into the kernel; they are for human audit and codegen.

   8.3. **VMM-oriented profile (broadest)**

      8.3.1. Covers memory management (mapping, protection changes, advice), file descriptors and regular file I/O, epoll and eventfd, io_uring submission paths for storage experiments, virtio-net vector I/O, vsock, timers, futex, and signal masks for runtime abort paths.

      8.3.2. ioctl rules are the largest category: block device topology, tap/tun setup, KVM ancillary ioctls not covered in the vCPU profile, and housekeeping ioctls needed for device models and metrics.

   8.4. **API-oriented profile**

      8.4.1. Emphasizes **network-facing** syscalls: socket creation, accept with close-on-exec, TLS read/write paths, epoll, timerfd, and signal handling for graceful shutdown.

      8.4.2. Deliberately omits KVM vCPU ioctls so a compromised API worker cannot pivot into CPU emulation primitives without already satisfying the broader VMM profile on other threads.

   8.5. **vCPU-oriented profile**

      8.5.1. Focuses on **KVM run loop** requirements: ioctl allowances for run, register access, IRQ routing, clock sources, and controls needed after vCPU pause—mapped narrowly so unrelated device ioctls fail.

      8.5.2. Complements the VMM profile: device setup happens elsewhere; vCPU threads stay in their lane.

   8.6. **Stub profile**

      8.6.1. A JSON document may define an **allow-all** stance while retaining structural compatibility so tooling can attach filters during early porting or when enumerating syscall usage before tightening lists.

   8.7. **Per-target artifacts**

      8.7.1. Separate JSON documents exist for x86-64 and AArch64 musl targets because syscall numbers and needed ioctl subsets differ; build integration selects the matching document when compiling filters into the monitor binary.

   8.8. **Operational consequence**

      8.8.1. When Rust, libc, or third-party code introduces a new syscall on a monitored thread type, the process fails closed—developers see the trap rather than silently gaining capability. That turns policy drift into an explicit edit to the JSON followed by regeneration.

9. **Seccomp triple-model diagram**

```
     +----------------+     +----------------+     +----------------+
     | VMM-coordinator|     | API server     |     | vCPU workers   |
     | threads        |     | threads        |     | (KVM run loop) |
     +-------+--------+     +-------+--------+     +-------+--------+
             |                      |                      |
             v                      v                      v
      +-------------+        +-------------+        +-------------+
      | Profile W:  |        | Profile A:  |        | Profile V:  |
      | wide file + |        | sockets +   |        | narrow KVM  |
      | mm + io_uring|        | TLS + epoll |        | ioctl allow |
      +-------------+        +-------------+        +-------------+
             \                      |                      /
              \                     |                     /
               \                    |                    /
                v                   v                   v
             +-----------------------------------------------+
             |  seccomp BPF loaded per thread / clone flags   |
             |  default: trap on unknown syscall               |
             +-----------------------------------------------+
```

10. **Cross-cutting concerns**

    10.1. **Reproducibility:** manifests and saved kernel configuration snapshots make artifacts self-describing.

    10.2. **Separation of concerns:** guest images encode what runs inside tests; seccomp JSON encodes what the host OS may do on behalf of which monitor threads.

    10.3. **Testability:** MMIO boot signals, deterministic networking, memory stress tools, and balloon verification helpers are **cooperative**—they assume a harness that drives signals, snapshots, and device configurations in lockstep.

    10.4. **Security posture:** reduced services in the guest, no unprivileged BPF, syscall filtering by thread role, and documented kernel limits for density reinforce **minimal privilege** appropriate for CI and microVM workloads.

11. **Relationship to CI and automation hooks**

    11.1. Changes under the seccomp JSON subtree are treated as **performance- and security-sensitive** in upstream automation: post-merge hooks may schedule statistical performance comparisons when those files change alongside Rust sources, because syscall policy affects hot paths and binary layout.

    11.2. Guest artifacts rebuilt from this bundle are published alongside minor Firecracker versions; the host-side orchestration downloads that scope so tests always evaluate a coherent kernel, rootfs, and hypervisor triple.
