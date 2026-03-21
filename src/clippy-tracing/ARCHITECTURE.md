# Observability Instrumentation Subsystem

This document describes a cross-cutting subsystem formed by three cooperating pieces: a standalone command-line tool that scans and edits Rust source trees, a small runtime library that implements lightweight enter-and-exit tracing, and a procedural macro crate that attaches that behavior to individual functions at compile time. Together they provide automated guardrails for consistent observability hooks and a developer workflow that can bulk-apply, verify, or remove those hooks across large codebases without hand-editing thousands of function signatures.

---

## Purpose & Boundaries

**Responsibility.** The subsystem’s core responsibility is to make function-level observability instrumentation predictable and enforceable. The macro layer rewrites attributed functions so that every entry and exit emits structured trace-level diagnostics tied to the function’s identity and to nested calls on the same thread. The runtime layer maintains a per-thread stack of active call names so that nested invocations print a hierarchical path rather than a flat sequence of unrelated labels. The bulk tool closes the loop for engineering process: it can report the first location in a tree where expected instrumentation is absent, inject the standard attribute in front of eligible functions, or strip instrumentation lines that match its recognition rules—optionally skipping paths that should remain untouched and optionally emitting attributes guarded by compile-time configuration predicates.

**Out of scope.** This subsystem does not replace a full distributed tracing implementation, OpenTelemetry exporters, or span correlation across processes. It does not instrument async runtimes, closures, or every syntactic form of callable code automatically; it focuses on ordinary free functions and methods in implementation blocks that the abstract-syntax visitor walks. It does not interpret business metrics, enforce log retention policies, or configure downstream log aggregation. The bulk tool operates at the textual and syntactic level on Rust sources; it is not a semantic analyzer that proves behavioral equivalence before and after edits. Const functions and functions already marked as tests or formal proofs are deliberately excluded from mandatory instrumentation in the checker and fixer, reflecting the assumption that those categories either cannot support the runtime pattern or are validated under different conventions.

**Impact if unavailable.** If the macro and runtime library were removed, attributed functions would lose automatic enter-exit tracing, and operators would rely solely on ad hoc logging inside bodies—often inconsistently applied and harder to correlate across call depth. If the bulk tool failed in continuous integration, teams could still compile and run, but policy that “every non-test function must carry the attribute” would erode: new code would slip through without review catching it uniformly, and large migrations would return to error-prone manual editing.

---

## Interfaces & Contracts

**What the macro exposes.** The procedural attribute accepts only function items. Applying it to any other syntactic item is treated as a programmer error at macro expansion time. For a valid function, the macro inserts a small statement at the beginning of the body and adds a lint-allow attribute so that the injected statement ordering satisfies style rules in strict lint configurations. The expansion depends on the runtime library’s helper type, which is constructed with a static string derived from the function’s name and dropped at scope exit.

**What the runtime library exposes.** Consumers obtain the attribute macro through the library’s public surface; the library re-exports the macro so that dependents need only one direct dependency for both compile-time rewriting and runtime support. The runtime exposes a guard type whose constructor records an entry trace and whose destructor records the matching exit trace. Internally it coordinates with a lazily initialized, mutex-protected map from thread identifiers to vectors of active function-name fragments, building a prefix string that reflects the current stack depth for each thread.

**What the bulk tool exposes.** Operators invoke the tool with a required action mode: verify presence, apply missing attributes, or remove recognized instrumentation. Optional parameters control the root path (defaulting to the current directory), a comma-separated list of substrings that cause any matching file path to be skipped, an optional suffix prepended to the instrument token so generated attributes can target a namespaced macro path, and an optional predicate string used to wrap the generated attribute in a conditional compilation attribute so instrumentation can be toggled by features or similar compile-time configuration hooks. The tool walks the filesystem, follows symbolic links, ignores build scripts by filename convention, and considers only Rust source files.

**Invariants and guarantees.** Callers of the macro must ensure the logging facade is initialized if they expect output; the subsystem only emits at the most verbose level of the standard Rust logging facade’s leveled macros. The bulk tool guarantees that check stops at the first missing site it encounters while traversing files in walk order, which is deterministic for a given tree but not necessarily sorted by human intuition. The fix and strip modes rewrite whole files when they parse successfully; malformed sources surface parse errors rather than partial application. The strip mode removes contiguous source lines covered by the span of a recognized instrumentation attribute, so multi-line attributes are removed as a block.

---

## Data Flow

**Inputs.** Source text enters the bulk tool as UTF-8 file contents. The macro receives token streams representing the attributed item. The runtime receives static string slices for function names at the point of guard construction.

**Transformations.** The bulk tool parses each file into a full crate abstract syntax tree, then traverses function items in nested modules and impl blocks. For verification, it classifies each function’s attributes to decide whether it is already considered instrumented, is a test or proof-like entry, or is a const function—only the remaining functions are flagged. For automated insertion, it computes the line immediately above the function and injects a new attribute line with indentation aligned to the function’s start column, optionally wrapping the attribute in a conditional compilation form. For removal, it maps attribute spans to line ranges and deletes those lines from a line-indexed representation before serializing back to text.

The macro parses the function item, injects one statement at the head of the block, and emits the transformed function. At runtime, the guard’s constructor pushes the current name onto the per-thread stack, formats a prefix from the stack below the new top, and emits a trace record including thread identity, prefix, direction marker, and name. The destructor pops the stack and emits the symmetric exit record.

**Outputs.** The bulk tool prints a human-readable location when verification fails, writes transformed sources on success for mutating modes, and sets process exit codes to distinguish success, verification failure, and general errors. The runtime sends trace records through the logging facade to whatever backend the application configured.

The following diagram summarizes how source policy, compile-time rewriting, and runtime logging connect.

```
  +------------------+     reads/writes      +-------------------------+
  |  source tree     | <-------------------> |  bulk verification    |
  |  (Rust sources)  |                       |  and rewrite tool     |
  +------------------+                       +-----------+-------------+
        |                                                |
        | parse / print                                  | policy:
        v                                                | check | fix | strip
  +------------------+                                   |
  |  syntax-tree     |                                   |
  |  visitors        |                                   |
  |  (functions only)|                                   |
  +--------+---------+                                 |
           |                                            |
           |  fix emits attributes                      |
           v                                            v
  +------------------+     expands at compile    +----------------------+
  |  attributed      | -----------------------> |  procedural attribute |
  |  function items  |                           |  (function-only)      |
  +--------+---------+                           +----------+-----------+
           |                                                |
           |  generated body prefix                         |
           v                                                v
  +------------------+                           +----------------------+
  |  compiled binary | -----------------------> |  guard + thread stack |
  |  (calls)         |   trace at runtime      |  (trace-level records) |
  +------------------+                           +----------+-----------+
                                                             |
                                                             v
                                                  +----------------------+
                                                  |  logging backend     |
                                                  |  (app-configured)    |
                                                  +----------------------+
```

Before the diagram, the key idea is that policy enforcement happens early in the pipeline on raw source, while behavioral instrumentation is realized only after macro expansion and linking. After the diagram, note that the logging backend is shared with the rest of the application: this subsystem does not own log routing, filtering, or formatting beyond choosing the verbosity level for its own records.

---

## Control Flow

**Triggers.** Developers and automation invoke the bulk tool explicitly. The compiler invokes the procedural macro when it processes an attributed function. The runtime guard runs when the generated statement executes: construction runs at function entry, and destruction runs when the function body completes or unwinds.

**Critical path for verification.** The tool parses each candidate file. If parsing fails, the per-file processing step returns an error. If parsing succeeds, a visitor walks the tree. The first eligible function lacking acceptable attributes causes the visitor to capture its source span and short-circuit further deep visitation of that subtree according to visitor rules—tests still recurse into blocks where appropriate, but the check visitor stops descending into a function body once it has flagged the function itself. The main driver returns the first reported location and a distinct exit code so continuous integration can fail fast.

**Critical path for fix.** The visitor inserts text before specific lines in a segmented line buffer, then serializes. Write failures propagate as errors.

**Critical path for strip.** The visitor removes line keys associated with instrument spans, then serializes.

**Branching inside the macro.** Only functions proceed; all other items panic during expansion, which is intentional and loud.

The control-flow sketch below highlights decision points in the bulk tool’s verification mode.

```
                    start
                      |
                      v
              +---------------+
              | walk tree     |
              +-------+-------+
                      |
                      v
              +---------------+
              | open source   |---- error ----> report I/O
              | file          |
              +-------+-------+
                      |
                      v
              +---------------+
              | parse to      |---- error ----> report parse
              | syntax tree   |
              +-------+-------+
                      |
                      v
              +---------------+
              | visit         |
              | functions     |
              +-------+-------+
                      |
          +-------------+-------------+
          |                             |
          v                             v
   +-------------+              +-------------+
   | eligible &  |              | has attr or |
   | missing?    |              | excluded    |
   +------+------+              | category?   |
          | yes                 +------+------+
          v                             |
   +-------------+                      v
   | record span |              recurse into
   | and return  |              nested blocks
   +-------------+
```

Before this figure, the important branching is eligibility: const functions and test or proof-like functions are not required to carry instrumentation. After the figure, note that recursion into nested blocks allows the visitor to find inner functions that are themselves subject to the same rules.

---

## State & Lifecycle

**Bulk tool.** The tool is a short-lived process with no persistent internal state across invocations. Per run, it holds parsed command-line configuration and, while processing each file, transient buffers for file bytes and the segmented line representation. No server lifecycle applies.

**Macro expansion.** Expansion is stateless aside from compiler-internal caches; each attributed function is rewritten independently.

**Runtime library.** A process-global, lazily initialized mutex-backed map holds, for each active thread identifier, a vector of static name strings representing the current call stack of instrumented functions on that thread. The first time a thread enters an instrumented function, the map acquires an entry; nested calls append to the vector; returns pop. The guard type’s drop runs on both normal return and unwinding, so the stack is cleaned up as scopes end. There is no explicit shutdown hook; the map persists for the process lifetime. Threads that never call instrumented code incur no entries.

**State machine view of one thread’s stack.** The stack depth toggles with entry and exit; reentrancy is represented by repeated names if the same function is recursive.

```
        empty
           |
   entry to A
           |
           v
        [ A ]
           |
   entry to B
           |
           v
      [ A, B ]
           |
    exit from B
           |
           v
        [ A ]
           |
    exit from A
           |
           v
        empty
```

Before this diagram, each thread’s vector behaves like a lightweight shadow stack mirroring dynamic call depth for instrumented functions only. After the diagram, remember that non-instrumented callees do not push frames in this structure, so the path prefix reflects only attributed functions, not every frame on the real call stack.

---

## Failure Modes

**Parse and I/O errors in the bulk tool.** Unreadable paths, non-UTF-8 sources, or syntactically invalid Rust produce errors surfaced to standard error with a non-zero exit code. These are explicit failures; the tool does not silently skip broken files except when paths match exclusion rules by design.

**Verification failures.** Missing instrumentation is not an internal error; it is a policy outcome reported with a dedicated exit code and a printed location. Automation should treat that code as a failed gate.

**Macro misuse.** Applying the attribute to non-functions panics at compile time. This is a developer-facing mistake, not a recoverable runtime condition.

**Runtime panics.** The guard’s destructor assumes the per-thread stack is non-empty and that the popped name matches the constructor’s push. Normal instrumented call graphs preserve this invariant. Async code or manual misuse that could break pairing is outside the happy path this design optimizes for; such violations would manifest as panics in the destructor path, which is a sharp edge relative to more defensive designs.

**Strip limitations.** If instrumentation was hand-edited into forms the attribute matcher does not recognize, strip might not remove it; conversely, strip removes by span lines and could be dangerous if combined with unusual formatting—though typical attribute shapes are contiguous lines.

**Silent corruption risks.** There is little silent data corruption: worst cases are missed policy detection if someone evades the simple attribute patterns the tool recognizes, or unintended removal if spans overlap oddly—both are unlikely under normal Rust formatting.

---

## Operational Characteristics

**CPU and memory.** The bulk tool is linear in total source size for each run, with additional overhead for parsing and visiting. Deep directory trees pay walk costs. The runtime mutex serializes access to the shared map across threads; highly parallel workloads with heavy use of instrumentation could contend on that lock, making it a potential scalability bottleneck compared with lock-free or thread-local designs.

**Disk.** Fix and strip modes rewrite files in place via truncate-and-write after reading fully into memory.

**Observability of the subsystem itself.** The runtime emits trace-level records through the standard logging facade, distinct from application logs unless targets align. The bulk tool’s own behavior is visible through its exit code and minimal standard streams output; it does not emit structured metrics.

**Scaling.** The approach scales organizationally to large repositories through exclusion lists and conditionally compiled attributes, letting teams stage rollouts behind feature flags or skip generated or third-party vendored subtrees.

---

## Design Rationale

**Why combine a bulk tool with a custom macro.** Ecosystem-standard instrumentation attributes integrate deeply with the tracing ecosystem, which is powerful but may be heavier than teams want for every function in every crate. A project-specific attribute backed by the ubiquitous logging facade offers a lighter, more predictable trace line for call graphs without mandating a particular telemetry stack. The bulk tool exists because manual adoption at scale is impractical and because consistency is a policy problem, not only a library problem. Checking and fixing at the source level keeps enforcement independent of whether the project currently compiles with every feature combination.

**Why a per-thread stack in the runtime.** Nested calls are common; printing only the innermost function name loses context. A shadow stack of attributed names yields a compact textual hierarchy in log lines, which aids grep-based debugging when distributed tracing is unavailable or overkill.

**Why conditional wrapping is optional.** Some builds omit observability entirely for size or performance; generating conditional attributes avoids either compiling the hooks or requiring separate source forks.

**Tradeoffs.** The bulk tool is syntax-driven and may lag novel attribute spellings; the runtime trades simplicity and global mutex contention for a small implementation; the macro’s panic-on-misuse favors failing fast over graceful degradation. Together these choices prioritize enforceable conventions and low integration cost over maximal flexibility.

This subsystem should be understood as a developer guardrail and a minimal observability amplifier: it nudges code toward uniform hooks, prints cheap hierarchical traces for local diagnosis, and keeps large-scale adoption mechanically feasible.
