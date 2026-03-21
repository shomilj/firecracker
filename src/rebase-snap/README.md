# Rebase-snap

1. **Purpose**

   1.1. The executable performs one storage-level operation: merge a sparse “diff” snapshot of guest memory bytes into an existing “base” snapshot by copying only those byte ranges where the diff actually holds data. Checkpointing stacks often keep memory images as sparse files so unchanged or logically zero regions do not consume backing blocks; incremental layers may record only divergences from an earlier image. Rebasing applies that layer onto a writable base at matching offsets so the base’s on-disk layout reflects “preserve base bytes wherever the diff has a hole” and “take diff bytes wherever the diff has data.”

   1.2. The base file is both the merge target and the output: it is opened read-write and updated in place. The diff is read-only. No separate output path exists; operators choose the base path knowing it will be mutated.

   1.3. Every successful run that actually performs the merge first prints a deprecation notice on standard output, steering new integrations toward the maintained successor workflow bundled with the broader snapshot tooling. Help and version modes also surface the same deprecation text so even informational invocations reinforce migration.

2. **Merge algorithm**

   2.1. **Initialization.** A single 64-bit merge cursor starts at offset zero in the logical byte stream of the diff. The algorithm never scans the diff linearly from start to end in fixed steps; it only jumps from one data extent to the next.

   2.2. **Outer loop: discover the next data extent.** From the current cursor, the implementation queries the kernel for the start offset of the next region that contains stored data (not a hole). If no such region exists, the merge completes successfully: nothing further is copied.

   2.3. **Bound the current extent.** Once a data region starts at some offset, the end is found by seeking from that start to the beginning of the next hole. If the file has no trailing hole (data runs all the way to logical end-of-file), the end offset is taken as the file length reported by metadata. The work item for this iteration is the half-open interval from region start up to but not including region end.

   2.4. **Inner loop: drain the extent.** While the merge cursor is still strictly less than region end, the base descriptor is positioned to the same absolute offset as the cursor. A bulk zero-copy transfer moves bytes from the diff descriptor into the base descriptor. The transfer API uses a pointer to a 64-bit offset that the kernel advances on the read side so that repeated calls continue where the previous call stopped; that offset is the same variable as the merge cursor, which keeps diff-side read position and “which base offset we are writing” aligned. If the kernel returns a short count, the inner loop retries until the cursor reaches region end or an error occurs.

   2.5. **Advance to the next extent.** After an extent is fully drained, the outer loop again asks for the next data region at or after the cursor. The cursor always moves forward; there is no backward seek in normal operation.

   2.6. **Termination and empty inputs.** Empty diff and empty base both yield a vacuous outer loop (no data regions) or inner loops of zero width. A diff that has been extended to a large logical size but contains only holes produces no copy operations: the base remains byte-for-byte as before (aside from open metadata effects).

   2.7. **Strict failure semantics.** Any failure while opening either file, any seek used for data/hole discovery or for positioning the base, any metadata read needed for the end-of-file edge case, or any negative result from the bulk transfer maps to a typed error printed on standard error and a non-zero exit. There is no resume-from-checkpoint mode and no partial-success flag.

3. **Hole versus zero semantics**

   3.1. **Holes mean “leave the base unchanged.”** If a byte range in the diff is a hole, the merge never writes those offsets. Whatever the base already contained—older snapshot bytes, another hole, or a mix—remains. That is how a diff expresses “unchanged from the layer below” without storing redundant bytes.

   3.2. **Allocated zeros are data.** If a range is stored as real blocks filled with zero bytes, sparse discovery treats that range as data. The merge overwrites the base with those zeros. That is not equivalent to a hole of the same logical length: zeros win over the previous base contents; holes do not.

   3.3. **Pipeline and block size.** Whether a run of zeros is punched as a hole or kept as allocated data depends on how the diff was produced and on filesystem behavior (minimum hole size, fallocate patterns, etc.). The tool does not interpret guest memory meaning; it interprets the storage map. Empirical tests in the crate use runs at least as large as typical minimum hole sizes on common Linux configurations so that “hole in diff” cases are actually sparse at the storage layer.

   3.4. **Interaction matrix (conceptual)**

   ```
        diff has     base had      after merge
        --------     --------      -----------
        hole         anything      base unchanged there
        data         data          diff data wins
        data         hole          hole filled with diff bytes
        zeros(data)  anything      base gets zeros there
   ```

4. **Alignment and offset correspondence**

   4.1. **Single address space.** The merge cursor is one scalar that simultaneously tracks the read position in the diff (via the kernel-updated offset passed into the bulk transfer) and the write position in the base (explicit seek before each chunk). Correctness requires that byte offset *k* in the diff always maps to byte offset *k* in the base. The tool never adds headers, skips padding, or applies sliding windows.

   4.2. **Relative length.** If the diff is shorter than the base, high offsets in the base are never written because the diff has no data regions there. If the diff is longer, data regions past the former base end extend the base file as writes proceed. Tests cover diff longer than base and base longer than diff with interleaved hole and data patterns at aligned block sizes.

   4.3. **No cross-region reordering.** Data regions are processed in ascending offset order as returned by sparse seeks; within a region, copying is strictly increasing in offset.

5. **Command-line interface**

   5.1. **Argument model.** Parsing uses the same hand-built argument parser as other related operator tools. Two required parameters name the base file and the diff file. Each is passed as a flag-style argument with a long name (base first in help text, diff second); both values must be present for a normal merge.

   5.2. **Help mode.** When the user asks for help, the program prints the tool name and version line, a one-line description of copying non-sparse sections from diff onto base, the deprecation banner, and formatted help listing the two required parameters. No merge runs.

   5.3. **Version mode.** When the user asks for version, the program prints version and the deprecation banner on standard output and exits without merging.

   5.4. **Default run.** After parsing succeeds and neither help nor version was requested, the deprecation line is printed, then both paths are opened (base writable, diff readable), then the merge runs. Silent success means only the deprecation line on standard output unless the environment or shell adds its own noise; errors go to standard error.

   5.5. **Validation behavior.** Open failures are classified by which role failed (base versus diff) so messaging can distinguish permission or missing-file problems on the writable side from the read-only side.

6. **Error taxonomy (behavioral)**

   6.1. **File-operation layer.** Distinct variants cover: invalid base open, invalid diff open, failure to seek to the next data extent, failure to seek to the next hole, generic seek failure while positioning the base, failure to read file metadata for length, failure in the bulk transfer syscall.

   6.2. **Top-level grouping.** A second level separates argument parsing failures (with a hint to retry help), failures that occur while opening inputs before merge, and failures that occur during the merge loop. Parsing errors include a trailing hint line for discoverability.

7. **Platform assumptions**

   7.1. The implementation targets Unix-like systems where sparse seek helpers and the 64-bit-offset bulk transfer API exist. Non-Unix targets are out of scope for this crate as structured.

8. **Testing strategy**

   8.1. **Argument binding.** Tests exercise bad base path and bad diff path and assert the corresponding open error class, then a happy path where both open.

   8.2. **Merge coverage.** Tests include empty both sides; diff sized to match base but all holes (base content preserved); diff dense with data only (base becomes an exact copy of diff bytes). Larger tests vary block sizes (aligned to typical filesystem hole behavior) and interleave: data in both; data in base with hole in diff; data in base with an explicit zero block in diff; diff extending past base; base extending past diff. Assertions read back file bytes at offsets and compare to expected vectors so correctness is end-to-end rather than tied to internal stepping.

9. **Optional build features**

   9.1. An optional tracing feature wires in instrumentation from shared utility crates; when disabled, the dependency surface stays minimal for default builds.

10. **Operational summary**

   10.1. **One sentence.** Walk diff data extents in order, copy each extent’s bytes into the base at the same offsets, stop when no further data extents exist, fail hard on any I/O error, and remind operators that the standalone binary is deprecated in favor of the integrated workflow.

11. **End-to-end data movement diagram**

   ```
        DIFF (sparse, read-only)              BASE (read-write)

        offset:  0    4K   8K   12K         same offset axis
                 |----|hole|----|data|----|
                          \___________/
                          one seek_data/seek_hole pair
                                   \
                                    \  bulk copy chunks
                                     ----------------------> base seeked to
                                                             same offsets

        Regions skipped by design: every gap where diff is hole.
   ```
