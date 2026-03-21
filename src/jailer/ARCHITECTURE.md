# Jailer Architecture

## Purpose & Boundaries

The jailer is a small, privileged helper whose job is to prepare a tightly constrained execution environment on Linux and then replace itself with a main workload binary—typically a virtual machine monitor—running with reduced privileges and a minimal view of the host filesystem and kernel interfaces. Its responsibility is everything that must happen *before* the main binary runs: validating inputs, copying the executable into an instance-specific root, joining optional kernel namespaces, applying cgroup membership and resource limits, performing a hardened filesystem pivot so the process cannot escape upward, creating a minimal device layer inside the new root, optionally daemonizing, and finally executing the target with a defined identity and forwarded arguments.

What the jailer is *not* responsible for is equally important. It does not implement the virtual machine itself, device emulation, guest memory management, or API servers. It does not define or install seccomp-bpf syscall filters; those policies, when present, belong to the workload or to other layers that load after this helper has finished its setup. The helper also does not orchestrate networking beyond optionally entering an existing network namespace path supplied by the operator. It does not manage persistent disk images, cloud metadata services, or lifecycle of the guest beyond handing off execution.

If this component fails, the larger system loses the ability to start the main binary in a standardized, locked-down posture. Operators would either run the workload without the same isolation guarantees or would block launches entirely. Misconfiguration surfaces as early errors during setup rather than silent weakening of isolation, which is intentional: failing closed on ambiguous cgroup layout or invalid paths is preferable to proceeding with a partially applied sandbox.

---

## Interfaces & Contracts

The jailer exposes a command-line interface understood by an argument parser shared with related tooling. Callers must supply a stable instance identifier, a path to the executable that will run inside the jail, and numeric user and group identifiers to which the process will drop after filesystem setup. Optional inputs include a base directory under which per-instance chroot trees are rooted, a file path referring to a network namespace to join, flags to daemonize via session and standard I/O handling, a flag to spawn the workload in a new process-ID namespace, repeated cgroup property assignments, a cgroup version selector, an optional parent cgroup segment, repeated process resource limit pairs, and free-form arguments passed through to the target after a separator token.

The helper consumes the host operating system’s cgroup virtual filesystem layout as discovered through the kernel’s mount table, the ability to create directories and special files inside the chosen root, and standard POSIX process and session APIs. A critical invariant is that cgroup configuration that must be written from user space must be completed before the filesystem root is switched, because after that point the view of cgroup mount points may no longer be reachable at the original host paths.

The guarantees the helper provides are procedural rather than cryptographic: it will construct a dedicated directory hierarchy for the instance, copy the executable into that tree (avoiding hard links so distinct instances do not share executable pages in memory), apply namespace and limit settings requested on the same process that will eventually exec, pivot the filesystem root using mount namespace isolation plus pivot-root semantics, create only the device nodes required for typical virtualization and optional features, drop to the configured unprivileged identity for the exec, and pass timing metadata forward so the workload can account for setup overhead.

---

## Data Flow

Configuration enters as strings and flags from the process environment and command line. Before any higher-level logic runs, inherited open file descriptors beyond the standard three are closed and the environment variable table is cleared, so no accidental leakage of secrets or library state crosses into the jailed phase. The instance identifier and filesystem paths are validated and normalized; the executable path is canonicalized and must refer to a regular file. From these inputs the helper derives an absolute path to the per-instance root directory under the configured base, combining the base, the executable’s basename, the instance identifier, and a final segment that denotes the root of the jail.

The executable’s bytes are copied into that tree. Separately, optional cgroup directives are parsed into controller-specific property lists. The mount table file is read to locate cgroup v1 hierarchies or the cgroup v2 unified mount, depending on the selected version. Resource limit directives are parsed into numeric caps for file size and open file descriptor count, with defaults applied where not overridden.

At runtime inside the host mount namespace, cgroup directories may be created and special cgroup files written, sometimes after consulting parent cgroup files to inherit mandatory values when nested directories start empty—a pattern needed because some controllers require parent-defined state before child values make sense. Process identifiers may be written into cgroup membership files in two passes: first writing all properties, then attaching the current process, so ordering constraints between controllers are respected.

After the root filesystem change, the helper materializes a small hierarchy: root, a device directory, a nested path for tunnel devices, and a run directory for sockets. Device nodes are created with fixed major and minor numbers for virtualization and networking interfaces, with ownership aligned to the target user and group. On one architecture, selected sysfs-derived CPU topology files are copied into the jail so the workload can introspect cache geometry without exposing the entire sysfs tree. A pseudo-device for user fault handling may be exposed if the host publishes a discoverable minor number through a kernel proc file.

Finally, the main binary receives new arguments injected by the helper (instance id and timing information) plus any operator-supplied trailing arguments, and inherits standard streams unless daemonization replaced them with the null device.

The following diagram summarizes how inputs become durable filesystem and kernel state before control transfers to the workload.

```
  [Operator CLI + env (cleared)]
            |
            v
  +---------------------+
  | Parse & validate    |
  | identifiers, paths  |
  +---------------------+
            |
            v
  +---------------------+       +------------------+
  | Copy executable     |       | Build cgroup     |
  | into instance root  |       | writes & limits  |
  +---------------------+       +------------------+
            \                       /
             v                     v
           +-----------------------------+
           | Join netns (optional)        |
           | Apply rlimits, cgroups       |
           +-----------------------------+
                         |
                         v
           +-----------------------------+
           | Pivot root + mount namespace |
           +-----------------------------+
                         |
                         v
           +-----------------------------+
           | Create dev nodes, dirs       |
           | (optional daemonize)        |
           +-----------------------------+
                         |
                         v
           +-----------------------------+
           | Exec main binary as uid/gid  |
           +-----------------------------+
```

---

## Control Flow

Execution begins at process start, immediately performing sanitization: closing inherited descriptors and stripping environment variables. Parsing then determines whether the process should print help or version information and exit, or continue into setup.

The critical path for a normal launch constructs an environment object capturing all decisions, ensures the instance root directory exists on disk, then enters the run sequence. The run sequence copies the executable, optionally joins a network namespace, installs resource limits, applies cgroup configuration while still able to see host cgroup mounts, optionally opens the null device in preparation for daemonization, optionally copies architecture-specific sysfs snippets, then performs the filesystem pivot.

Inside the jail, directories are created with restrictive permissions and ownership handed to the configured user and group. Device nodes are created and owned accordingly. CPU time spent in the helper is accumulated so it can be reported to the workload. If daemonization was requested, a double-fork sequence detaches from controlling terminals and replaces standard input, output, and error with duplicates of the null device.

Branching occurs around whether the workload should run inside a new process-ID namespace. If so, the helper duplicates the process with a flag that creates a child namespace; the child becomes the first process in that namespace and may adjust session leadership to avoid signal delivery races when the helper exits. The helper writes the child’s process identifier to a sibling file next to the executable basename so external orchestration can discover the true PID when it differs from the helper’s. If no new process-ID namespace is requested, the helper records its own identifier and attempts the final exec directly, surfacing failure through the process image replacement error path.

Throughout, errors in system calls and filesystem operations map to typed failures with descriptive messages; there is no attempt to recover partially constructed cgroup state beyond what the operating system implies when the process exits.

The control-flow diagram below highlights decision points and the main spine.

```
                    START
                      |
                      v
               +--------------+
               | Sanitize FDs |
               | Clear env     |
               +--------------+
                      |
                      v
               +--------------+
               | Parse args   |----> help/version --> terminate OK
               +--------------+
                      |
                      v
               +--------------+
               | Build config |
               | Create root    |
               | directory      |
               +--------------+
                      |
                      v
               +--------------+
               | Copy binary  |
               +--------------+
                      |
                      v
            +-------------------+
            | Netns? join        |
            +-------------------+
                      |
                      v
            +-------------------+
            | Rlimits + cgroups  |
            +-------------------+
                      |
                      v
            +-------------------+
            | Pivot filesystem   |
            +-------------------+
                      |
                      v
            +-------------------+
            | Devices + perms    |
            +-------------------+
                      |
                      v
            +-------------------+
            | Daemonize?         |--yes--> double fork, dup null
            +-------------------+          |
                      |no                  |
                      +<-------------------+
                      |
                      v
            +-------------------+
            | New PID ns?       |
            +-------------------+
                 /        \
               yes        no
                |          |
                v          v
         clone+exec    direct exec
                \          /
                 v        v
               REPLACE PROCESS
```

---

## State & Lifecycle

Persistent state on disk consists of the per-instance root directory tree, including the copied executable, optional pid sidecar file, and any files created under the run directory by the workload after exec. Transient state in the helper includes parsed configuration, cgroup builder caches of mount locations, open file handles such as the null device during daemonization, and measured CPU time attributable to setup.

Initialization begins with an empty slate for environment variables and a minimal file descriptor table. Configuration state is assembled once parsing succeeds. The cgroup subsystem state transitions from discovery (reading mount information) to mutation (creating directories and writing pseudo-files) to attachment (writing the process id) before root isolation makes host paths potentially inaccessible.

The pivot into a dedicated mount namespace and new root is a one-way transition for the helper’s worldview: afterward, path operations are relative to the jail unless explicitly engineered otherwise. Device creation and ownership changes assume the process still has sufficient capability during that window.

Steady state for the helper is vanishingly short: it ends at successful exec, which does not return. If exec fails, the error is the last observable behavior. If daemonization forks were used, intermediate parent processes exit cleanly to avoid zombie relationships, leaving the grandchild to continue.

Recovery is not a first-class feature: a failed launch leaves whatever directories and partial files were created and relies on operators or higher-level orchestrators to reap or retry. Cgroup inheritance logic attempts to tolerate races when parallel launches populate parent cgroup files concurrently.

The state machine below abstracts the helper lifetime relative to isolation milestones.

```
  [Born with inherited FDs/env]
            |
            | sanitize
            v
  [Clean slate: stdio only]
            |
            | configure
            v
  [Host-visible: cgroups addressable]
            |
            | pivot root
            v
  [Jailed view: limited FS]
            |
            | prepare dev, drop creds for exec
            v
  [Ready to exec]
            |
            | exec
            v
  [Workload process]  (helper image gone)
```

---

## Failure Modes

Failures fall into several categories: input validation errors, host configuration mismatches, resource exhaustion, permission denials, and syscall errors during namespace or mount operations.

Validation errors include malformed instance identifiers, non-file executable paths, cgroup property strings that do not split cleanly into name and value, cgroup file names that do not resemble controller-qualified properties, resource limit keys that are unknown, or parent cgroup paths that attempt to escape upward using relative components. These fail fast with explicit errors and do not partially apply cgroup writes.

Host mismatches include selecting a cgroup version whose unified hierarchy is not mounted, referencing controllers not present in cgroup v2’s available set, or requesting cgroup v1 controllers absent from mount options. The helper surfaces these as missing hierarchy or unavailable controller failures rather than writing to wrong locations.

Filesystem operations can fail when directories cannot be created, copies fail across devices or due to space, pivot operations fail because mount propagation or bind prerequisites were not met, or device node creation conflicts with existing nodes. Daemonization can fail if session creation cannot proceed due to process group leadership constraints mitigated by forking.

A subtle class of failures involves cgroup inheritance: when parent special files are empty up the hierarchy, the helper may refuse to proceed because child values cannot become valid. Parallelism can cause transient empty reads; the design tolerates some races by re-reading parent values after recursive attempts.

Silent corruption is unlikely for configuration parsing because values are validated before application. The highest-risk silent assumption is operator correctness: an overly permissive user identifier combined with a loose jail root could allow a compromised workload to read host-mounted content if the operator bound sensitive mounts into the base directory—this is outside the helper’s ability to infer intent, so contract violations by callers are not automatically detected.

Regarding syscall filtering, absence of seccomp setup in this helper means a vulnerability in the main binary before it installs its own filters would not be mitigated by this layer. That boundary is intentional: syscall policy is specialized to the workload and evolves independently of filesystem and namespace setup.

---

## Operational Characteristics

The helper is CPU-bound during parsing and moderate I/O during copying the executable and writing cgroup pseudo-files. The copy trades disk space and duplicate mapping for stronger isolation between instances. Peak file descriptor usage is small and bounded; inherited descriptors are aggressively closed early to reduce accidental descriptor leaks to the workload.

Scaling concerns are mostly operational: each launch reads cgroup mount metadata and may walk multiple cgroup hierarchies. Rapid concurrent launches may contend on cgroup filesystem updates; the inheritance helper mitigates some races but cannot remove all kernel-side serialization. Disk usage grows linearly with retained per-instance roots and copied binaries.

Observability is primarily stderr-oriented error reporting on failure paths and optional warning lines when optional devices cannot be created. There is no embedded metrics server; timing information is forwarded to the workload for accounting rather than exported centrally. Tracing can be compiled in via optional features, but baseline builds rely on explicit error strings and process exit codes.

Privilege requirements include the ability to change mount namespaces, pivot root, create device nodes, and adjust ownership—capabilities typically associated with superuser or carefully delegated rights. The design assumes the helper runs with sufficient privilege for these operations and then drops to an unprivileged identity only for the final exec.

---

## Design Rationale

The overall problem is to make production-grade isolation reproducible without embedding all policy in shell scripts. Centralizing the sequence in a single helper reduces divergence between deployments and encodes ordering constraints—especially that cgroup configuration must precede filesystem isolation—that are easy to get wrong when expressed imperatively.

The hardened root switch uses a dedicated mount namespace and pivot-root rather than a bare chroot because chroot alone is weaker against certain escape techniques when the process retains undue capability or mount visibility. Bind-mounting the jail root onto itself satisfies pivot-root’s same-filesystem expectation, and recursive mount propagation adjustments align with pivot-root prerequisites on modern kernels.

Choosing to copy the executable rather than hard link reflects a threat model where shared text segments between instances would be undesirable. The cost is extra disk and cache footprint per instance, traded for stronger separation.

Optional network namespace joining lets platform networking own interface creation while the workload simply lands in the right view. Optional process-ID namespaces give the workload a clean process tree at the cost of more complex parent-child relationships and pid bookkeeping via sidecar files.

Cgroup v1 and v2 support acknowledges long migration timelines across Linux distributions: discovery via mount metadata avoids hard-coded paths while still allowing explicit version selection to prevent ambiguous behavior on hosts exposing both layouts.

Resource limits via per-process rlimit settings complement cgroup controllers: they cap file descriptors and file sizes cheaply without requiring controller availability for those facets.

Daemonization through double forking follows established Unix practice to avoid acquiring controlling terminals and to cooperate with session semantics when combined with process-ID namespace creation—where session leadership nuances could otherwise deliver stray signals to the workload during helper exit.

Finally, syscall filtering is left to the workload because effective seccomp profiles depend intimately on the syscalls the monitor uses, which change across versions and features. Mixing those policies into the helper would couple release cycles incorrectly and duplicate logic. The jailer focuses on orthogonal mechanisms—namespaces, filesystem containment, cgroup placement, privilege dropping, and minimal device exposure—so that syscall policy can evolve where the experts on the workload’s behavior reside.

---

### Diagram: Isolation Topology (conceptual)

Before presenting this diagram, recall that “topology” here means how isolation layers wrap the eventual workload, not network topology. After the diagram, the narrative takeaway is that multiple independent mechanisms stack: namespaces reduce what the kernel shows, cgroups classify and limit resources, rlimits bound per-process artifacts, filesystem pivoting constrains path resolution, and dropped credentials reduce impact if the workload is compromised.

```
  +-------------------------------------------------------------+
  | Host (initial helper privileges)                             |
  |  +------------------------+    +--------------------------+   |
  |  | Optional net namespace |    | cgroups + rlimits       |   |
  |  | (join existing path)   |    | (host paths, pre-pivot)   |   |
  |  +------------------------+    +--------------------------+   |
  |              \                      /                         |
  |               v                    v                          |
  |         +----------------------------------+                  |
  |         | New mount namespace + pivot root|                  |
  |         +----------------------------------+                  |
  |                       |                                       |
  |                       v                                       |
  |         +----------------------------------+                  |
  |         | Jailed FS view + curated /dev   |                  |
  |         +----------------------------------+                  |
  |                       |                                       |
  |                       v                                       |
  |         +----------------------------------+                  |
  |         | Optional new PID namespace        |                  |
  |         +----------------------------------+                  |
  |                       |                                       |
  |                       v                                       |
  |         +----------------------------------+                  |
  |         | Exec main binary as unprivileged  |                  |
  |         | (syscall policy not set here)      |                  |
  |         +----------------------------------+                  |
  +-------------------------------------------------------------+
```

This stack clarifies why ordering matters: cgroup writes belong to the host view, while the device layer belongs to the jail view, and privilege dropping happens at the last moment before the workload’s code executes.
