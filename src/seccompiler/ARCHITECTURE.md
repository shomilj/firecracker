# Architecture: Security Policy Compilation and Snapshot Tooling Trio

This document describes three related command-line utilities that ship with the microvirtualization stack. Together they form an operational tooling layer around two distinct concerns: translating human-editable security policies into kernel-enforceable programs, and manipulating persisted virtual machine snapshots outside the long-running daemon. The first utility compiles JSON-described seccomp policies into a compact binary artifact consumed by the main runtime. The second utility is the preferred, multi-purpose snapshot editor for sparse memory merging, virtual machine state inspection, and targeted state surgery on specific hardware platforms. The third utility implements the same memory-merge algorithm as a standalone legacy entry point and is explicitly deprecated; operators should migrate to the preferred snapshot editor for equivalent functionality.

The three utilities are intentionally small, focused programs. They do not embed the full hypervisor. They depend on the same core libraries as the production binary where overlap exists, which keeps serialization formats and syscall semantics aligned with what the runtime expects. This document treats them as one architectural surface because operators encounter them together in release bundles, documentation, and day-two workflows such as diff snapshot merging and custom seccomp policy iteration.

## Purpose & Boundaries

The security policy compiler exists to bridge the gap between policy authors who work in structured text and the runtime component that must install Berkeley Packet Filter bytecode into the kernel before restrictive syscall policies take effect. Its responsibility begins when a validated JSON document describing per-thread-category filters is available, and it ends when a serialized map of named filter blobs is written to disk. It is not responsible for deciding which syscalls the product allows in production; that policy lives in version-controlled resources and changes through the normal engineering process. It is also not responsible for installing filters at process startup, validating guest behavior, or handling runtime policy updates. If this compiler were unavailable or misconfigured, builds that embed filters would fail or produce unusable artifacts, and any workflow that relies on custom policies would be blocked. The running daemon could fall back to policies baked into an older build only if such a path exists; in practice, missing or corrupt compiled output is a build-time or release-packaging failure rather than a silent runtime degradation.

The snapshot editing utility exists to manipulate snapshot artifacts after the hypervisor has written them to disk. It covers three families of tasks. First, it merges diff snapshot memory layers onto a base memory image using sparse-file semantics so that operators materialize a full, resumable memory file without rewriting unchanged regions. Second, it prints diagnostic views of persisted virtual machine state, including semantic version information, per-virtual-processor views, and a full structured dump for deep inspection. Third, on one supported instruction set architecture it can load persisted state, remove selected hardware register entries from virtual processor state, and write a new state file, which supports migration and compatibility scenarios where specific register payloads must be stripped. The snapshot editor is not a replacement for the snapshot creation and load APIs exposed by the daemon. It does not validate that a merged memory image matches a particular state file, does not perform guest-side migration, and does not implement the full snapshot restore path inside the hypervisor.

The legacy memory rebase utility exists solely to copy non-hole data from a diff memory file onto a base memory file using the same kernel-assisted copy strategy as the snapshot editor’s memory merge path. It prints a deprecation warning and directs operators to the snapshot editor. Its continued presence supports older scripts and documentation; it should not receive new features. Boundaries are identical to the memory-merge subset of the snapshot editor: it touches files only, not structured state blobs.

What lies outside this trio is equally important. None of these tools schedule virtual machines, configure networking, or manage cgroups. None replace observability agents or log aggregation. The security compiler does not parse arbitrary BPF; it targets the seccomp ecosystem through the platform’s standard helper library for rule construction and export. The snapshot tools do not understand disk image formats or virtio devices beyond what is already encoded in persisted state handled by shared libraries.

## Interfaces & Contracts

The security policy compiler exposes a command-line interface that accepts a target architecture label, an input path, an optional output path, and an optional legacy mode flag that strips argument-level checks from compiled rules. The architecture label must correspond to a supported platform tuple used elsewhere in the project so that the emitted bytecode matches where the daemon will run. The input must conform to the JSON schema used across the repository’s policy resources: a map from thread category names to filter objects, each carrying default behavior, per-rule behavior, and a list of syscall rules with optional argument predicates. The compiler links against the system seccomp helper library at build time and emits a length-prefixed binary serialization using a fixed-endian, fixed-integer encoding with an explicit upper bound on deserialized size to mitigate denial-of-service via oversized inputs when filters are loaded later.

Consumers of the compiler’s output include packaging steps that embed the serialized blob and the runtime component that deserializes and installs filters when threads are brought up. The contract on the output side is stable within a release family: the serialized map must round-trip through the shared deserialization path used by the daemon. Callers must not hand-edit the binary output. On the input side, policy authors must keep syscall names and argument conditions consistent with what the guest kernel and libc actually issue; the compiler resolves syscall names to numbers for the chosen architecture and may fail if a name is unknown.

The snapshot editing utility exposes subcommands for memory rebase, optional virtual machine state surgery on supported hosts, and several inspection modes. Memory rebase takes writable base path and read-only diff path; it mutates the base file in place to reflect the layers encoded in the diff’s sparse layout. Inspection subcommands take a state file path and print to standard output; they do not modify files. The register-removal path takes input and output paths so that transformations are explicit and non-destructive relative to the original artifact. Contracts with the rest of the system flow through the same snapshot loading and saving abstractions as the hypervisor: files must be complete snapshots readable by the shared snapshot module, and any written file must be produced with the same version metadata that was read unless a deliberate upgrade path exists elsewhere.

The legacy rebase utility exposes only positional-style arguments mapping to base and diff paths through the shared argument parser used by older tooling. It requires the same file semantics as the snapshot editor’s merge operation: the base open for write, the diff open for read, and both on a filesystem that supports hole reporting primitives used to skip unallocated ranges efficiently.

## Data Flow

Incoming data for the security policy compiler is textual JSON describing filters. The compiler parses that structure into an in-memory representation of actions, syscall names, and optional argument comparisons, including comparison width hints that matter for correctness on certain libc implementations. That representation drives creation of a seccomp context for the requested architecture, accumulation of rules, and export of BPF instructions into an anonymous memory-backed file descriptor. The exported instructions are read back as raw machine words, grouped per named filter, and then serialized as a single map into the output file. No network or inter-process communication is involved; the transformation is entirely local and deterministic given the same inputs and library versions.

The following diagram situates the compiler between policy authors and the eventual runtime installation path. Arrows indicate primary data movement rather than control dependencies.

```
  Policy resources (JSON text)
            |
            v
  +---------------------+
  | Security policy     |
  | compiler (local     |
  | transform)          |
  +---------------------+
            |
            v
  Serialized BPF map (binary file)
            |
            +------> Build / packaging embed
            |
            +------> Custom policy workflows
            |
            v
  Runtime filter load (separate component)
```

For snapshot tooling, data flows from disk files into either byte-level merge logic or structured deserialization. In the memory rebase path, the diff file’s sparse regions drive which byte ranges are copied into the base file using a kernel copy mechanism; holes in the diff leave the corresponding base content unchanged, while written regions overwrite or extend the base. In inspection paths, the state file is loaded into a rich in-memory model that includes version information and virtual processor payloads; selected fields are then formatted for human review on standard output. In register removal, the structured model is transformed and written to a new file with the original version preserved. The legacy utility’s data flow is a subset: only the byte-level merge path appears.

```
  Base memory file (sparse)          Diff memory file (sparse)
            \                               /
             \                             /
              v                           v
           +-----------------------------------+
           | Merge engine (hole-aware copy)    |
           +-----------------------------------+
                           |
                           v
                Updated base memory file

  State file (binary snapshot) -----> Load into structured model -----> stdout or new file
```

Between the security compiler and snapshot utilities there is no shared data path. They are related operationally because both appear in release archives and both touch persistence boundaries of the product, but they do not exchange artifacts.

## Control Flow

Execution of the security policy compiler begins when an operator or build script invokes the binary with explicit arguments. The critical path is linear: open and read input, parse JSON, iterate each named filter, initialize and populate a seccomp context, export BPF, pack instructions, serialize the aggregate map, and exit successfully. Branching occurs inside rule handling: the legacy mode flag forces a simplified rule addition path that omits argument comparators, while the default path attaches optional argument conditions when present. Architecture parsing may fail early if the label is not recognized. Any failure in the underlying library returns a structured error upward without partial output files in the success path; failed runs may leave no output or incomplete files depending on when the failure occurred, which is why builds should treat non-zero exit codes as fatal.

Snapshot editor control flow starts with subcommand dispatch. Memory rebase runs the merge loop until the diff iterator reports no further data regions. Inspection subcommands load state once per invocation, then branch to a small formatter that either prints version text, per-virtual-processor details, or the entire structured state. Register removal loads state, applies a filter that drops specified register identifiers from each virtual processor’s register list while logging which requested identifiers were found, then saves to the output path. On hosts where architecture-specific editing is not compiled in, those subcommands are unavailable, which is a build-time branch rather than a runtime surprise on a given binary.

The legacy rebase utility parses arguments through the older parser, optionally handles help and version printing, emits a deprecation message on normal runs, opens files, and enters the same merge loop. Control flow is simpler than the snapshot editor because no subcommands exist.

## State & Lifecycle

The security policy compiler is effectively stateless across invocations. Per run, it holds in-memory buffers for input text, parsed JSON, accumulated BPF words per filter, and an output handle. It does not maintain a daemon or cache between executions. Startup is the process start; steady state is the compilation loop; shutdown closes file descriptors and exits. There is no recovery story beyond rerunning the command after fixing inputs.

The snapshot editor holds transient state only: open file handles for merge operations, deserialized models for inspection and editing, and buffers used by the kernel copy primitive. Lifecycle for a merge begins with opening files and ends with implicit flush on close; there is no multi-phase transaction across files. For state editing, lifecycle is load-transform-write; if save fails, the input file remains untouched because output goes to a separate path. Inspection avoids write state entirely.

The legacy rebase utility mirrors the merge lifecycle of the snapshot editor’s memory path with the same transient resource pattern.

From a product lifecycle perspective, these tools are versioned with the repository release. They are not hot-upgraded independently in production clusters; operators replace binaries when they upgrade the overall distribution.

## Failure Modes

Compilation failures are generally explicit and loud. JSON that does not match the schema fails during deserialization with parse errors. Unknown syscall names surface as resolution failures from the underlying library. Export failures indicate the policy could not be translated to BPF under current constraints. Output write failures propagate as I/O errors. A subtle class of issues arises when policies are accepted but semantically wrong for the workload: the compiler cannot infer application intent, so incorrect allow or deny choices manifest only when the guest misbehaves or legitimate syscalls are blocked at runtime. The legacy basic mode increases that risk by discarding argument checks, which is why it is deprecated for general use.

Snapshot merge failures include inability to open paths, seek failures if the filesystem does not support required operations, and copy failures from the kernel primitive. Sparse-file semantics depend on the backing filesystem; unexpected behavior there could theoretically yield surprising byte patterns, though the implementation is conservative about error checking after system calls. State load failures occur when files are truncated, version-incompatible, or corrupted; those bubble up as snapshot module errors. Partial writes to output state files could leave operators with an incomplete artifact if a failure happens mid-save after truncation; the shared save path should be atomic enough for typical use, but operators should still treat non-zero exits as failure and avoid using outputs.

The legacy utility shares merge failure modes with the snapshot editor. Its additional risk is operational: teams may miss deprecation warnings in automation logs and continue relying on a binary scheduled for removal.

Silent corruption is most likely when humans manually edit binary outputs or mix memory and state files from unrelated snapshot operations. The tools assume coherent artifact sets produced by the supported APIs.

## Operational Characteristics

All three utilities are lightweight processes. The security policy compiler’s CPU time scales with policy size and syscall rule count; memory use is bounded in part by explicit deserialization limits for the binary format when downstream components load filters, and by the size of exported BPF per filter. Peak memory is far below hypervisor-scale workloads. I/O is sequential reads and writes of small to medium files.

Snapshot merge operations are I/O bound and proportional to the number and size of populated regions in the diff file, not the logical size of sparse files. The kernel copy primitive avoids userspace buffering of entire guest memory images, which keeps resident memory small even for large guests. Inspection commands allocate structured state proportional to snapshot complexity; dumping very large states to the terminal can be verbose and slow, but that is an operator experience concern rather than a cluster scalability bottleneck.

Observability is primarily exit codes and human-readable error messages on standard error. These tools are not typically wired into metrics systems; they are invoked ad hoc or from scripts. The legacy utility always prints deprecation text on successful runs, which is visible operational noise intended to drive migration.

Scaling concerns are minimal because each invocation is independent. Parallel invocations should not target the same output files without external coordination. Build farms may parallelize compilation of different architectures or policy variants safely.

## Design Rationale

Splitting security policy compilation into a standalone utility keeps the heavy dependency on the seccomp helper library and the JSON policy surface out of the hot path of the running daemon except for deserialization and installation, which are comparatively small. Generating BPF at build time ensures reproducible artifacts, supports auditing before deployment, and avoids JIT compilation surprises on production hosts. Using a binary serialization with explicit limits aligns with defense-in-depth against malformed or maliciously large blobs at load time.

Providing a dedicated snapshot editor consolidates file-oriented snapshot workflows that operators were already doing with ad hoc tools. Implementing merge with hole awareness matches how diff snapshots are stored: only touched regions carry data, and sequential copying would waste time and bandwidth. Sharing deserialization logic with the hypervisor reduces format drift and avoids reimplementing version negotiation.

Keeping the legacy rebase utility for a time acknowledges long-lived operational scripts while steering new work toward the richer snapshot editor, which can grow subcommands without multiplying standalone binaries for every tweak.

The trio’s separation from the core daemon reflects a broader principle: build-time and maintenance tools should remain small, scriptable, and explicit about inputs and outputs, while the daemon focuses on steady-state request handling and guest execution. Together, these utilities let operators and integrators close the loop from policy text to enforceable bytecode and from layered snapshot files to inspectable, resumable system state, without expanding the attack surface or operational complexity of the runtime itself.
