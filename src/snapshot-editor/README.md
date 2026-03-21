# Snapshot editor

1. **Role at the toolchain boundary**

   1.1. A small command-line binary performs offline work on two kinds of artifacts that accompany Firecracker-style snapshots: guest physical memory images and serialized microVM state. Nothing in the process runs a live VMM or touches the kernel virtual machine interface; the tool is a maintenance and inspection layer that assumes the same on-disk formats the hypervisor already uses.

   1.2. Work is organized into three conceptual lanes that remain isolated at the user interface: (a) applying a sparse memory diff onto a mutable base image, (b) read-only inspection of persisted VM state at several depth levels, and (c) on AArch64 only, a surgical edit that rewrites selected virtual CPU register entries in a snapshot before saving to a new file. A shared codec path loads and saves machine state so inspection and editing never fork the serialization format.

   1.3. The overall process model is: parse arguments, dispatch to exactly one handler, perform I/O, and on failure print one human-readable reason on standard error and exit non-zero. Success either mutates the base memory file in place, writes a new VM state file, or prints to standard output only.

2. **Top-level command surface**

   2.1. The root command is split into subcommand trees. One tree is dedicated to memory; it currently exposes a single operation that pairs a writable base path with a read-only diff path. Another tree is dedicated to VM state inspection and exposes three modes that differ only in how much of the deserialized state is printed. On AArch64 builds only, a third tree exposes VM state editing; on other architectures that tree is absent so the binary cannot advertise a no-op edit path.

   2.2. Naming and flags follow the usual conventions: short and long options for paths, with no mixing of memory and VM-state flags in a single invocation because they belong to different subcommand trees.

3. **Error representation and surfaces**

   3.1. Failures are modeled as a closed enum at the top level that wraps the specific error type of whichever lane ran. Memory operations have their own enum whose variants map one-to-one to observable stages: opening the base for write, opening the diff for read, advancing to the next data region in the diff, finding the end of that region via hole semantics or file length, seeking the base to the active offset, and bulk-transferring bytes. Any negative result from the bulk transfer is converted to the standard I/O error type for that stage.

   3.2. VM state paths share a common utility error type for open, metadata read, decode, output create/truncate, and encode. Inspection and editing both depend on that type; the edit path additionally wraps it only where the edit pipeline needs a uniform error propagation shape. Load failures originate in the snapshot decoder; save failures originate in the encoder.

   3.3. The main entry point prints the display string of any error to standard error and returns the error again so the process exits with failure. There is no partial success reporting: a failed rebase leaves whatever partial writes the kernel already applied (same as any in-place file writer), and a failed save after a successful load does not pretend the run succeeded.

   3.4. Optional tracing exists behind a compile-time feature that pulls instrumentation into shared crates; default builds omit it so the binary stays lean for distribution next to the hypervisor.

4. **Sparse memory rebase: problem and semantics**

   4.1. **Layered memory snapshots.** Many workflows store a large “base” guest RAM image and a separate “diff” that records only bytes that diverge from that base. The diff is often stored as a sparse file: ranges that match the conceptual layer below are left as holes; ranges that changed are written as real data. Rebasing merges the diff into the base so the base becomes the logical union of the prior base and the diff’s non-hole bytes at identical offsets.

   4.2. **Why iteration follows data extents.** A sparse diff may be huge in logical size but almost empty on disk. Walking byte-by-byte would waste I/O and CPU. Instead, the implementation asks the kernel for the next interval that contains stored data, then finds where that interval ends (start of the next hole or end-of-file). Only those half-open intervals are copied. Control flow is therefore driven by the diff’s sparse map, not by the nominal file length.

   4.3. **Cursor and region boundaries.** A single 64-bit cursor advances along the diff’s logical byte range. Each outer iteration sets the cursor to the start of the next data region returned by “seek to data” semantics from the current position. If no such region exists, the merge completes. For each data region, the end offset is either the position of the next hole after the region start, or, if the file is dense to EOF, the file’s reported length from metadata. The interval from region start up to region end (half-open) is the contiguous range to synchronize.

   4.4. **Transfer mechanism and kernel contract.** For each byte remaining in the current data region, the writable base is positioned to the same offset as the logical cursor on the diff side. A single syscall copies from the diff’s descriptor to the base’s descriptor. The syscall takes a pointer to a 64-bit offset that the kernel updates to reflect how far reading has progressed in the source file for this transfer; the implementation passes the same cursor variable used to track the merge position. That pointer must refer to the same in-memory offset word the outer loops use so that partial transfers (when the kernel returns fewer bytes than requested) resume correctly without duplicating or skipping ranges.

   4.5. **Inner loop until the region is drained.** While the cursor is still before the region end offset, the base is re-seeked to the current cursor value, then another chunk is requested with a count equal to the remaining bytes in the region (converted to a machine word size with saturation for safety). A negative return from the syscall is turned into an error for the transfer stage. The loop continues until the cursor reaches the region end, then the outer loop asks for the next data region starting from the cursor (which now sits at the end of the drained extent).

   4.6. **Holes versus explicit zeros.** A hole in the diff means “do not change the base at these offsets.” The base may still contain data, another hole, or a mix; those bytes are untouched. If the pipeline instead materializes zeros as real allocated blocks, those ranges count as data for sparse walking and overwrite the base with zeros—semantically different from leaving the base unchanged. That distinction is storage-level, not guest-memory semantics.

   4.7. **Length and extension behavior.** If the diff is shorter than the base, offsets beyond the diff’s EOF are never visited because there is no data region there. If the diff is longer, writing through the bulk transfer at offsets past the old base length extends the base file as the filesystem and write semantics allow. Tests lock in patterns at typical block granularities (for example 4 KiB and 8 KiB) where the filesystem reliably punches holes, including cases where the diff grows past the base and where the base grows past the diff.

   4.8. **Invariants**

   4.8.1. The base and diff describe the same linear address space starting at offset zero: offset *N* in the diff corresponds to offset *N* in the base. There is no remapping, relocation, or padding.

   4.8.2. The base is read-write; the diff is read-only. The base is the only file mutated by the merge.

   4.8.3. The merge is idempotent in the sense that a diff consisting entirely of holes produces no copy operations and leaves the base unchanged (aside from open side effects).

   4.8.4. Where the diff has data, that data wins for that offset range regardless of what the base had before.

5. **VM state container: load and save**

   5.1. **Loading.** Opening a snapshot path yields a reader; the implementation reads the file’s length from metadata and passes that length into the snapshot loader so the decoder knows how many bytes remain. The loader returns a fully hydrated in-memory microVM state plus a semantic version value from the header. The editor does not reinterpret the version for its own logic beyond threading it through on save; it exists so round-tripped files remain compatible with the same snapshot revision expectations.

   5.2. **Saving.** Saving takes the in-memory state and an explicit version, constructs a snapshot writer bound to that version, and writes to a destination opened with create/truncate semantics so each run replaces the previous output file rather than appending.

   5.3. **Format ownership.** The on-disk layout (magic, version, payload, checksum rules) lives in the hypervisor’s snapshot module. The editor delegates entirely so that validation limits, evolution of the wire format, and error taxonomy stay centralized.

6. **Inspection modes (read-only)**

   6.1. **Version-only.** The shallowest path loads the snapshot and prints a single line beginning with the letter “v” and continuing with the semantic version string (major.minor.patch style). This is intended for quick scripting and compatibility checks without dumping registers or devices.

   6.2. **Per-vCPU state.** A deeper mode loads the full state and prints each virtual CPU index with a labeled block, then the debug representation of that CPU’s state. The format follows Rust’s pretty-debug conventions for the underlying types; it is meant for human diagnosis, not stable machine parsing.

   6.3. **Full microVM state.** The broadest mode prints the entire deserialized structure via the same debug path. Useful when correlating device state with multiple virtual CPUs; verbosity is highest.

   6.4. **Common properties.** All three modes load through the same open path, so failures on corrupt files, truncated inputs, or version mismatches surface identically. None of the inspection modes write files.

7. **AArch64 VM state editing**

   7.1. **Compile-time scope.** The edit subcommand is compiled only when the target architecture is AArch64 because it depends on AArch64-specific register vector types in the hypervisor model. Other architectures do not see the command at all.

   7.2. **Pipeline.** The editor loads state and version, applies a pure transformation that returns new in-memory machine state or fails, then saves to a user-chosen output path with the same version as the input. Input and output paths are distinct so a failed edit never overwrites the only copy unless the user explicitly pointed the output at the same path as the input.

   7.3. **Single operation: register removal.** The user supplies a list of numeric register identifiers in the encoding used by the Linux virtual CPU ioctl layer for AArch64. The parser accepts decimal or hexadecimal forms (flexible hex parsing). For each virtual CPU, the implementation builds a new register vector by retaining every register whose numeric identifier is not in the removal set. Matching is by exact integer equality on the identifier.

   7.4. **Duplicate and missing identifiers.** If the user lists the same identifier more than once, only the first occurrence participates in the “removed” bookkeeping; the rest are redundant. If an identifier is not present for a given virtual CPU, that is not an error: after processing, the tool prints one line per requested identifier for that CPU, marking either “removed” or “not present.”

   7.5. **Observability.** For each virtual CPU, a line announces which logical index is being modified. Then, for each requested register id in order, a line states whether it was removed or absent. Standard output is therefore suitable for light auditing when trimming registers before loading on a different kernel or hypervisor revision.

   7.6. **What is not edited.** Only the per-virtual-CPU register lists are rebuilt. Other snapshot fields (devices, memory layout metadata inside the state blob, and so on) pass through unchanged except as incidentally carried by cloning the rest of the structure.

8. **Cross-cutting properties**

   8.1. **Separation.** Memory rebasing never parses VM state. VM state commands never interpret guest RAM bytes except as the snapshot codec already does. The only shared substrate is the snapshot load/save helpers.

   8.2. **Determinism.** Rebase mutates the base file in place. Editing always writes a new file (or truncates an existing path). Inspection is read-only with respect to inputs.

   8.3. **Failure staging.** Memory errors name the stage (which seek failed, which open failed). VM state errors name whether the problem was open, metadata, load, output open, or save.

9. **Diagrams**

   9.1. **Sparse rebase data flow**

   ```
        diff (sparse)                    logical cursor
        +----+ hole +----+ data +----+   starts at 0
        |    |      |DDDDDDDD|    |      seek_data -> start of D
        +----+      +--------+    +      seek_hole -> end of D (or EOF len)
                    |<- copy ->|        bulk transfer advances cursor in diff
                              v
        base (writable)         same offsets written
        +----+ old +----+ ? +----+
        |    |      |DDDDDDDD|    |
        +----+      +--------+    +
   ```

   9.2. **VM state load–transform–save (edit)**

   ```
        bytes on disk                    hydrated machine state + version
              |  deserialize                    |
              v                                 v
        +-------------+              +----------------------+
        |   snapshot  | ---------->| in-memory model      |
        |   bytes     |            +----------------------+
        +-------------+                      |
                                               | filter registers (AArch64)
                                               v
                                     +----------------------+
                                     | updated model        |
        +-------------+              | same version         |
        | new bytes   | <----------- +----------------------+
        | on disk     |   serialize
        +-------------+
   ```

   9.3. **Inspection fan-out**

   ```
        load once
            |
            +----> print version only
            |
            +----> for each virtual CPU: print debug CPU state
            |
            +----> print full machine-state debug view
   ```
