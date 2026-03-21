# Virtual machine configuration layer

## Purpose & Boundaries

This layer defines the typed shapes that correspond to externally supplied configuration for a micro virtual machine: how much compute and memory to expose, how the guest boots, which block and network devices exist, optional virtio services such as host–guest sockets, memory ballooning, entropy, telemetry output, metadata service routing, and parameters for saving or restoring machine state. Its responsibility is to capture those inputs in a form suitable for validation, merge with partial updates where the API allows patches, and hand off coherent descriptions to builders that materialize live devices and subsystems.

It is not responsible for the HTTP or command-line framing of requests, for the long-running execution loop of the virtual machine monitor, or for the low-level implementation of individual virtio devices. It also does not own guest memory layout beyond expressing constraints such as page-backing policy and compatibility with other options. If this layer mis-specifies or fails to reject invalid combinations, callers may see deserialization errors, explicit validation failures, or downstream errors when opening host resources; in the worst case, inconsistent configuration could contribute to a guest that fails to boot or to incorrect resource limits, though the surrounding monitor is expected to enforce additional checks.

## Interfaces & Contracts

The configuration surface is expressed as JSON-compatible documents with strictly defined keys: unknown keys are rejected where the schema uses a closed-object policy, which reduces the risk of silent typos and forces clients to adopt the supported vocabulary.

Cross-cutting concerns appear in several places. Rate limiting for block input/output, network receive and transmit, and the entropy source is described with the same conceptual model: optional bandwidth and operations buckets, each carrying a bucket size, an optional one-time burst allowance, and a refill interval. When both optional buckets are absent at the outer level, serializers may omit the rate limiter entirely; when building a live limiter from an empty description, defaults yield pass-through behavior. Token bucket fields that are left at zero-sized defaults still participate in construction rules used elsewhere in the monitor.

Machine-wide settings include virtual CPU count, memory size in mebibytes, whether simultaneous multithreading is requested, an optional static CPU template selection for feature filtering, whether dirty page tracking is enabled for incremental snapshots, optional huge-page backing for guest memory, and under certain build configurations an optional path for debugger attachment. Defaults favor a single virtual CPU, one hundred twenty-eight mebibytes of RAM, simultaneous multithreading off, no template, dirty tracking off, and conventional page size backing.

Boot configuration requires a host path to the kernel image; an optional initial ramdisk path; and optional kernel command-line text. When the command line is omitted, a long default string is substituted that favors fast boot and predictable shutdown behavior in minimal guests.

Block devices are identified by a stable string id, may declare a partition UUID for root-disk boot, may be marked as the single root volume, carry a cache policy defaulting to a mode that does not advertise flush semantics to the guest unless configured otherwise, and for virtio-backed disks include read-only mode, host file path, optional rate limits, and an optional I/O engine selector. Alternative backends may be selected using a host Unix socket path instead of a file path; the document must be consistent with exactly one backend style.

Network interfaces bind a guest-facing identifier to a host tap or similar device name, optionally fix a guest MAC address, and may attach receive and transmit rate limiters.

The virtual socket facility accepts an optional device identifier for ingestion (serialization may omit it when empty), a thirty-two-bit guest context identifier, and a host Unix domain socket path.

Balloon configuration carries a target size in mebibytes, a flag to allow automatic deflation when the guest is under memory pressure, and a statistics polling interval with a default of zero meaning statistics are not driven by a timer unless set.

Entropy configuration is minimal: an optional rate limiter wrapping the virtio random source.

Metrics configuration names a single host path used as the sink for line-oriented metrics; that path may be a regular file or a pipe, opened in a way that avoids blocking the process when consumers are slow.

Metadata service configuration selects a version, lists which guest network interface identifiers may forward metadata requests, and optionally sets a link-local IPv4 address for the service endpoint.

Snapshot creation names snapshot type (full versus differential), and paths for persisted virtual machine state and guest memory. Snapshot loading names state and memory sources, optional dirty-page tracking and automatic resume flags, optional network interface re-mapping from saved identifiers to new host device names, and either a flat memory file path or a structured memory backend describing a file or user-fault file descriptor handler path. A separate small document models pause versus resume for snapshot-related control flows.

Callers must supply internally consistent documents: for example, memory backends must not specify conflicting exclusive options, and machine memory must not be driven below an already configured balloon target. The layer provides explicit error categories for many of these cases when higher-level merge logic runs.

## Data Flow

Configuration data enters as serialized JSON (or equivalent) from the control plane. Deserialization produces strongly typed values. Closed schemas strip unknown keys at parse time; optional fields receive defaults prescribed in the schema or by type defaults.

The following diagram sketches how a single configuration document is refined before it affects running devices.

```
  External JSON body
        |
        v
  Deserialize + deny-unknown-keys
        |
        +--> Machine / boot / device payloads -----> Validation hooks
        |                                              (paths, counts,
        |                                               cross-field rules)
        v
  Builder or store (block list, net list, vsock slot, etc.)
        |
        v
  Live devices, rate limiters, memory policy, MMDS wiring
```

Machine and partial machine updates combine prior state with optional fields: unspecified patch fields retain previous values. Boot configuration is validated by attempting to open the kernel and optional initrd and by parsing the command line within a maximum length enforced by the loader layer. Block and network builders resolve identifiers, enforce uniqueness rules such as a single root disk and non-colliding guest MAC addresses, and construct or replace entries in ordered collections. The virtual socket builder replaces any previous socket endpoint and may remove an old host socket file before binding a new one. Balloon and entropy builders instantiate or replace their respective singleton slots. Metrics initialization opens the configured sink and registers a line writer with the global metrics facade.

Data exits when the monitor serializes current configuration back to clients: for example, machine configuration may omit custom CPU templates from the wire representation while preserving static template selections, and several optional fields are skipped when empty to keep responses small.

## Control Flow

Execution is request-driven. Each API action supplies a document that triggers deserialization, optional merging with existing resources, validation, and builder updates. The critical path for bringing up a guest typically sequences machine configuration, boot source, drives, network interfaces, and optional devices, then proceeds to execution not shown here.

Branching is extensive. Machine updates refuse simultaneous multithreading on architectures where it is unsupported, reject virtual CPU counts outside the supported range or inconsistent with simultaneous multithreading parity rules, and reject zero memory or memory sizes incompatible with the selected huge-page granularity. Memory size must align with page policy: conventional four-kilobyte backing accepts any positive integer mebibytes; two-megabyte hugepage backing requires the memory size in mebibytes to be divisible by two. Combining huge-page guest memory with ballooning is treated as incompatible when those checks run in context. Boot validation branches on missing versus provided command line and on presence of an initial ramdisk.

Block configuration branches between virtio file-backed disks and vhost-user socket backends; invalid combinations fail early. Inserting a new root disk when another root already exists and is not the same slot fails. Network configuration checks MAC uniqueness among interfaces before creation. Snapshot load paths branch between loading guest memory from a single file versus a described backend type, and honor deprecated boolean names for dirty tracking while preferring the newer field.

The following control sketch shows high-level decision points for machine configuration updates.

```
                    Request received
                           |
                           v
              +---------------------------+
              | Parse partial update      |
              +---------------------------+
                           |
              +------------+------------+
              |                         |
              v                         v
     SMT requested?              vCPU count in range
     on unsupported arch              and SMT parity OK?
              |                         |
              +------------+------------+
                           |
                           v
              +---------------------------+
              | Memory size > 0 and       |
              | compatible with page mode |
              +---------------------------+
                           |
                           v
              +---------------------------+
              | Emit merged machine view  |
              +---------------------------+
```

## State & Lifecycle

The layer itself is largely stateless in terms of persistent storage; state lives in companion builder structures that hold handles to devices and ordered lists. Block devices keep a deque ordering where the root device, if any, is always first to stabilize boot semantics when partition identifiers change. Network devices sit in a vector with replace-on-id semantics for updates. Virtual socket and balloon builders hold at most one active device each; inserting replaces the previous instance.

Initialization begins empty unless restoring from a snapshot, in which case devices may be rehydrated from saved state and wired through setter paths that bypass fresh construction from JSON.

Steady-state operation services updates: partial network and block updates only touch fields present in the patch; rate limiter updates interpret absent optional buckets as “no change” versus present buckets that may disable or reconfigure limits. Balloon size updates are separate from statistics interval updates; toggling whether statistics are enabled is not permitted after boot, only interval changes when statistics were already enabled at configuration time.

Shutdown and teardown are owned by the broader monitor; this layer’s builders release resources such as Unix sockets when superseded.

For snapshot-related pause and resume, a small enumerated state distinguishes paused snapshots from resumed operation; this flows through serialization for control APIs rather than through the main device configuration structs.

## Failure Modes

Failures fall into several classes. Parse-time errors arise from unknown JSON keys, malformed types, or incompatible combinations that serde can express. Machine configuration can fail when balloon memory targets exceed proposed RAM, when balloon state cannot be read for validation, when kernel version probing for huge-page compatibility fails, or when simultaneous multithreading is invalid for the architecture.

Boot failures surface as inability to open kernel or initrd paths, or invalid kernel command lines after length and parsing checks.

Block operations may fail when constructing devices from bad paths, when adding a second root disk, or when updates use backends that do not support the requested operation. Network failures include duplicate guest MACs for different interface identifiers, tap open errors when host names collide, or device implementation errors. Virtual socket failures include backend creation or bind errors on the Unix path.

Balloon errors include missing device, inactive device, invalid statistics changes after boot, excessive page requests, missing statistics when disabled, creation and update failures, and incompatibility with huge-page memory policy.

Entropy failures stem from device creation or rate limiter construction. Metrics initialization fails when the output path cannot be opened or the metrics system rejects repeated initialization.

Metadata service configuration fails when the allowed interface list is empty, when the IPv4 address is not link-local, when listed interface identifiers do not exist, or when the backing data store cannot be initialized.

Snapshot creation and load failures are largely handled outside this module’s types but invalid memory backend combinations and missing required paths are caught at deserialization or merge time. Using deprecated snapshot flags alongside newer ones can confuse operators; the schema keeps older names but steers toward the current dirty-page tracking field.

Silent corruption is guarded by closed schemas and explicit validation rather than coercion; the most subtle risk is misunderstanding default behaviors such as omitted rate limiters, omitted balloon statistics intervals, or boot command-line substitution, which can look like “the system ignored my setting” when the client omitted a field.

## Operational Characteristics

Configuration processing is lightweight compared to emulation: it allocates small structs, walks short lists, and opens files or sockets as needed. The main scaling limits are the maximum virtual CPU count, the number of block and network devices supported by the surrounding implementation, and host resource availability for tap devices and disk files.

Rate limiting configuration allows operators to cap bandwidth and operation rates for storage, network, and entropy independently. Metrics output uses nonblocking opens so a slow consumer cannot deadlock the monitor; writes may eventually fail if a pipe fills beyond the platform buffer, which is an intentional back-pressure signal.

Observability of this layer is indirect: errors are surfaced to callers with categorized messages; successful paths update builders whose devices expose their own logging and metrics. There is no distributed tracing inside these structs themselves.

## Design Rationale

The design favors explicit, versionable JSON documents with deny-unknown-fields semantics so automated clients and humans get predictable failures. Shared rate limiter shapes avoid duplicating subtly different parameter names across subsystems. Strong defaults reduce boilerplate for minimal guests while still allowing advanced tuning.

Separating full machine configuration from partial updates makes it possible to validate merged state once rather than scattering defaults. Boot command-line defaulting centralizes a vetted minimal policy instead of relying on guest kernel defaults that vary by version.

Block device ordering with a dedicated root slot prevents ambiguous boot ordering when users switch between partition-based and device-based root. Network MAC collision checks fail early rather than producing opaque guest misbehavior.

Balloon and huge-page incompatibility reflects kernel and memory management constraints: ballooning relies on mechanisms that conflict with fixed huge-page mappings in this stack. Metadata service configuration requires explicit interface allowlists so exposure is opt-in per NIC.

Snapshot parameters distinguish full and differential types and multiple memory restore strategies so operators can optimize for space, migration time, or integration with custom page fault handlers. Deprecated fields remain parseable to avoid breaking older automation while nudging users toward current terminology.

---

## Diagram: Configuration domains and dependencies

The following picture shows conceptual groupings and dependencies between major configuration areas. Arrows read as “depends on or must be consistent with.”

```
  +------------------+
  | Machine (CPU,    |
  | RAM, pages,      |
  | dirty tracking)  |
  +--------+---------+
           |
           |  RAM vs balloon / huge pages
           v
  +------------------+       +------------------+
  | Balloon          |       | Boot source      |
  | (target memory)  |       | (kernel, cmdline)|
  +------------------+       +--------+---------+
                                      |
  +------------------+                |
  | Drives           |<---------------+ (root disk / boot)
  +------------------+

  +------------------+       +------------------+
  | Network          |------>| MMDS             |
  | interfaces       |       | (allowed NICs)   |
  +------------------+       +------------------+

  +------------------+       +------------------+
  | Metrics          |       | Snapshots        |
  | (output path)    |       | (save/load paths) |
  +------------------+       +--------+---------+
                                      |
                                      v
                            +------------------+
                            | Machine state    |
                            | (pause / resume) |
                            +------------------+

  +------------------+       +------------------+
  | Vsock            |       | Entropy          |
  | (CID, UDS path)  |       | (optional RL)    |
  +------------------+       +------------------+
```

Machine configuration sits at the top because it constrains memory semantics that interact with ballooning and page backing. Boot and drives intersect on root filesystem placement. Network configuration feeds metadata service policy. Snapshots depend on machine-level dirty tracking choices and reattach network and memory resources on load. Virtual socket and entropy configurations are largely independent aside from general resource contention on the host.

## Diagram: Snapshot load decision outline

Loading persisted state requires choosing how guest memory is supplied. The next figure outlines the decision between a legacy single file path and a structured backend description.

```
              Snapshot load request
                       |
                       v
         +-----------------------------+
         | Memory file path only?      |
         +-----------------------------+
                 |              |
                yes             no
                 |              |
                 v              v
    +--------------------+   +------------------------+
    | Load RAM from file |   | Require backend block  |
    +--------------------+   | (file or user-fault      |
                             | handler socket)        |
                             +------------+-----------+
                                          |
                                          v
                             +------------------------+
                             | Optional resume VM,    |
                             | dirty tracking, NIC    |
                             | overrides              |
                             +------------------------+
```

This separation exists so simple deployments can keep a single memory image path while advanced integrations can supply a user-fault handler for lazy memory fill. Network overrides allow restored guests to attach to differently named host interfaces without editing stored state files by hand.

## Closing note on defaults and validation summary

Across the surface, defaults are chosen so that a minimal JSON document still yields a runnable microVM: modest memory and one virtual CPU, conventional pages, no dirty tracking unless requested, standard boot arguments when the guest kernel command line is omitted, block cache behavior aligned with performance-first virtio defaults, no rate limits unless specified, balloon statistics polling off unless an interval is set, metadata service version defaulting to the stack’s primary version when unspecified, and full snapshots when the snapshot type is omitted. Validation is layered: syntactic closure of JSON objects, field-level constraints, cross-field rules within a struct, and cross-resource checks that require consulting builders or host state. Together, these properties define how external configuration is normalized into a consistent operational picture for the rest of the virtual machine monitor.
