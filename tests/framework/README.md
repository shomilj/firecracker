# Python integration test framework

1. **Purpose and scope**

   1.1. The Python integration test framework is the programmatic layer that drives Firecracker end-to-end in automated tests: it launches the VMM under the jailer, configures guests through the HTTP control plane, attaches storage and networking, exercises lifecycle operations (boot, pause, snapshot, restore), and tears everything down deterministically. It is not a standalone product; it is the glue between pytest (or similar runners), host tooling (network namespaces, SSH, process control), and the Firecracker binaries.

   1.2. Design goals include: strict validation of API outcomes (failures surface as test failures with rich context), reproducible isolation per VM (separate network namespaces and chroots), minimal surprise around asynchronous startup (polling and retries at well-defined boundaries), and optional observability hooks (logs, metrics, memory sampling, latency checks) that can be turned off when they would add noise (for example under parallel test workers).

2. **Layered architecture**

   2.1. At the highest level, tests obtain a **factory** that knows where release-quality or workspace-built `firecracker` and `jailer` binaries live. The factory mints **microVM handles**, each with a unique identity, an associated **jailer context** (chroot layout, UID/GID, daemonize vs. interactive mode, cgroup hooks), and a **network namespace** for TAP devices and SSH to the guest.

   2.2. Once spawned, each microVM exposes an **HTTP API client** bound to the Unix domain socket inside the jail. That client is organized as REST-shaped resources (machine configuration, boot source, drives, network interfaces, VM actions, snapshots, optional entropy, balloon, vsock, MMDS, etc.). The client uses HTTP over Unix sockets with a connection pool sized to stay just under the server’s concurrent-connection limit, avoiding a class of “server full” races when connections churn.

   2.3. A **helpers** surface (conceptually separate from the core lifecycle object) supports interactive debugging: how to SSH, how to attach debuggers, how to enable serial consoles via `screen`, and ad-hoc networking tricks (for example bridging a guest TAP to the wider host for ingress). These helpers mutate launch parameters (for example disabling daemonization) and are meant for developer workflows more than CI determinism.

   2.4. Cross-cutting utilities implement **host command execution** with consistent logging, **CPU topology mapping** for containers, **process lifetime** helpers (pidfd-based wait), **screen** orchestration for console tests, and small **MMDS** curl builders. Additional focused modules cover vhost-user block backends, CPU template naming, IMDS-like HTTP from guests, and optional ftrace/iperf/vsock conveniences—each extending the core without bloating the central object.

3. **Jailer integration and filesystem view**

   3.1. Each microVM’s jailer context captures everything needed to construct a correct command line: identity string, executable path, UID/GID, chroot base, optional network namespace path, whether to detach as a daemon, whether to create a new PID namespace, cgroup-related parameters, and an open-ended map of Firecracker-specific flags that appear after `--` in the jailer invocation.

   3.2. The **chroot layout** follows the jailer’s contract: a per-VM directory under the chroot base, containing a `root` tree where linked artifacts live. The context exposes the full host path to the API socket (defaulting under `run/` inside the jail) and the path to the PID file written for the jailed Firecracker process.

   3.3. **Resource publication** into the jail is implemented by hardlinking (when source and jail are on the same device) or copying (when they are not), optionally creating block special nodes, then `chown`ing to the jail UID/GID. The returned path is always the **in-jail** absolute path (string beginning with `/`) suitable for API payloads. This keeps tests honest about what the VMM actually sees while avoiding unnecessary duplication when hardlinks work.

   3.4. **Cleanup** walks cgroup controllers associated with the VM when configured: it waits until tasks files are empty (with bounded retries), then removes the per-VM cgroup directories. This complements test-level `SIGKILL` of Firecracker by reducing leftover kernel cgroup state.

4. **Process launch modes and readiness**

   4.1. Two launch paths exist: **daemonized** jailer (Firecracker fully detached; tests talk only to the API socket and logs) and **screen-wrapped** jailer (Firecracker’s stdio attached to a `screen` session so serial console traffic lands in a log file). A new PID namespace interacts with the screen path: Firecracker may reparent such that the screen supervisor exits; the framework clears the remembered screen PID to avoid killing an unrelated process.

   4.2. Readiness is established in layers:

   - If a JSON **config file** path is present in the extra arguments and at least one network interface was configured, the framework waits for **guest SSH** (see below) because that implies userspace is far enough along for most tests.

   - Else, if the API is enabled (no `no-api` flag), and logging is verbose enough, it waits for a log line stating the API server started; if logging is too quiet for that, it polls for the API socket file with retries.

   - Else, if logging still provides boot messages, it looks for a generic “running” line; otherwise the caller is responsible for synchronization.

   4.3. **Optional NUMA binding** is achieved by prepending `numactl` (or similar) command fragments ahead of the jailer invocation, so the entire jailer/Firecracker stack inherits policy without the Python code needing to know VMM internals.

5. **HTTP control plane client**

   5.1. The client wraps a small resource abstraction: each resource knows its path, whether it carries an identifier in the URL (for example drives keyed by id), and how to issue `GET`, `PUT`, or `PATCH`. Non-204 responses deserialize JSON error bodies and raise with the fault string when present—tests therefore fail loudly on unexpected API errors instead of silently proceeding.

   5.2. The Unix socket session mounts a dedicated adapter with a **bounded connection pool** (one less than the server’s advertised maximum). The rationale is subtle: when the pool evicts connections, close and open events can reorder at the kernel level; keeping headroom avoids spurious “server full” conditions during bursty test traffic.

   5.3. The top-level API object also wires an **error callback** used by the microVM to dump diagnostics whenever a low-level request exception occurs (socket issues, deserialization problems). That ties transport failures to the same debug bundle as SSH failures.

6. **Guest configuration shortcuts**

   6.1. A “basic configuration” path applies machine settings (vCPU count, SMT flag, memory, optional dirty-page tracking for differential snapshots, huge-page policy), records the intended memory size for later metrics labels, optionally starts a memory sampling helper, then configures the boot source (kernel path, optional initrd, command line). Root filesystem attachment is optional but typical: read-only mode is inferred from squashfs images.

   6.2. **CPU templates** support both named static templates (string identifiers) and structured custom templates posted to the dedicated CPU configuration endpoint. Template naming is normalized for telemetry dimensions.

   6.3. **Block devices** can be file-backed (linked into the jail) or **vhost-user** sockets. For vhost-user, the framework spawns an external backend process, waits until its socket appears, `chown`s the socket into the jail, and registers the drive with the socket form of the API. Replacing a vhost-user drive id kills the previous backend first to avoid orphaned processes.

7. **Networking and guest access**

   7.1. Each TAP is created inside the microVM’s network namespace with a host IPv4 address and prefix derived from a structured interface description object (guest MAC, guest IP, TAP name, etc.). The framework stores both the high-level config and the TAP handle for later snapshot restores and SSH.

   7.2. **SSH** uses a persistent connection object bound to the namespace, the private key copied alongside the rootfs, a control master socket path colocated with the chroot, and the same diagnostic callback as the API client. The first access to a given interface index constructs and caches the connection; the framework tracks all open connections to close them before killing Firecracker.

   7.3. **Waiting for SSH** is the default “guest is usable” gate after `InstanceStart` when interfaces exist. It ensures initialization completed, not merely that the VMM thread scheduled.

8. **Snapshots: data model and operations**

   8.1. Snapshots are first-class immutable bundles: vmstate file, memory image, captured disk map (ids to paths), list of network interface configs, ssh key material, snapshot **kind** (full vs. differential variants), and arbitrary metadata (kernel path, vCPU count, etc.).

   8.2. **Kinds** differ in external requirements:

   - Full snapshots are self-contained memory images.

   - Differential kinds require dirty page tracking to have been enabled before taking them, and may require **rebasing** incremental memory files onto a base layer using either a dedicated rebase binary or a snapshot editor tool—tests pick the mechanism explicitly.

   - One differential variant pairs with **mincore** behavior at the hypervisor level; the same API string maps to distinct enum values locally so tests can choose semantics precisely.

   8.3. **Creation** pauses the VM first (to quiesce devices), then calls the snapshot creation endpoint with on-disk paths relative to the jail. Returned object paths are anchored at the chroot root on the host.

   8.4. **Serialization** to a directory copies or hardlinks vmstate, memory, ssh key, and disks, then writes a JSON manifest mapping logical names to filenames plus all structured fields. Deserialization rebuilds the snapshot object for restore pipelines.

   8.5. **Restore** copies snapshot files into a fresh jail with distinct basenames to avoid clobbering golden artifacts, recreates TAP devices from saved interface objects (initially without issuing API calls), hardlinks disks, then posts a load request. Memory backends are either plain files or **userfaultfd** sockets: when the latter, an external page-fault handler binary is published into the jail, executed inside the chroot with dropped privileges matching the jail user, and its socket is wired as the memory backend. Optional **network override** maps allow renaming host-side TAP devices for compatibility across baseline vs. candidate binaries in A/B comparisons.

9. **Userfaultfd restore path**

   9.1. The handler process runs jailed, logs to a dedicated file, chmods the binary executable inside the chroot, and listens on a fixed socket path. Firecracker connects as a client; the framework ensures the socket is owned by the jail UID/GID after creation.

   9.2. Failure to start fails fast with log contents printed to the console stream, because silent hangs here mask root causes (bad snapshot path, seccomp issues, etc.).

10. **Factories and multi-VM snapshot scenarios**

    10.1. The factory remembers every VM it created so a single `kill` sweep tears down all instances: stop monitors, close SSH, terminate vhost-user backends, `SIGKILL` Firecracker and any lingering screen supervisor, wait on pidfds, optionally validate no stray jailer-id processes remain, kill UFFD handlers, validate API latency if enabled, and finally `rmtree` the chroot if safe.

    10.2. Building from a snapshot spawns a fresh VM and immediately loads the snapshot with resume, optionally with UFFD.

    10.3. A **generator** pattern can restore many VMs from the same snapshot or walk an incremental chain: each iteration builds a VM, restores, yields it to the test, optionally takes the next snapshot (rebasing differentials as needed), deletes superseded files to conserve disk, kills the VM, and finally deletes the last snapshot artifacts. This supports stress and longevity scenarios without manual bookkeeping.

11. **Serial console testing**

    11.1. When not daemonized, Firecracker output goes to a `screen` log file. A small **serial** helper polls that file with `select`, sends keystrokes via `screen`’s `stuff` command, and reads either character-by-character or line-delimited with a timeout that fails the test and kills the VM on excessive delay.

    11.2. A companion **state machine** layer matches incremental input against target strings character-wise, enabling scripted boot menus or firmware interactions without fragile full-string compares on bursty console IO.

12. **Observability and guardrails**

    12.1. **Logging**: if enabled, Firecracker logs to a host-visible file also linked into the jail. Log level interacts with optional **API latency assertions**: only debug-level logs carry the paired “request received” / “total duration” lines the framework correlates. Those assertions parse the log stream sequentially (requests are processed synchronously, so ordering is stable), convert microseconds to milliseconds, and enforce a hard ceiling except for intentionally long operations like snapshot endpoints.

    12.2. **Parallel pytest workers** disable API latency checks by default because timing becomes noisy when the host is contended—environment variables signaling `xdist` worker counts flip that switch.

    12.3. **Metrics**: optional newline-delimited JSON metrics files can be linked and flushed via an API action; helpers iterate valid JSON lines defensively (tolerating partial trailing writes).

    12.4. **Memory monitors** attach when requested and validate collected samples on shutdown—useful for leak or ballooning regressions.

    12.5. **Debug dumps** on failures bundle Firecracker logs, optional UFFD logs, and **thread stack traces** scraped from `/proc` for every Firecracker thread—unless the VM was already marked dead cleanly.

13. **Thread and CPU controls**

    13.1. For performance isolation experiments, helpers enumerate threads by name via `psutil`, then set affinities using a **CPU map** that translates logical indices to the actual host CPUs visible to cgroups—critical in Docker where `/proc/cpuinfo` lies about available cores.

    13.2. Convenience methods pin all vCPUs to consecutive cores, then place VMM and API threads on the next two cores, returning the next free core index for additional host workloads.

14. **Artifacts, versioning, and pytest parametrization**

    14.1. Guest kernels and root filesystems are discovered under a machine-specific artifact directory (with a documented fallback when shared storage is absent). Kernels are filtered by regex allowlists so special variants (for example ACPI-less builds) remain opt-in.

    14.2. Firecracker release binaries are globbed, parsed for semantic versions, capped to the workspace’s next minor boundary, and exposed as pytest parameters. The **current workspace build** is also advertised as an artifact by hardlinking the built `firecracker`/`jailer` to version-suffixed names that mirror release layout—letting performance and compatibility tests treat “tip” like a normal release.

    14.3. Snapshot **version strings** for a given binary can be queried via a dedicated flag when Firecracker decoupled snapshot formats from release numbers; tests use that to gate behavior across baselines.

15. **Global environment properties**

    15.1. A process-wide singleton captures immutable host metadata: CPU vendor, model name, codename, microcode, kernel version (several granularities), OS pretty name, libc and Rust toolchain versions, optional Buildkite identifiers, PR vs. non-PR mode, git commit/branch when available, and EC2 instance metadata when the environment responds to IMDSv2 probes.

    15.2. MicroVMs derive **CloudWatch-style dimensions** from this singleton plus per-VM configuration (guest kernel stem, rootfs name, vCPU count, memory size) for consistent metrics tagging in CI.

16. **A/B testing infrastructure**

    16.1. **Motivation**: some checks are inherently unstable against absolute baselines (security audit outputs drift as the advisory database changes) but still must catch PR-induced deltas. **A/B tests** run the same procedure twice—on a reference revision and on the candidate—and compare outputs with a pluggable comparator.

    16.2. **Git-based A/B** clones the repository to a temporary path for revision A (defaults to the PR base branch in CI, else `main`), runs the user-supplied callable there, runs again at the working tree for B, then compares. Cloning uses a short-lived local branch trick to materialize arbitrary commitishes without mutating the developer’s checkout.

    16.3. **Binary-directory A/B** skips git entirely: callers point one run at a directory of release-tagged binaries and the other at the default cargo output tree—ideal for snapshot compatibility across shipped artifacts vs. local builds.

    16.4. **Host command A/B** wraps shell pipelines: in PR CI, stdout/stderr must match across revisions; outside PRs, the command simply must succeed—keeping local workflows fast while preserving gating where it matters.

    16.5. **Set non-growth** comparators parse command output into sets and require the candidate’s set to be a subset of the reference’s set—useful for “no new vulnerable crates” style assertions.

    16.6. **Statistical regression testing** for performance exposes a permutation test over two sample populations, returning a full hypothesis-test object so callers can threshold p-values for noisy metrics.

17. **Concurrency control**

    17.1. Certain git and artifact operations are wrapped in **file locks** keyed only by callable name (a known caveat if two functions share a name). The lock serializes access across parallel pytest workers that would otherwise race cloning or hardlinking shared binaries.

18. **Static analysis helpers (seccomp introspection)**

    18.1. Separately, a static analysis module (invoked by specialized tests) disassembles binaries, tracks syscall numbers through simplified register backtracking, and compares observed syscalls against installed seccomp policies to flag redundant rules. This is heavy machinery kept isolated from the hot path of normal integration tests.

19. **End-to-end data flow (typical test)**

```
  +------------------+       build / select       +------------------+
  | pytest parameter | ---- binaries & rootfs ---> |     Factory      |
  +------------------+                            +--------+---------+
                                                           |
                    +----------------------------------------+
                    v
           +------------------+     jailer + netns     +------------------+
           |  MicroVM handle  | ---- spawn ---------> | Firecracker proc |
           +--------+---------+                       +--------+---------+
                    |                                          |
        configure   |  HTTP/UDS                                | vCPUs
        via API     v                                          v
           +------------------+                         +------------------+
           | Guest (SSH/serial)| <------ TAP -------- |     devices      |
           +------------------+                         +------------------+

   teardown: stop monitors -> close SSH -> kill vhost-user -> SIGKILL FC
             -> wait pidfd -> verify no stray PIDs -> optional latency parse
             -> cleanup cgroups/netns/chroot
```

20. **Behavioral invariants worth relying on**

    20.1. API calls that should succeed are checked strictly; there is no “best effort” silent fallback when the control plane errors.

    20.2. Snapshot operations that need the VM paused perform the pause explicitly—tests do not depend on implicit pausing side effects.

    20.3. Differential snapshot types that need rebasing encode that requirement in the type system so misuse raises before touching disks.

    20.4. Kill ordering prefers stopping **external backends** before Firecracker when both exist, avoiding socket races during teardown.

    20.5. When UFFD is enabled, memory backends switch from file paths to the userfaultfd socket transparently to the rest of the restore logic—only the backend descriptor changes.

21. **Extensibility model**

    21.1. New device types generally arrive as additional API resources plus optional host-side helpers (for example a new virtio backend process). The core microVM object should only gain thin forwarding logic; heavier orchestration belongs in focused utility modules.

    21.2. New CI dimensions should extend the global properties singleton cautiously—everything constructed there runs once per process and must remain side-effect free beyond probing.
