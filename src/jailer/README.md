# Jailer

1. **Role and threat model**

   1.1. The jailer is a small, privileged bootstrap program whose job is to start the Firecracker virtual machine monitor under production-style constraints: it establishes Linux namespaces, cgroups, filesystem isolation, and resource limits, then drops from root-equivalent privileges to configured non-root identities before replacing itself with the target binary. The design assumes an untrusted or at least tightly bounded workload inside the jail and a host that may run many instances; copying the executable into the jail instead of hard-linking avoids shared executable pages between processes, which matters for certain threat models around memory disclosure.

   1.2. Operation splits cleanly into three conceptual layers: **process hygiene** (strip inherited state that could leak across invocations), **configuration assembly** (validate identifiers, paths, cgroup and rlimit settings, and compute the jail layout), and **ordered system calls** (netns, rlimits, cgroups, pivot-based jail, device nodes, optional daemonization and PID namespace, then exec). Order is not arbitrary—cgroups must be applied while the process can still see the host cgroup filesystem; chroot/pivot happens after that; device creation happens inside the new root.

2. **Startup and process hygiene**

   2.1. Before any argument parsing, the process aggressively reduces its inherited surface area. All open file descriptors except the standard three are closed using the kernel `close_range` facility when available, with the `CLOSE_RANGE_UNSHARE` flag so closure semantics match the intent to shed unrelated descriptors. This runs before clearing the environment so that a minimal, predictable descriptor table exists regardless of how the parent launched the jailer.

   2.2. Every environment variable inherited from the parent is removed in a single-threaded loop. The rationale is to prevent accidental leakage of secrets, locale overrides, or library-controlled behavior into the jailed child. After this step, the only meaningful process state left is the program image, argv, and the three standard streams (which may themselves be redirected later).

   2.3. Argument parsing is strict: required fields include a stable instance identifier (validated against the project’s instance-ID rules), paths to the binary to exec, numeric user and group IDs to assume after isolation is prepared, and a configurable base directory for jail roots. Optional flags control network namespace attachment, cgroup version and properties, parent cgroup placement, per-process resource limits, daemonization, and whether to exec inside a new PID namespace. Text after a `--` separator is forwarded verbatim to the target binary.

   2.4. Help and version requests short-circuit before any filesystem mutation: they print usage or version and exit successfully.

3. **Jail path layout and executable handling**

   3.1. The chroot target path is derived by canonicalizing the jail base, appending the **basename** of the resolved executable path (not the full path), then the instance ID, then a fixed leaf segment naming the actual root seen inside the jail. That scheme colocates all instances that share the same Firecracker binary name under one subtree while still isolating each instance by ID. The outer process ensures this directory exists before proceeding.

   3.2. The executable file is **copied** into the jail root under its original basename. The copy is intentional: hard links would fail across devices, and more importantly, sharing text segments between Firecracker processes is considered undesirable for isolation. The destination path inside the jail becomes the argv[0] target for the final exec.

4. **Resource limits (rlimits)**

   4.1. Configurable limits map to classical Unix `setrlimit` resources. File-size limits are optional; if omitted, no change is made to the default maximum file size beyond what the process already inherited. The open-file-descriptor limit always has an explicit value: a built-in default applies when the user does not override it, so every run establishes a known ceiling on descriptor count.

   4.2. Installation sets **both** current (soft) and maximum (hard) limits to the same target value. That avoids a gap where a process could raise its soft limit back toward a higher hard limit; the jailer’s goal is a single, tight bound, not gradual enforcement. Parsing accepts only non-negative integer magnitudes; malformed tokens fail fast during configuration assembly rather than at syscall time.

5. **Cgroup integration**

   5.1. **Discovery.** At initialization time the implementation scans the kernel’s view of mounted filesystems to find cgroup v1 and v2 mount points. A line-oriented regular expression distinguishes `cgroup` versus `cgroup2` entries and extracts mount directory and mount options. For v1, multiple mount points may exist (one per controller hierarchy or combined hierarchies); each is cached. For v2, a single unified hierarchy is expected; discovery records its root. If the requested cgroup version’s mounts are absent, configuration fails early with a hierarchy error rather than at first write.

   5.2. **Controller resolution (v1).** Properties are expressed as `controller.property=value`. The controller name is the substring before the first dot; this supports dotted property names such as memory controller knobs with multiple segments. Each controller maps to a hierarchy root discovered by scanning mount options for the controller keyword. Unknown or unmounted controllers produce a clear “controller unavailable” error.

   5.3. **Unified hierarchy (v2).** All properties are written under the unified mount. Before writing leaf properties, the implementation may need to enable controllers along the path using `cgroup.subtree_control` writes of `+controller` form. Enabling walks upward so parent nodes delegate control to children; duplicate enables are skipped. Available controllers are detected once from the root’s `cgroup.controllers` list; attempts to set properties for controllers not exposed there fail rather than silently no-op.

   5.4. **Parent cgroup path.** The parent cgroup is a **relative** path segment (no leading slash, no `.` or `..` components). It defaults to the executable basename when not specified, which yields a per-binary grouping under the host’s cgroup tree. The leaf cgroup directory is `parent/id` under the resolved hierarchy root(s).

   5.5. **Inheritance for empty parent files.** Cgroup pseudo-files sometimes appear in new directories empty even when the parent’s file holds a value the child must mirror before writes are accepted (cpuset is the canonical example). A recursive helper reads the parent’s first line; if empty, it walks up the parent chain up to a bounded depth, then copies the first non-empty line into the child. The algorithm tolerates concurrent jailers racing to populate the same parent file: a failed write on a race may still leave the parent eventually non-empty, and a second read picks up the value. Writes to cgroup files use newline-terminated payloads because many controllers expect a single line.

   5.6. **Two-phase application.** For each cgroup object, values are written first (creating directories as needed), then the process attaches itself. The split exists because some controllers require configuration ordering (e.g., cpuset memory and CPUs before membership). Attachment uses v1’s `tasks` file or v2’s `cgroup.procs` as appropriate.

   5.7. **v2-only “move to parent” behavior.** When no cgroup properties are supplied but cgroup v2 is selected and a parent cgroup path is given, if that parent directory already exists on disk, the current process writes its PID into the parent’s `cgroup.procs` to move itself into that group without creating a new leaf. This supports deployments that pre-create a cgroup and only want the jailer to join it.

6. **Network namespace**

   6.1. If a netns path is provided, the process opens that path as a file and issues `setns` with the network namespace clone flag. This must occur before the mount namespace pivot so that the correct network configuration is visible to subsequent steps and to the child. Failure modes include missing files or non-namespace files, which surface as open or `setns` errors.

7. **Filesystem jail (mount namespace + pivot)**

   7.1. The jail is not a bare `chroot`. The sequence is: create a new **mount namespace**, mark mount propagation as **slave** recursively from the root (so subsequent mounts do not propagate unexpectedly), **bind-mount** the intended jail directory onto itself (working around pivot_root’s requirement that new and old roots differ in mount topology), change the working directory into that directory, create a placeholder directory for the old root, then **pivot_root** so the jail becomes `/` and the previous root is hidden under a temporary mount point. After pivot, the working directory is explicitly set to `/` because pivot does not guarantee cwd semantics. The old root is then lazy-unmounted and the empty directory removed.

   7.2. That sequence yields stronger isolation than chroot alone: the process loses visibility of the original host root tree except through the carefully chosen bind mount, and mount namespace boundaries reduce cross-talk with host mount events.

8. **Post-pivot content inside the jail**

   8.1. A fixed set of directories is created with restrictive permissions (typically owner-only) and ownership changed to the configured UID/GID. This includes the root, a device hierarchy, and a run directory commonly used for API sockets. Ownership walks use explicit `chown` calls per path because recursive ownership is not assumed from a single syscall.

   8.2. **Device nodes** are created with `mknod` for character devices with known static major/minor pairs: KVM, TUN/TAP, and `/dev/urandom`. Ownership is immediately set to the jailed user. If urandom creation fails, a warning is printed because some guest features (specifically MMDS v2) depend on entropy; the jailer continues rather than aborting, reflecting a degrade-gracefully policy for optional capabilities.

   8.3. **Userfaultfd.** The misc device minor number is not fixed; it is discovered at runtime by scanning the kernel’s misc device table for the userfaultfd entry. If present, a matching device node is created inside the jail with the dynamic minor; if absent, that feature is simply unavailable without failing the whole run.

   8.4. **Architecture-specific host facts (AArch64).** On ARM64 hosts, selected sysfs files describing CPU cache topology and the `MIDR_EL1` identification register are copied as small text files into parallel paths under the jail root so Firecracker can construct accurate CPU models without exposing all of sysfs. Each copied file is owned by the jailed UID/GID. This step is gated by architecture cfg so x86_64 builds do not include it.

9. **Daemonization**

   9.1. When daemonization is requested, `/dev/null` is opened **before** pivoting so the descriptor refers to the host’s device table; after pivot the same numeric FD still maps to a valid null sink. The implementation follows a **double-fork** pattern: first fork lets the parent exit; the child calls `setsid` to detach from any controlling terminal and become a session leader; a second fork ensures the final process is not a session leader (so it cannot reacquire a controlling terminal). Standard input, output, and error are then dup’d to the null descriptor.

   9.2. CPU time spent in the jailer is metered in microseconds across these forks so the child process can attribute how much time the wrapper consumed versus the workload.

10. **PID namespaces and session handling**

    10.1. Optional execution in a new PID namespace uses a raw `clone` syscall with the new-PID flag and a null stack pointer—legal because the child does not share memory with the parent like a threaded clone would. The child becomes PID 1 in the new namespace and execs the target. The parent writes the child’s PID to a sibling file named after the jailed executable path with a `.pid` suffix, then exits successfully; orchestrators read that file to discover the Firecracker PID on the host.

    10.2. A subtle interaction with POSIX sessions is handled: if the jailer would otherwise be a session leader (same SID as PID), exiting could deliver `SIGHUP` to the new namespace’s init after it installs handlers. When not daemonized, the code detects session leadership and moves the child into a new session so Firecracker is not spuriously signaled when the jailer exits. Daemonization already avoids session leadership via double fork, so that branch is skipped there.

11. **Final exec**

    11.1. Whether or not a new PID namespace is used, the last step replaces the process with the copied binary. The jailer injects timing arguments—monotonic start time, process CPU start time, and accumulated jailer CPU time—so Firecracker can log accurate startup metrics. The configured UID and GID are applied via the standard POSIX user/group fields for exec. Additional CLI tokens after `--` are appended for Firecracker-specific flags.

    11.2. Exec failure surfaces as an IO error bubbled through the jailer’s error taxonomy; successful exec does not return.

12. **Error handling philosophy**

    12.1. Errors are typed and descriptive: cgroup misconfigurations, missing hierarchies, invalid parent paths, pivot failures, and permission problems each map to distinct variants. Many IO failures carry the offending path for operator debugging (while this document avoids enumerating those strings as API).

    12.2. Tests throughout the crate exercise cgroup parsing, inheritance edge cases, mock cgroup filesystems, close_range behavior on sufficiently new kernels, environment scrubbing, and resource limit installation—reflecting that correctness is validated at the unit level because integration tests for full namespace behavior are environment-sensitive.

13. **End-to-end flow (conceptual)**

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
  | direct exec      |
  +------------------+
        |
        v
  [Firecracker runs as unprivileged UID/GID inside jail]
```

14. **Optional tracing feature**

    14.1. The crate can be built with an optional tracing feature that pulls in shared instrumentation helpers from the workspace. When enabled, log spans can be emitted around critical sections; when disabled, the binary stays minimal. This does not change the isolation semantics—only observability.

15. **Dependencies (conceptual)**

    15.1. Beyond the Rust standard library, the implementation relies on: safe wrappers around Linux syscalls; a small regular expression engine for parsing mount tables; lightweight error derivation; and shared utilities for CLI parsing, time queries, and instance-ID validation. The overall dependency footprint is intentionally small for a security-sensitive bootstrap binary.
