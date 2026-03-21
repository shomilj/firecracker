# clippy-tracing architecture

1. Purpose and problem shape

   1.1. The tool is a batch-oriented source transformer and linter for Rust crates. It walks a directory tree, opens each Rust source file, parses it into a syntax tree, and applies one of three mutually exclusive operations: verify that every eligible function carries a particular instrumentation attribute, insert that attribute where it is missing, or remove matching instrumentation attributes from the source text.

   1.2. The design assumes repositories large enough that hand-editing thousands of function items is impractical. Automation therefore favors predictable, mechanical rules over semantic understanding of call graphs or runtime behavior. The tool does not compile the crate; it only parses and rewrites text according to syntactic patterns.

   1.3. Despite naming that suggests integration with the Clippy linter, the implementation is a standalone binary driven by explicit command-line actions. It does not embed inside the compiler or Clippy; it is an external pass over Rust source files.

2. External interface: actions, flags, and process outcomes

   2.1. The binary accepts a required action flag with exactly one of three enumerated values. Each action defines how the same parsed abstract syntax tree is interpreted and what side effects occur on disk.

   2.2. **Check** reads files but does not write them. It succeeds only if every visited function item that falls under the tool’s rules already satisfies the instrumentation predicate. The first violation short-circuits further descent into that function’s body in the visitor (the implementation records the offending span and does not recurse into nested items inside that function). The process prints a single human-readable line to standard output naming the location (filesystem path and 1-based line and column from the span) and exits with a dedicated non-zero status reserved for “check failed,” distinct from generic I/O or parse failures.

   2.3. **Fix** rewrites each selected file in place: open for read, parse, transform, then truncate-and-write through the same path. It inserts a new attribute line immediately above each function that needs instrumentation, using indentation copied from the function’s own span so inserted lines align with the surrounding code.

   2.4. **Strip** also rewrites in place. It removes contiguous runs of source lines that correspond to the span of any attribute the tool classifies as an instrumentation attribute, for each matching function item. Removal is line-oriented: the span’s start and end lines (converted to a zero-based index space) are deleted from a line buffer built by splitting the file on newline characters.

   2.5. Optional **path** roots the walk at a directory or file; when omitted, the current working directory is used. **Exclude** accepts a comma-separated list of substrings; any filesystem path whose string representation contains any listed substring is skipped entirely—useful for vendored code, generated sources, or specific modules without listing every file.

   2.6. **Suffix** controls which procedural macro path is emitted on **fix**. The default prefix resolves to a crate-qualified `instrument` macro under a logging shim namespace; an empty suffix produces a bare `instrument` attribute (appropriate when the macro is imported into scope). Arbitrary dotted path prefixes can be supplied for projects that wrap or re-export instrumentation under a custom module tree. The help text warns that strip semantics are tied to how faithfully future source matches what fix emitted; exotic suffixes may not round-trip with strip if detection logic no longer recognizes the attribute shape.

   2.7. **cfg_attr** optionally wraps the inserted attribute in a conditional compilation form: the outer attribute carries the user-supplied predicate, and the inner macro invocation is the second argument. That allows instrumented builds to compile tracing only when a feature flag is enabled, for example. When this flag is absent, fix emits a plain attribute whose path is the suffix plus `instrument`. Note that this flag only affects **fix**; **check** and **strip** infer satisfaction from the parsed attributes already present in the file and do not re-emit anything using the predicate.

   2.8. Exit status is tri-valued beyond the usual success bit: success, hard error (I/O, UTF-8, parse failures), and check-failed are distinguishable so CI scripts can gate merges on missing instrumentation without conflating environmental failures with policy violations.

3. Exit codes and I/O contract

   3.1. The process reports completion through a small numeric exit convention wired into the platform’s normal process termination channel. Three outcomes are distinguished:

   | Code | Meaning |
   |------|---------|
   | 0 | Success: walk completed; for check, no missing instrumentation was found. Standard output and standard error are typically empty for fix and strip; check emits nothing when successful. |
   | 1 | Hard error: unreadable paths, UTF-8 decode failure, syntax parse failure, or failure to open or write a target file. A one-line message prefixed for human readability is written to standard error. |
   | 2 | Check failed: at least one eligible function lacked required instrumentation. Exactly one line is written to standard output describing the first violation (path plus line and column). Standard error is empty for this outcome. |

   3.2. Ordering of discovery matters for check: the directory walk yields entries in depth-first order; the first failing file in that order is fully processed until a violation is found, then the tool returns immediately without scanning remaining files. Thus “first reported problem” is coupled to walk order, not to a global sort of all functions in the tree.

   3.3. Fix and strip only open the write handle after a successful read and parse; if transformation fails before writing, the original bytes are untouched. If writing fails after truncation, the file may be left partial or empty—an inherent risk of in-place rewrite without atomic replace.

4. How the syntax tree is used: visitors and traversal

   4.1. Parsing uses a full-file parse that preserves spans (line and column) for every node. Those spans anchor all three actions: check records where a violation starts; fix inserts text immediately before the line index derived from a function’s span start; strip deletes the line range covered by an attribute’s span.

   4.2. The generic visitor pattern from the syntax-tree ecosystem walks the crate root and recursively visits nested items (modules, impl blocks, etc.) through default implementations. The tool does not implement a bespoke tree walk from scratch; it overrides only the hooks for two syntactic shapes: free-standing function items and function items that appear inside inherent implementation blocks.

   4.3. **What gets visited.** Anything that parses as one of those two function shapes is subject to the rules. That includes inherent methods and methods declared inside `impl Trait for Type` blocks, because they share the same AST node kind as inherent methods. Shapes that are deliberately out of scope include: closures (they are expressions, not function items), functions generated only inside macro expansions in ways that do not surface as ordinary function items in the parsed tree, and any callable that is not represented as one of the two hooked node types.

   4.4. **Default recursion into modules.** For items nested under `mod` declarations (including inline modules), the default visitor continues into nested items. The tool’s custom logic only activates at function boundaries; everything else is transparently descended.

   4.5. **The three visitor implementations** share a parallel structure but differ in state and control flow:

   ```text
   ┌─────────────────────────────────────────────────────────────────┐
   │                     visit_file (crate root)                      │
   └─────────────────────────────┬───────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
   ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
   │    CHECK     │       │     FIX      │       │    STRIP     │
   │ Optional     │       │ Line buffer  │       │ Line index   │
   │ violation    │       │ + “insert    │       │ map minus    │
   │ span         │       │  before line”│       │ removed spans│
   └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
          │                      │                      │
          │  same entry hooks    │                      │
          └──────────┬───────────┴──────────────────────┘
                     ▼
            function item? ──yes──► apply predicate
                     │
                     └──► (default) recurse into other items
   ```

   4.6. **Check visitor control flow.** When a function item is visited, attributes are classified into “already instrumented,” “exempt as test or proof,” or “neither,” and constness is inspected. If the function is const, it is skipped entirely (no insertion requirement, no descent rationale tied to what fix can emit). If the function is exempt or already instrumented, the visitor recurses into the function body so that nested functions can still be evaluated. If the function is noncompliant, the visitor stores that function’s span as the single violation and **does not** recurse into the block. That means the first missing instrumentation in depth-first order within a file is reported at the outermost noncompliant layer; inner nested functions in the same body are not listed separately in the same run.

   4.7. **Fix visitor control flow.** The same predicate decides whether to insert an attribute line. Insertion always happens **above** the function’s opening line, with leading spaces equal to the start column of the function’s span (so the attribute lines up with the `fn` keyword). Critically, after handling the current function, the visitor **always** recurses into the block, even when an attribute was just added. Nested functions therefore each receive their own attribute when eligible. There is no “short-circuit” analogous to check.

   4.8. **Strip visitor control flow.** For each eligible function, the tool locates at most one attribute considered “the” instrumentation attribute using a linear scan that stops at the first match. If found, every full line touched by that attribute’s span is removed from a map keyed by zero-based line index. The visitor then recurses into the block so nested functions are processed independently. If multiple instrumentation-like attributes were stacked on one function, only the first one discovered by that scan is removed per pass; unusual layouts might require multiple strip runs.

   4.9. **Line-buffer representation for fix.** The implementation avoids pretty-printing the AST back to source. It keeps the original lines in order, paired with an optional “text to emit immediately before this line.” Reassembly walks the structure: optional prefix, newline boundary rules, then the original line, repeating. Insertions therefore interleave in source order without renumbering spans inside the visitor beyond addressing “before line N.”

   4.10. **Line-map representation for strip.** The file is split on newline characters into an ordered map from line index to line text. Deletion removes entire keys for every index in the half-open range from one before the attribute’s start line through the attribute’s end line (span end line is exclusive in index arithmetic as implemented). Remaining lines are sorted by index and joined with newlines. Completely removed lines disappear; relative order of surviving lines is unchanged.

5. Predicate: what counts as “instrumented” or “test”

   5.1. **Instrumented** is true if any attribute on the item matches one of: a path-shaped attribute whose last path segment is literally `instrument`; a list-shaped attribute whose path ends with `instrument`; or a `cfg_attr` list whose token stream contains an identifier `instrument` anywhere in the payload (a coarse token scan, not a full nested parse of cfg_attr’s grammar). This last case is what allows conditional wrappers around a macro path to satisfy check without the tool knowing the exact macro path inside the conditional.

   5.2. **Test exemption** treats unit tests and Kani-style proof harnesses similarly: path attributes ending in `test` or `proof`, or list attributes whose path ends in `proof`, mark the function as exempt from the instrumentation requirement. Const functions are always skipped—they cannot receive the same attribute pattern in the fix path in a useful way, and check does not require instrumentation on them.

   5.3. **Name-value attributes** (key = value forms) are never classified as instrumentation by the matcher; only path and list meta shapes participate.

   5.4. The fix path always emits a minimal attribute: the configured macro suffix with `instrument`, optionally wrapped in `cfg_attr`. It does not copy parameters from an existing attribute (levels, `skip` lists, renames) because those are project-specific; the inserted line is intentionally uniform. The signature is not consulted when generating the macro path—only the suffix and optional conditional wrapper matter.

6. Check versus fix versus strip: behavioral comparison

   6.1. **Side effects.** Check is read-only. Fix and strip rewrite files in place when the path filter includes them.

   6.2. **Recursion into nested functions after handling an outer function.**

   | Action | Outer function noncompliant / unstripped | Inner functions in same body |
   |--------|------------------------------------------|------------------------------|
   | Check | Records outer span; does **not** recurse | Not visited in that branch |
   | Fix | Inserts attribute; **does** recurse | Each eligible inner function also edited |
   | Strip | Removes first matching attribute span; **does** recurse | Each nested function processed |

   6.3. **Output.** Check may print one location line to standard output on failure. Fix and strip are silent on success unless the platform or wrapper adds noise.

   6.4. **Round-tripping.** A typical workflow is: check fails → fix → check passes → strip → source returns to prior shape for attributes the tool understands. Custom suffixes and cfg_attr wrappers remain strippable as long as the recognition rules still match the stored source.

7. File selection and traversal

   7.1. Directory walking follows symbolic links. Every entry is considered; only regular files ending in the Rust extension pass the filter. Files named for the build-script convention used by Cargo are always skipped so build logic is not mass-edited.

   7.2. Exclusion is purely lexical: a substring match against the full path string. There is no glob engine and no awareness of module structure—only whether the path contains a forbidden needle.

8. Edge cases and subtle behaviors

   8.1. **Newline model.** Splitting and joining on `\n` assumes Unix-style newlines in the working copy. Files with `\r\n` line endings may leave carriage returns attached to line text; unusual line endings could theoretically desynchronize span lines from the conceptual line buffer.

   8.2. **Column reporting.** The reported column for a check violation is the start column of the spanning node (typically the opening of the function item). This matches integration expectations for nested functions inside modules.

   8.3. **First attribute wins for strip.** If multiple attributes could match, stripping targets the first in attribute list order, not necessarily the visually topmost line if attributes were reordered unconventionally.

   8.4. **cfg_attr token scan.** Any `instrument` identifier inside a cfg_attr token tree satisfies the instrumented predicate, which is slightly broader than “macro path ends with instrument” and may theoretically match comments or strings only if they appeared as identifier tokens—unlikely in practice for normal Rust surface syntax.

   8.5. **Async, unsafe, and ABI qualifiers.** The visitor keys off function items regardless of asyncness or unsafe headers; those parse as part of the same item shape.

   8.6. **Trait default methods and impls.** Methods in trait implementations are visited the same as inherent methods; default bodies inside trait definitions would be separate item kinds not covered by the same hooks—another scope boundary.

9. Limitations (explicit non-goals)

   9.1. **No compilation or name resolution.** Paths in attributes are not resolved against `use` trees; recognition is purely syntactic on the last segment or token scan.

   9.2. **No closure or local-function coverage beyond nested `fn`.** Closures are never instrumented by this pass.

   9.3. **No duplicate-attribute consolidation.** Fix may add a second instrumentation attribute if the developer hand-wrote a different partial form the predicate does not treat as satisfied—teams should rely on check to catch policy and avoid double attributes manually.

   9.4. **Macro-generated code.** Source that only exists inside a macro expansion may not appear as visitable function items in the parsed file; such generated code is invisible to the tool.

   9.5. **Order of operations in directory walks.** The walk is depth-first as provided by the directory iterator. The first reported missing instrumentation during check corresponds to that traversal order, not necessarily lexical file order or innermost function first.

10. Conceptual data flow

    ```mermaid
    flowchart TD
      A[CLI: action, path, exclude, suffix, cfg_attr] --> B[Filtered directory walk]
      B --> C[Read bytes, UTF-8 decode, parse file to AST]
      C --> D{Action}
      D -->|check| E[Visitor: optional first violation span]
      D -->|fix| F[Visitor: interleaved insertions, then write]
      D -->|strip| G[Visitor: delete line ranges, then write]
      E --> H{Violation before end of walk?}
      H -->|yes| I[Stdout line, exit 2, walk stops]
      H -->|no| J[Exit 0]
      F --> K{Write ok?}
      G --> K
      K -->|yes| J
      K -->|no| L[Stderr, exit 1]
      C -->|I/O, UTF-8, or parse| L
    ```

    ```text
    CLI flags
         |
         v
    Directory walk ----exclude / extension / build-script filter
         |
         v
    For each Rust source: bytes -> UTF-8 -> AST
         |
         +--> CHECK ----> optional span (first violation) ----> early exit from walk
         |
         +--> FIX ------> line buffer + inserted lines -> rewrite file
         |
         +--> STRIP ----> line map minus attribute line ranges -> rewrite file
    ```

11. Integration testing strategy (behavioral contract)

   11.1. Black-box tests invoke the compiled binary against temporary files under a system temp directory with unique names. They assert exit codes, stdout/stderr contents, and final file contents for representative scenarios: default suffix fix, cfg_attr wrapping, strip removing inserted attributes, check reporting line-column locations, exclusion leaving some files untouched, error handling when the path does not exist, and end-to-end cycles of check → fix → check → strip for a small sample crate fragment including nested modules and test exemption.

   11.2. Those tests encode the de facto specification for indentation, default macro path, and interaction between flags; the architecture above is consistent with that contract.
