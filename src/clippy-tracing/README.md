# clippy-tracing architecture

1. Purpose and problem shape

   1.1. The tool is a batch-oriented source transformer and linter for Rust crates. It walks a directory tree, opens each Rust source file, parses it into a syntax tree, and applies one of three mutually exclusive operations: verify that every eligible function carries a particular instrumentation attribute, insert that attribute where it is missing, or remove matching instrumentation attributes from the source text.

   1.2. The design assumes repositories large enough that hand-editing thousands of function items is impractical. Automation therefore favors predictable, mechanical rules over semantic understanding of call graphs or runtime behavior. The tool does not compile the crate; it only parses and rewrites text according to syntactic patterns.

   1.3. Despite naming that suggests integration with the Clippy linter, the implementation is a standalone binary driven by explicit command-line actions. It does not embed inside `rustc` or Clippy; it is an external pass over `.rs` files.

2. External interface: actions, flags, and process outcomes

   2.1. The binary accepts a required `--action` with exactly one of three enumerated values. Each action defines how the same parsed abstract syntax tree is interpreted and what side effects occur on disk.

   2.2. **Check** reads files but does not write them. It succeeds only if every visited function item that falls under the tool’s rules already satisfies the instrumentation predicate. The first violation short-circuits further descent into that function’s body in the visitor (the implementation records the offending span and does not recurse into nested items inside that function). The process prints a single human-readable line to standard output naming the location (path and 1-based line and column from the span) and exits with a dedicated non-zero status reserved for “check failed,” distinct from generic I/O or parse failures.

   2.3. **Fix** rewrites each selected file in place: open for read, parse, transform, then truncate-and-write through the same path. It inserts a new attribute line immediately above each function that needs instrumentation, using indentation copied from the function’s own span so inserted lines align with the surrounding code.

   2.4. **Strip** also rewrites in place. It removes contiguous runs of source lines that correspond to the span of any attribute the tool classifies as an instrumentation attribute, for each matching function item. Removal is line-oriented: the span’s start and end lines (converted to a zero-based index space) are deleted from a line buffer built by splitting the file on newline characters.

   2.5. Optional **path** roots the walk at a directory or file; when omitted, the current working directory is used. **Exclude** accepts a comma-separated list of substrings; any filesystem path whose string representation contains any listed substring is skipped entirely—useful for vendored code, generated sources, or specific modules without listing every file.

   2.6. **Suffix** controls which procedural macro path is emitted on **fix**. The default prefix resolves to a crate-qualified `instrument` macro under a logging shim namespace; an empty suffix produces a bare `instrument` attribute (appropriate when the macro is imported into scope). Arbitrary dotted path prefixes can be supplied for projects that wrap or re-export instrumentation under a custom module tree. The help text warns that strip semantics are tied to how faithfully future source matches what fix emitted; exotic suffixes may not round-trip with strip if detection logic no longer recognizes the attribute shape.

   2.7. **cfg_attr** optionally wraps the inserted attribute in a conditional compilation form: the outer attribute carries the user-supplied predicate, and the inner macro invocation is the second argument. That allows instrumented builds to compile tracing only when a feature flag is enabled, for example. When this flag is absent, fix emits a plain attribute whose path is the suffix plus `instrument`.

   2.8. Exit status is tri-valued beyond the usual success bit: success, hard error (I/O, UTF-8, parse failures), and check-failed are distinguishable so CI scripts can gate merges on missing instrumentation without conflating environmental failures with policy violations.

3. File selection and traversal

   3.1. Directory walking follows symbolic links. Every entry is considered; only regular files ending in the Rust extension pass the filter. Files named for the build-script convention used by Cargo are always skipped so build logic is not mass-edited.

   3.2. Exclusion is purely lexical: a substring match against the full path string. There is no glob engine and no awareness of module structure—only whether the path contains a forbidden needle.

4. Core pipeline (per selected file)

   4.1. Raw bytes are read fully into memory, decoded as UTF-8 (failure is a hard error), then parsed as a whole crate file with a full-syntax parser that preserves enough structure to visit items and spans.

   4.2. The active action installs a different tree visitor. All three visitors share the same high-level strategy: only two kinds of function items are explicitly handled—free functions and methods inside inherent impl blocks. Other callable shapes (closures, trait impl methods not distinguished here in the same way, etc.) are out of scope for the visitor hooks shown; nested functions are discovered only when the visitor recurses into blocks of functions that passed the “already instrumented or exempt” gate.

   4.3. For **fix**, the tool does not pretty-print the AST back to source. Instead it keeps a parallel line-based representation: each original line is stored paired with an optional “insert before this line” buffer. Visiting a function that needs instrumentation computes the attribute text (including indentation), and assigns that text to the slot immediately before the function’s opening line. Reassembly concatenates the first segment, optional newline, then for each subsequent line emits the optional prefix line, newline if needed, then the original line—interleaved so insertions appear in source order without shifting line numbers inside the visitor beyond the modeled “before line N” indirection.

   4.4. For **strip**, the visitor starts from a map from line index to the text of that line (every line initially present). When an instrumentation attribute is found on a function, the tool computes the 1-based start line and end line from the attribute’s span, converts to a half-open range in zero-based indices, and removes those keys from the map. Nested blocks are still visited so nested functions are processed. The map is then turned back into a string by sorting remaining lines by index and joining with newlines—so relative order of non-removed lines is preserved; completely removed lines disappear.

   4.5. For **check**, state is a single optional span carried in the visitor. When a function is encountered that is not `const`, is not marked as a test (or proof) function, and lacks a recognized instrumentation attribute, that function’s span replaces the state. The visitor does not descend into the block in that case, so inner items are not diagnosed separately once an outer layer is already noncompliant. When the function is exempt or already instrumented, the visitor recurses into the block to find deeper functions.

5. Predicate: what counts as “instrumented” or “test”

   5.1. **Instrumented** is true if any attribute on the item matches one of: a path-shaped attribute whose last segment is literally `instrument`; a list-shaped attribute whose path ends with `instrument`; or a `cfg_attr` list whose token stream contains an identifier `instrument` anywhere in the payload (a coarse token scan, not a full nested parse of cfg_attr’s grammar). This last case is what allows `cfg_attr(feature = "…", tracing::instrument(...))` to satisfy check without the tool knowing the exact macro path inside the cfg.

   5.2. **Test exemption** treats unit tests and Kani proof harnesses similarly: path attributes ending in `test` or `proof`, or list attributes whose path ends in `proof`, mark the function as exempt from the instrumentation requirement. Const functions are always skipped—they cannot receive the same attribute pattern in the fix path, and check does not require instrumentation on them.

   5.3. The fix path always emits a minimal attribute: the configured macro suffix with `instrument`, optionally wrapped in `cfg_attr`. It does not copy parameters from an existing attribute (levels, `skip` lists, renames) because those are project-specific; the inserted line is intentionally uniform.

6. Consistency properties and limitations

   6.1. **Newline model**: Splitting and joining on `\n` assumes Unix-style newlines in the working copy; unusual line endings could theoretically desynchronize span lines from the split buffer.

   6.2. **Strip vs. suffix**: Strip identifies attributes by structural rules, not by exact string equality to what fix last wrote. Custom suffixes remain strippable if they still end with the recognizable `instrument` path shape or cfg_attr-with-instrument pattern.

   6.3. **Order of operations in directory walks**: The walk is depth-first as provided by the directory iterator. The first reported missing instrumentation during check corresponds to that traversal order, not necessarily lexical file order or innermost function first.

   6.4. **In-place write**: Fix and strip open the write handle after the read completes successfully; a failure after parsing but before writing leaves the original file untouched, while a failure mid-write could truncate—standard risk for truncate-in-place tooling.

7. Conceptual data flow

   ```
   CLI flags
        |
        v
   Directory walk ----exclude / extension / build.rs filter
        |
        v
   For each .rs: bytes -> UTF-8 -> AST
        |
        +--> CHECK ----> optional span (first violation)
        |
        +--> FIX ------> line buffer + inserted lines -> rewrite file
        |
        +--> STRIP ----> line map minus attribute line ranges -> rewrite file
   ```

8. Integration testing strategy (behavioral contract)

   8.1. Black-box tests invoke the compiled binary against temporary files under a system temp directory with unique names. They assert exit codes, stdout/stderr contents, and final file contents for representative scenarios: default suffix fix, cfg_attr wrapping, strip removing inserted attributes, check reporting line-column locations, exclusion leaving some files untouched, error handling when the path does not exist, and end-to-end cycles of check → fix → check → strip for a small sample crate fragment including nested modules and `#[test]` exemption.

   8.2. Those tests encode the de facto specification for indentation, default macro path, and interaction between flags; the architecture above is consistent with that contract.
