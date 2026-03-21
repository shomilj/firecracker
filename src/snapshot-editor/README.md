# Snapshot editor

1. **Purpose and role in the larger system**

   1.1. A command-line utility sits at the edge of the Firecracker snapshot toolchain: it performs offline operations on memory images and on serialized virtual-machine state blobs without running a live VMM. The tool is intentionally narrow—each subcommand maps to one well-defined maintenance or inspection workflow rather than a general-purpose editor.

   1.2. Execution follows a uniform pattern: parse arguments, dispatch to a handler for the selected mode, perform I/O and transformations, and surface failures as human-readable errors on standard error while returning a non-zero exit status. Successful runs either mutate files in place or write new outputs, depending on the operation.

   1.3. The implementation is split along three conceptual axes—memory rebasing, optional machine-state surgery on AArch64, and read-only inspection of persisted VM state—plus a small shared layer for reading and writing the same snapshot container format the running hypervisor uses.

2. **Command-line surface and error model**

   2.1. The top level is modeled as a small set of subcommand trees. One tree targets guest physical memory files; another targets inspection of VM state snapshots; on AArch64 only, an additional tree targets editing VM state snapshots. This keeps unrelated concerns isolated at the CLI boundary so users never mix memory and vmstate flags accidentally.

   2.2. Errors are represented as a closed enum wrapping the specific failure modes of each subsystem. When the main entry point observes an error, it prints the display string to standard error and propagates the error outward so the process exits with failure. There is no partial success reporting: a failed rebase or a failed load leaves the user with a single coherent message.

   2.3. Optional tracing can be enabled at build time through feature flags that thread instrumentation into shared utility crates; the default build path is lean and suitable for distribution as a static helper binary next to Firecracker itself.

3. **Memory rebasing: sparse diffs overlaid on a base image**

   3.1. **Problem statement.** Snapshot workflows sometimes produce a “base” guest RAM file and a separate “diff” that captures only the regions that diverge from that base. The diff is stored as a sparse file: unwritten regions are holes; written regions are data. Rebasing means: for every extent of real data in the diff, copy those bytes into the base file at the same byte offsets, so the base file becomes the logical union of “whatever was there before” plus “whatever the diff says changed,” without rewriting unchanged regions unnecessarily.

   3.2. **Why sparse semantics matter.** On typical Linux filesystems, `SEEK_DATA` advances to the next region that contains non-hole bytes, and `SEEK_HOLE` advances to the next region that is a hole (or end-of-file). Iterating data extents this way avoids scanning the entire file linearly and avoids copying gigabytes of zeros when the diff is mostly empty. The algorithm is therefore I/O-pattern-driven rather than size-driven.

   3.3. **Control flow.** A running offset cursor starts at the beginning of the diff. The loop asks the kernel for the next data region; if none exists, work is complete. For each data region, the end is found by seeking for the next hole (or, if there is no further hole, by using the file length as the end). Within `[block_start, block_end)`, bytes are moved from the diff into the base at identical offsets.

   3.4. **Transfer mechanism.** Rather than read into userspace buffers and write back, the implementation uses the kernel’s zero-copy file-to-file copy primitive: the destination is seeked to the current offset, then a chunk is transferred from the diff’s file descriptor to the base’s file descriptor. The kernel updates a shared offset cursor across the call, so repeated invocations drain the current data extent until the cursor reaches the end of that extent. Any negative return from the syscall is turned into a standard I/O error for reporting.

   3.5. **Interaction with holes in the base.** Regions of the base that correspond to holes in the diff are never touched by this loop. If the base was shorter than a write implied by a long diff, seeking and copying implicitly extends the base as needed. If the diff contains explicit zero-filled blocks (materialized data, not holes), those zeros overwrite the corresponding base range—semantically distinct from “leave the base unchanged,” which is what a hole means.

   3.6. **Correctness intuition.** Unit tests exercise empty inputs, diffs that are entirely hole (base unchanged), diffs that are entirely data (base replaced), and interleaved patterns at filesystem block granularity (e.g. 4 KiB and 8 KiB) where base and diff disagree on which regions are data vs hole, including cases where one file is longer than the other. Together they encode the expected overlay semantics for offline tooling.

4. **VM state I/O: the shared snapshot container**

   4.1. **Loading.** Opening a VM state path yields a reader positioned at the start of the file. The file’s length is taken from metadata and passed into the loader so the format parser knows how many bytes are available. The loader returns two pieces of information: a fully hydrated in-memory representation of the microVM’s persisted state (vCPUs, devices, and related structures as modeled by the hypervisor crate), and a semantic version value carried in the snapshot header. That version is not reinterpreted by the editor—it is preserved so that round-tripped files remain compatible with the same snapshot revision expectations as the source.

   4.2. **Saving.** Writing accepts the in-memory state plus an explicit version. The writer constructs a snapshot encoder configured with that version, serializes the state through it, and writes to a destination opened with create-and-truncate semantics so outputs are always replaced atomically from the user’s perspective (old content is not appended to).

   4.3. **Format coupling.** The on-disk layout (magic identifier, version string, serialized state payload, optional integrity check) is owned by the hypervisor’s snapshot module. The editor does not duplicate serialization logic; it delegates entirely so that any evolution of the format, limits, or validation rules stays centralized.

5. **Inspecting VM state (read-only commands)**

   5.1. **Version query.** The lightest inspection path reads only enough to recover the header’s version string and prints it in a conventional `vMAJOR.MINOR.PATCH` form. This supports quick scripting around snapshot compatibility without dumping large structures.

   5.2. **Per-vCPU views.** A deeper mode loads the full state and prints each vCPU’s state using the debug representation provided by the underlying types. This is aimed at developers diagnosing register sets, not at stable machine parsing—the output format follows Rust’s pretty-debug conventions.

   5.3. **Full state dump.** The broadest mode prints the entire deserialized microVM snapshot structure via the same debug path. It is the most verbose option and is appropriate when correlating device state with vCPU state in complex snapshots.

6. **Editing VM state on AArch64 (register filtering)**

   6.1. **Scope.** Stateful edits are limited to manipulating the register lists attached to each vCPU inside the persisted AArch64 view. Other snapshot fields are carried through unchanged by the generic load-transform-save pipeline.

   6.2. **Pipeline.** The editor loads state and version, applies a pure transformation function that returns new state (or fails), then saves to a distinct output path with the original version. Input and output paths are separate so that failed attempts never destroy the only copy of a snapshot; the user opts in by choosing the destination.

   6.3. **Register removal semantics.** The user supplies a set of KVM register identifiers (parsed flexibly, including hexadecimal). For each vCPU, the implementation builds a new register vector by retaining every register whose identifier is not in the removal set. Registers are matched by exact identifier equality; duplicates in the user’s removal list are harmless, and identifiers that never appeared in a given vCPU are reported as “not present” rather than treated as errors.

   6.4. **Observability.** For each vCPU, the tool logs to standard output which logical CPU index is being modified, then emits one line per requested register id indicating whether it was removed or absent. This makes batch removals auditable when trimming incompatible registers before loading a snapshot on a different kernel or KVM version.

   6.5. **Platform gating.** The edit path is compiled only for AArch64 because it depends on AArch64-specific register representations in the hypervisor model. Other architectures continue to use the inspection commands without exposing a no-op or misleading edit surface.

7. **Cross-cutting design properties**

   7.1. **Separation of concerns.** Memory operations never parse VM state; VM state operations never interpret guest RAM bytes beyond what the snapshot codec already does. The only shared substrate is the snapshot load/save helpers used by both inspection and editing.

   7.2. **Determinism and side effects.** Rebase modifies the base memory file in place. VM state editing always writes a new file. Inspection commands are read-only with respect to the inputs.

   7.3. **Failure modes.** Memory paths fail on open/seek/sendfile errors with explicit stages (open base, open diff, seek in diff, seek in base, transfer). VM state paths fail on open, load, output open, or save errors, with errors wrapped so the user sees a single chain from the utility’s perspective.

8. **Conceptual data-flow diagrams**

   8.1. **Memory rebase**

   ```
        +------------------+     SEEK_DATA / SEEK_HOLE      +------------------+
        |  diff (sparse)   | ------------------------------> |  extent [s,e)    |
        +------------------+                                 +------------------+
                    |                                                |
                    | sendfile chunking at offset cursor             |
                    v                                                v
        +------------------+     seek + write via kernel      +------------------+
        |  base (mutable)  | <------------------------------ | same byte offsets |
        +------------------+                                 +------------------+
   ```

   8.2. **VM state load–transform–save**

   ```
        +-------------+    deserialize     +------------------+
        | snapshot on | ------------------> | live snapshot    |
        |    disk     |                     | model + version  |
        +-------------+                     +------------------+
                                                    |
                                                    | pure transform
                                                    | (e.g. filter regs)
                                                    v
        +-------------+    serialize        +------------------+
        | new snapshot| <------------------ | updated model    |
        |    on disk  |    (same version)   +------------------+
        +-------------+
   ```

   8.3. **Inspection-only path**

   ```
        +-------------+    load            +------------------+
        | snapshot on | ------------------> | snapshot model + |
        |    disk     |                     | version          |
        +-------------+                     +------------------+
                                                    |
                    +-------------------------------+-------------------------------+
                    |                               |                               |
                    v                               v                               v
             print version only              print each vCPU                  print full state
   ```
