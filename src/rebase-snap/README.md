# Rebase-snap

1. **Purpose and problem domain**

   1.1. The executable implements a single, narrowly scoped operation: it applies a sparse “diff” view of memory snapshot bytes onto an existing “base” snapshot by overwriting only those byte ranges where the diff actually carries data. In virtualization and checkpointing pipelines, memory snapshots are often stored as sparse files so that unmodified or zero-filled regions do not consume backing storage. A subsequent incremental or layered representation may record only the regions that diverged from an earlier image. This tool closes that loop by taking such a diff-shaped sparse file and merging its non-hole content into a writable base file at matching offsets, so the base file’s on-disk layout reflects the union of “keep what was already there” for holes and “take the diff bytes” for data regions.

   1.2. The operation is in-place on the base side: the base is opened for read-write and mutated; the diff is read-only and serves only as the source of bytes to copy. No new output file is created by the tool itself—the base path is both input and output for the merged result.

   1.3. The user-facing surface is intentionally minimal: exactly two path arguments identify the base and the diff. Help and version modes exist; normal execution always prints a deprecation notice directing operators toward a successor workflow (“snapshot-editor”), signaling that new integrations should not assume long-term availability of this binary even though it remains functional.

2. **Sparse-file semantics as the driver of control flow**

   2.1. On Unix systems that support it, a file may contain “holes”: ranges that are logically zero or unallocated from the application’s perspective but need not occupy physical blocks. The tool does not read the diff byte-by-byte across its full length. Instead, it walks the diff using kernel-assisted sparse semantics: advance to the next region that contains real data, then determine where that region ends (the start of the next hole or end-of-file). Only those intervals are copied.

   2.2. The outer loop structure is therefore driven entirely by the diff’s data/hole map. An empty diff or a diff that consists only of holes results in zero copy operations—the base is left unchanged (aside from any side effects of opening it). A fully dense diff from offset zero to EOF behaves like a sequential copy of the entire file length, modulo the transfer mechanism described later.

   2.3. This design encodes an important invariant: **holes in the diff mean “do not overwrite the base at these offsets.”** The base retains whatever bytes it already had (including its own sparseness pattern) for those ranges. Conversely, **data regions in the diff always overwrite the corresponding offsets in the base**, regardless of whether the base previously held data or a hole at those positions.

3. **Merge algorithm**

   3.1. A running offset cursor starts at the beginning of the logical file. Each iteration locates the next data region in the diff at or after the current cursor by querying “next data” semantics from the current position. If no such region exists, the merge completes successfully.

   3.2. For a located data region, the end boundary is resolved by seeking to the next hole after the region’s start. If the file has no trailing hole (dense to EOF), the end is taken as the diff’s reported length. The half-open interval `[region_start, region_end)` is the contiguous range of bytes to synchronize from diff to base.

   3.3. Within that interval, bytes are transferred in one or more kernel chunks using a zero-copy path: the base descriptor is positioned to the same offset as the logical merge cursor, and a system call moves bytes from the diff descriptor into the base descriptor while atomically advancing a shared offset variable that tracks the current position in both streams for that segment. The count passed to each transfer is the remaining length in the current data region, possibly split across multiple calls if the kernel returns partial progress (the loop continues until the cursor reaches `region_end`).

   3.4. Error handling is strict: any failure in seeking (whether for positioning the base, finding data, or finding holes), any failure retrieving file size for the edge case at EOF, or any negative result from the bulk transfer syscall maps to a typed error that surfaces to the user on standard error and yields a non-zero process exit. There is no partial-success mode—an I/O error aborts the whole operation.

4. **Why alignment of offsets matters**

   4.1. The merge cursor is a single scalar that simultaneously represents “where we are in the diff” for read-side semantics and “where we are writing in the base.” That is only correct if both files describe the **same linear address space** starting at offset zero: snapshot slot *N* in the diff corresponds to snapshot slot *N* in the base. The tool does not remap, pad, or relocate regions; it assumes identical logical lengths and alignment of semantic content.

   4.2. If the diff is shorter than the base, later offsets in the base are never touched—there is no data in the diff beyond its EOF to iterate. If the diff is longer, data regions that extend past the former base length will still be written through the bulk transfer, which extends the base as needed by the kernel write semantics (subject to filesystem behavior). Tests in the crate explicitly cover “diff longer than base” and “base longer than diff” scenarios to lock in this behavior.

5. **Interaction between holes, explicit zeros, and “unchanged” regions**

   5.1. A hole is not the same as a block of zero bytes stored as data. If the diff contains an explicit run of zero-valued bytes that occupies real storage, that range counts as **data** for sparse-walking purposes and will overwrite the base. If the same logical zeros were represented as a hole instead, the base would retain prior content for that range. Operators must therefore understand how their snapshot pipeline materializes zeros—this tool does not interpret guest memory semantics; it interprets **storage** semantics.

   5.2. When the diff has a hole where the base had data, the base’s data remains after the run—this is how “unchanged from the previous layer” is expressed. When the diff has data where the base had a hole, the hole is filled with the copied bytes. When both have data, the diff wins for that range.

6. **High-level data movement (conceptual)**

```
  Diff (sparse)                         Base (read-write)
  +---+---+---+---+---+                 +---+---+---+---+
  | D |   | D |   | D |   seek_data /   | ? | ? | ? | ? |
  +---+---+---+---+---+   seek_hole     +---+---+---+---+
        |                       |               ^
        |    sendfile-style     +---------------+
        |    copy per data run  (same offsets)
        v
  Only "D" intervals are read and written; gaps leave base as-is.
```

   6.1. The diagram compresses many details (multiple disjoint data runs, partial transfers, EOF handling) but captures the essential pipeline: discovery of `D` intervals on the diff side, then targeted writes into the base at matching offsets.

7. **Command-line interface and validation**

   7.1. Parsing is delegated to a shared argument parser used elsewhere in the workspace. Two positional parameters are required; both must be present for normal execution. If either path cannot be opened as intended (writable for the base, readable for the diff), the error is classified as a file-open failure for the corresponding role.

   7.2. Help output includes the tool version string, a one-line description of copying non-sparse sections from diff onto base, the deprecation banner, and formatted help for the two parameters. Version output similarly includes version and deprecation text. Thus even successful informational invocations reinforce migration away from this binary.

8. **Error taxonomy (behavioral)**

   8.1. Errors are grouped at two levels: a file-operation layer distinguishes invalid base open, invalid diff open, seek-to-data failures, seek-to-hole failures, generic seek failures, bulk-transfer failures, and metadata read failures. Above that, a top-level type separates argument parsing problems from snapshot open problems and from merge-time failures, so that messaging can remain specific without conflating user mistakes with I/O faults.

9. **Platform assumptions**

   9.1. The implementation targets Unix-like environments where sparse seek helpers and the bulk-transfer syscall used are available; the seek helpers are provided through a small utility trait, and the transfer uses the 64-bit offset variant of the classic zero-copy API. Non-Unix platforms are out of scope for this crate as structured.

10. **Testing strategy (what is guaranteed)**

    10.1. Unit coverage exercises argument binding by forcing bad base path and bad diff path and asserting the correct error class, then a happy path where both open.

    10.2. Merge tests include degenerate cases: both empty; diff sized but all holes (base content must survive); diff dense with data only (base becomes an exact copy of diff bytes). Larger tests vary block sizes (aligned to typical filesystem minimum hole punch sizes) and interleave patterns: data in both; data in base with a hole in diff; data in base with explicit zero block in diff; extending diff beyond base; extending base beyond diff. Expected results are built as byte vectors and compared after the merge with positioned reads, which validates end-to-end offset behavior without relying on internal stepping details.

11. **Optional build features**

    11.1. The crate declares an optional tracing feature that pulls in instrumentation hooks from shared workspace crates. When enabled, the binary can participate in broader logging/tracing topologies; when disabled, the dependency surface stays minimal for default builds.

12. **Operational summary**

    12.1. In operation, the tool is a thin, deterministic sparse merge: **walk diff data regions → overwrite base at identical offsets → stop at EOF on the diff side.** It encodes snapshot-layer semantics at the storage level, fails hard on I/O errors, and remains available only as a deprecated stepping stone toward the maintained replacement workflow.
