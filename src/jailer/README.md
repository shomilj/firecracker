# Jailer

1. **Role and threat model**

   1.1. A small, privileged bootstrap program starts the virtual machine monitor under production-style constraints: Linux namespaces, cgroups, filesystem isolation, and resource limits are established first; root-equivalent privileges are then dropped to configured non-root identities before the process image is replaced by the target binary. The design assumes a tightly bounded workload inside the jail and a host that may run many instances. The executable is **copied** into the jail rather than hard-linked so that distinct processes do not share executable text pages—important for certain memory-disclosure threat models.

   1.2. Work splits into **process hygiene** (strip inherited state that could leak across invocations), **configuration assembly** (validate identifiers, jail layout, cgroup and rlimit settings), and **ordered system calls** (network namespace attachment, rlimits, cgroups, pivot-based jail, device nodes, optional daemonization and PID namespace, then exec). Order is contractual: later steps assume earlier invariants; several steps are impossible to perform after the mount namespace pivot.

2. **Kernel prerequisites (hard assumptions)**

   2.1. Descriptor hygiene uses a single batch close of every open file descriptor above the standard three, with a flag that unshares the file descriptor table first so closure semantics match the intent to shed unrelated descriptors. This facility requires a sufficiently recent kernel; there is no fallback path—failure aborts the whole program before argument parsing. Operators must treat this as a **hard floor** on kernel version for production.

   2.2. The program assumes a single-threaded process during environment scrubbing; removing all inherited environment variables uses APIs that are only safe without concurrent access from other threads.

3. **Startup and process hygiene**

   3.1. Before any argument parsing, inherited surface area is reduced aggressively. The standard input, output, and error slots are left intact so orchestration can still wire logging and control channels; everything else is closed. This runs before clearing the environment so that a minimal, predictable descriptor table exists regardless of how the parent launched the process.

   3.2. Every environment variable inherited from the parent is removed in a loop. The rationale is to prevent accidental leakage of secrets, locale overrides, or library-controlled behavior into the jailed child. After this step, meaningful ambient process state is reduced to the program image, argv, and the three standard streams (which may be redirected later by optional daemonization).

   3.3. Argument parsing is strict: required fields include a stable instance identifier (validated against the project’s instance-ID rules), paths to the binary to exec, numeric user and group IDs to assume after isolation is prepared, and a configurable base directory for jail roots. Optional flags control network namespace attachment, cgroup version and properties, parent cgroup placement, per-process resource limits, daemonization, and whether to exec inside a new PID namespace. Text after a `--` separator is forwarded verbatim to the target binary.

   3.4. Help and version requests short-circuit before any filesystem mutation: they print usage or version and exit successfully.

4. **Jail path layout and executable handling**

   4.1. The chroot target path is derived by canonicalizing the jail base, appending the **basename** of the resolved executable path (not the full path), then the instance ID, then a fixed leaf segment naming the actual root seen inside the jail. That scheme colocates all instances that share the same binary name under one subtree while isolating each instance by ID. The outer process ensures this directory exists before proceeding.

   4.2. The executable file is **copied** into the jail root under its original basename. The copy is intentional: hard links fail across devices, and sharing text segments between monitor processes is considered undesirable for isolation. The destination path inside the jail becomes the argv[0] target for the final exec.

5. **Resource limits (rlimits)**

   5.1. Configurable limits map to classical Unix resource limit resources. File-size limits are optional; if omitted, no change is made beyond what the process already inherited. The open-file-descriptor limit always has an explicit value: a built-in default applies when the user does not override it, so every run establishes a known ceiling on descriptor count.

   5.2. Installation sets **both** current (soft) and maximum (hard) limits to the same target value. That closes the gap where a process could raise its soft limit back toward a higher hard limit; the goal is a single tight bound, not gradual enforcement. Parsing accepts only non-negative integer magnitudes; malformed tokens fail during configuration assembly rather than at syscall time.

6. **Cgroup integration — discovery and version split**

   6.1. **Discovery.** At initialization time the implementation reads the kernel’s line-oriented mount table to find cgroup v1 and v2 mount points. A regular expression distinguishes legacy `cgroup` mounts from unified-hierarchy `cgroup2` mounts and extracts mount directory and options. For v1, **multiple** mount points may exist (one per controller or combined hierarchies); each is cached for on-demand resolution. For v2, discovery records a **single** unified hierarchy root and stops after the first matching line. If the requested cgroup version’s mounts are absent, configuration fails early with a hierarchy error rather than at first write.

   6.2. **Mutual exclusion between versions.** The scan is version-specific: requesting v2 while only legacy mounts exist fails, and requesting v1 while only unified hierarchy exists fails. Mixed hosts are common in production transitions; misconfiguration surfaces immediately.

7. **Cgroup v1 — per-controller hierarchies**

   7.1. **Controller resolution.** Properties are expressed as `controller.property=value`. The controller name is the substring before the **first** dot; this deliberately supports dotted property names with multiple segments (for example memory controller knobs with several dot-separated parts). Each controller name maps to a hierarchy root by scanning cached mount options for that controller keyword. Unknown or unmounted controllers produce a clear “controller unavailable” error rather than silent omission.

   7.2. **Multiple controller objects.** Because legacy cgroup can split controllers across different hierarchy roots, the configuration may hold **separate** cgroup objects—one per controller that received at least one property. Each object knows its own leaf directory under `parent/id` within that controller’s mount tree.

   7.3. **Parent path rules.** The parent cgroup is a **relative** path segment (no leading slash, no `.` or `..` components). It defaults to the executable basename when not specified, yielding a per-binary grouping under the host’s cgroup tree. The leaf directory is always `parent` followed by `id` under the resolved hierarchy root for that controller.

   7.4. **Inheritance for empty parent pseudo-files.** Some controllers expose pseudo-files whose effective value must match parent and child (cpuset is the canonical example). When a new cgroup directory is created, a child pseudo-file can exist but be empty while the parent holds the authoritative line. A recursive helper reads the parent’s first line; if empty, it walks up the parent chain up to a bounded depth (derived from the number of components in the parent path), then copies the first non-empty line into the child before writing the user’s value. The algorithm tolerates concurrent jailers racing on the same parent: a failed write on a race may still leave the parent populated; a subsequent read picks up the value. Payloads are newline-terminated because many controllers expect a single line.

   7.5. **Per-property ordering inside v1 writes.** For each configured property, inheritance is applied **before** overwriting with the configured value, so the child is always in a valid state for controllers that require parent values to be non-empty first.

   7.6. **Two-phase application (all versions).** For each cgroup object, values are written first (creating directories as needed), then the process attaches. The split exists because some controllers require configuration ordering (for example cpuset memory and CPUs before membership). Attachment uses the legacy per-hierarchy membership file that accepts a PID.

   7.7. **ASCII — v1 mental model**

```
  [ mount table scan ]
           |
           v
  +---------------------------+
  | Cache v1 mount rows;      |
  | resolve controller ->     |
  | hierarchy root on demand  |
  +---------------------------+
           |
           v
  For each distinct controller in config:
  +---------------------------+
  | mkdir leaf: root/parent/id|
  | for each property:        |
  |   inherit if needed       |
  |   write property=value    |
  +---------------------------+
           |
           v
  +---------------------------+
  | For each cgroup object:   |
  |   attach PID (membership) |
  +---------------------------+
```

8. **Cgroup v2 — unified hierarchy**

   8.1. **Single tree.** All properties are written under the unified mount. One logical cgroup object holds the leaf path `unified_root/parent/id`.

   8.2. **Controller allowlist.** Available controllers are read once from the root’s controller list file. A property is accepted only if its controller (first segment before the dot) appears in that set; otherwise configuration fails with “controller unavailable.” This avoids silent no-ops when the host delegates a subset of controllers.

   8.3. **Enabling controllers along the path.** Before writing a leaf property for a controller, that controller must be enabled in every ancestor up to the root via the subtree-control pseudo-file (`+controller` form). The implementation walks **up** to the root first, then enables on the way back down so inner nodes inherit delegation. Each controller is enabled at most once per setup pass (tracked in a set).

   8.4. **Property writes.** After enabling, leaf values are written with newline-terminated payloads. Multi-dot property names are supported the same way as v1 (controller = first segment).

   8.5. **Attachment.** After all configured cgroup objects have written values, each attaches the current process by writing the PID to the unified membership file.

   8.6. **Two-phase application.** Same global pattern as v1: full write pass, then full attach pass, so ordering-sensitive controllers see a consistent filesystem state before membership.

   8.7. **“Join parent only” mode.** When no cgroup properties are supplied, unified hierarchy is selected, and a parent cgroup path is configured: if the parent directory **already exists** on disk under the unified root, the current process writes its PID into the parent’s membership file and **does not** create a new leaf. This supports deployments that pre-create a cgroup and only want the bootstrap to join it. If the parent does not exist, this branch does nothing—no implicit mkdir for join-only mode.

   8.8. **ASCII — v2 delegation walk**

```
  unified root
       |
       +--- [subtree_control +cpu +memory ...]
       |
  parent dir
       |
       +--- [subtree_control enables child knobs]
       |
  leaf (instance)
       |
       +--- cpu.max, memory.max, ...
       |
       +--- finally: attach process (unified membership write)
```

9. **Namespaces: which, when, and why**

   9.1. **Mount namespace (always for the jail path).** A new mount namespace is created for every run as part of the pivot sequence. This isolates subsequent mount operations from the host and from siblings; combined with recursive slave propagation from the root, it limits accidental propagation of mount events. The process does **not** enter new PID, UTS, IPC, or user namespaces unless separately requested (below).

   9.2. **Network namespace (optional, early).** If a netns handle is provided, it is opened as a file and the process joins that namespace using the network clone flag. This **must** occur before the mount-namespace pivot so that the correct interfaces, routes, and sockets remain visible for any later steps and for the child. Failure modes include missing files, wrong file types, or permission errors on open; the join step surfaces those as open or setns failures.

   9.3. **PID namespace (optional, late).** Optional execution in a new PID namespace uses a raw clone-style syscall with the new-PID flag and a null stack pointer—legal because the child does not share memory with the parent the way a threaded clone would. The child becomes PID 1 in the new namespace and replaces itself with the target binary. The parent records the child’s PID in a sibling file whose name is derived from the jailed executable basename with a fixed suffix, then exits successfully; orchestrators read that file to discover the monitor PID on the host.

   9.4. **Session and PID namespace interaction.** Kernel rules allow a parent in an ancestor PID namespace to signal the “init” of a child PID namespace only in specific circumstances; session leadership interacts with SIGHUP when the parent exits. If the bootstrap would be a session leader (session ID equals process ID), and daemonization is **not** used, the child in the new PID namespace creates a new session so the monitor does not receive a spurious hangup when the parent exits after recording the PID. Daemonization already uses a double fork that avoids session leadership, so that branch is skipped there.

   9.5. **Namespace ordering summary**

```
  [ optional: setns NET ]
            |
            v
  [ rlimits ]
            |
            v
  [ cgroups: write then attach ]
            |
            v
  [ optional: open host /dev/null for daemonize ]
            |
            v
  [ unshare MOUNT + pivot jail ]
            |
            v
  [ mkdir, devices, ... ]
            |
            v
  [ optional: double-fork daemonize ]
            |
            v
  [ optional: clone NEWPID + record PID ]
     or
  [ record self PID + exec ]
```

10. **Filesystem jail: mount namespace and pivot (not bare chroot)**

   10.1. The jail is **not** implemented as a bare chroot. The sequence is: create a new **mount namespace**; mark mount propagation as **slave** recursively from the root (so subsequent mounts do not propagate unexpectedly); **bind-mount** the intended jail directory onto itself (working around the pivot requirement that new and old root differ in mount topology); change the working directory into that directory; create a placeholder directory for the old root; **pivot** so the jail becomes `/` and the previous root is hidden under a temporary mount point; set working directory to `/` explicitly because pivot does not guarantee cwd semantics; **lazy-unmount** the old root and remove the empty directory.

   10.2. **Why bind-before-pivot.** The kernel requires the new root and the putative old root to be on **different** mounts in some configurations; binding the jail directory over itself creates a distinct mount at the same path, satisfying the constraint while leaving content unchanged from a logical perspective.

   10.3. **Relative pivot.** After bind and `chdir` into the jail directory, pivot uses **relative** paths (current directory and a created `old_root` subdirectory) so the operation does not depend on absolute path strings that will become invalid after the root switch.

   10.4. **Isolation strength.** Mount namespace plus pivot yields stronger isolation than chroot alone: the original host root is detached and unmounted from the process’s view (subject to kernel lazy-unmount semantics), and mount namespace boundaries reduce cross-talk with host mount events.

11. **Post-pivot content inside the jail**

   11.1. A fixed set of directories is created with restrictive permissions (typically owner-only) and ownership changed to the configured UID/GID. This includes the root, a device hierarchy, and a run directory commonly used for API sockets. Ownership walks use explicit per-path ownership calls because recursive ownership is not assumed from a single syscall.

   11.2. **Device nodes** are created with character device semantics for known static major/minor pairs: KVM, TUN/TAP, and the non-blocking random source. Ownership is immediately set to the jailed user. If random creation fails, a warning is printed because some guest features depend on entropy; the bootstrap continues—a degrade-gracefully policy for optional capabilities.

   11.3. **Userfaultfd.** The misc device minor number is not fixed; it is discovered at runtime by scanning the kernel’s misc device table for the userfaultfd entry. If present, a matching device node is created inside the jail with the dynamic minor; if absent, that feature is unavailable without failing the whole run.

   11.4. **Architecture-specific host facts (AArch64).** On ARM64 hosts, selected sysfs files describing CPU cache topology and CPU identification are copied as small text files into parallel paths under the jail root so the monitor can construct accurate CPU models without exposing all of sysfs. Each copied file is owned by the jailed UID/GID. This step is gated by architecture so other builds do not include it.

12. **File descriptor inheritance and stdio**

   12.1. **Initial sweep.** All descriptors except 0–2 are closed at the very beginning, before parsing. Inherited sockets, files, or pipes from a careless parent are therefore removed unless they occupied the first three slots—those are the supported channel for passing stdio.

   12.2. **Daemonization.** When daemonization is requested, the null device is opened **before** the pivot so the descriptor refers to the host’s device table; after the pivot the same numeric FD still maps to a valid sink. A double-fork pattern follows: first fork lets the parent exit; the child creates a new session; second fork ensures the final process is not a session leader (so it cannot reacquire a controlling terminal). Standard input, output, and error are then duplicated onto the null descriptor, replacing inherited stdio.

   12.3. **Non-daemon exec.** When not daemonizing, the final exec uses inherited standard streams—whatever was wired to 0–2 after the initial close sweep (typically the parent’s pipes or terminals). The target therefore sees the same stdio layout the orchestrator established, modulo the initial FD scrub.

   12.4. **Netns handle lifetime.** The network namespace join opens a file object whose descriptor is dropped when the function returns; no extra FD remains for the netns path.

13. **Privilege model and drop**

   13.1. The bootstrap runs with the invoking user’s privileges—typically effective root—through namespace setup, cgroup writes, `mknod`, ownership changes, and pivot. **Dropping to the configured UID and GID happens at exec time** via the standard POSIX credential fields on the exec call, not via `setuid`/`setgid` earlier. Until exec, the process remains privileged enough to create device nodes and chown jail content.

   13.2. **Implication.** Any code path that could execute untrusted logic before exec would run with full bootstrap privileges; the design keeps that surface minimal (no interpreters, no network listeners in the bootstrap itself).

14. **Daemonization and CPU accounting**

   14.1. CPU time spent in the bootstrap is metered in microseconds across forks so the child can attribute how much time the wrapper consumed versus the workload. Timestamps are passed as arguments to the target for logging startup metrics.

15. **Final exec**

   15.1. Whether or not a new PID namespace is used, the last step replaces the process with the copied binary. The bootstrap injects timing arguments—monotonic start time, process CPU start time, and accumulated bootstrap CPU time—so the monitor can log accurate startup metrics. The configured UID and GID are applied via the standard POSIX user/group fields for exec. Additional CLI tokens after `--` are appended for monitor-specific flags.

   15.2. Exec failure surfaces as an IO error in the bootstrap’s error taxonomy; successful exec does not return.

   15.3. **PID file semantics.** When not using a new PID namespace, the bootstrap writes the **current** process PID to the sibling pid file immediately before exec—the PID that will become the monitor after successful exec. When using a new PID namespace, the **child** PID is written and the parent exits without execing the monitor.

16. **Failure modes (operator-facing)**

   16.1. **Early abort.** Unsupported kernel for descriptor batch close; failure is fatal before any user-facing validation.

   16.2. **Configuration.** Invalid instance ID; non-numeric UID/GID; cgroup version string not 1 or 2; cgroup property format not `key=value`; empty value; cgroup file missing a dot (controller not parseable); parent cgroup path containing absolute or relative traversal; cgroup file tokens containing path components.

   16.3. **Host cgroup layout.** Missing hierarchy for selected version; controller not mounted (v1) or not listed at unified root (v2); inheritance cannot find a non-empty parent line; write failures to pseudo-files.

   16.4. **Resources.** Unknown resource limit name; non-integer limit value.

   16.5. **Filesystem and jail.** Canonicalization failures; target not a file; cannot create jail root; copy failure; mount/pivot/umount failures; cannot create device nodes or change ownership.

   16.6. **Network namespace.** Open failure; setns failure (wrong file type).

   16.7. **Daemonization and clone.** Fork or session errors; clone failure for new PID namespace.

   16.8. **Exec.** Last-step replacement failure (missing binary after copy, permission bits, etc.).

17. **Production invariants**

   17.1. **Cgroups before pivot.** Cgroup membership and properties are applied while the process can still reach the host’s cgroup filesystem layout; after the mount pivot, the same absolute paths are no longer guaranteed visible. This ordering is non-negotiable for correct resource accounting.

   17.2. **Network namespace before pivot.** Joining the desired netns must happen before the mount namespace changes which filesystem view is authoritative for later operations.

   17.3. **Rlimits before heavy syscall work.** Limits are installed early so subsequent steps run under the configured ceiling (notably open file count).

   17.4. **Daemonize null device opened pre-pivot.** Ensures the duplicated stdio descriptors reference a valid character device after the root filesystem switch.

   17.5. **Deterministic cgroup semantics.** Soft and hard rlimit pairs match; cgroup writes use newline-terminated lines; v1 and v2 both use two-phase write-then-attach; v2 rejects unknown controllers at the root allowlist.

   17.6. **PID publication** matches mode: parent writes child PID only in the new-PID-namespace path; otherwise the writing PID matches the post-exec monitor PID for the simple exec path.

18. **Error handling philosophy**

   18.1. Errors are typed and descriptive: cgroup misconfigurations, missing hierarchies, invalid parent paths, pivot failures, and permission problems each map to distinct variants. Many IO failures carry the offending path for operator debugging (without treating those strings as a stable API).

   18.2. Tests exercise cgroup parsing, inheritance edge cases, mock cgroup filesystems, descriptor closing on sufficiently new kernels, environment scrubbing, and resource limit installation—reflecting that correctness is validated at the unit level because integration tests for full namespace behavior are environment-sensitive.

19. **End-to-end flow (conceptual)**

```
  [privileged start]
        |
        v
  +------------------+
  | Close extra FDs  |
  | Clear all env    |
  +------------------+
        |
        v
  +------------------+
  | Parse & validate |
  | args, build Env  |
  +------------------+
        |
        v
  +------------------+
  | Copy binary into |
  | computed jail    |
  +------------------+
        |
        v
  +------------------+
  | Join netns if set|
  +------------------+
        |
        v
  +------------------+
  | setrlimit        |
  +------------------+
        |
        v
  +------------------+
  | Apply cgroups    |
  | (write, attach)  |
  +------------------+
        |
        v
  +------------------+
  | [aarch64] copy   |
  | sysfs snippets   |
  +------------------+
        |
        v
  +------------------+
  | [optional] open  |
  | null for daemon  |
  +------------------+
        |
        v
  +------------------+
  | pivot jail root  |
  +------------------+
        |
        v
  +------------------+
  | mkdir/chown      |
  | mknod devices    |
  +------------------+
        |
        v
  +------------------+
  | [optional]       |
  | double-fork      |
  | detach stdio     |
  +------------------+
        |
        v
  +------------------+
  | [optional] clone |
  | new PID ns +     |
  | write .pid OR    |
  | write .pid +     |
  | direct exec      |
  +------------------+
        |
        v
  [Monitor runs as unprivileged UID/GID inside jail]
```

20. **Optional tracing feature**

   20.1. The crate can be built with an optional tracing feature that pulls in shared instrumentation helpers from the workspace. When enabled, log spans can be emitted around critical sections; when disabled, the binary stays minimal. This does not change isolation semantics—only observability.

21. **Dependencies (conceptual)**

   21.1. Beyond the Rust standard library, the implementation relies on: safe wrappers around Linux syscalls; a small regular expression engine for parsing mount tables; lightweight error derivation; and shared utilities for CLI parsing, time queries, and instance-ID validation. The overall dependency footprint is intentionally small for a security-sensitive bootstrap binary.
