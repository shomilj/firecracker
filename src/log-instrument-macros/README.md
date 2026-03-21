# log-instrument-macros

1. **Role in the overall design**

   1.1. This package implements the compile-time half of optional call instrumentation: an attribute procedural macro that rewrites attributed function bodies so they cooperate with a small runtime guard type defined in the sibling runtime crate. The macro crate does not perform logging, allocate thread-local storage, or execute at application run time beyond the compiler’s expansion phase.

   1.2. The split exists so that crates which only need to *emit* the macro (the compiler’s procedural-macro harness) depend on a minimal graph—parser and quasi-quoting crates—while the final binary links the runtime instrumentation logic and the logging facade. End users normally depend on the umbrella crate that re-exports this macro under a single name; understanding this package is relevant for maintainers extending the attribute or diagnosing expansion failures.

2. **Macro expansion contract**

   2.1. **Inputs.** Expansion receives two token streams: the attribute’s own token stream (currently ignored entirely) and the item token stream that follows the attribute. The second stream must parse as a complete Rust item. The implementation uses a full function abstract-syntax-tree parse so that nested modules, generics, where-clauses, and bodies are represented faithfully.

   2.2. **Precondition: function items only.** If the item is not a function item, expansion stops with an explicit panic carrying a narrow message. Procedural macros in this path do not emit structured diagnostics; misuse surfaces as a compile-time error whose span is tied to the macro invocation rather than a refined secondary label. If the item is severely malformed, failure may instead occur at parse time with the compiler’s ordinary parse diagnostics.

   2.3. **Transformation (single guaranteed effect).** For a function item, the body is a block whose statements are held in an ordered vector. The transform clones or mutates that function item in place: it appends a lint-related attribute requested for pedantic style (so that certain valid block shapes—where inner items and statements coexist—remain acceptable after insertion), then inserts a new first statement into the statement vector.

   2.4. **The inserted statement.** That statement is a `let` binding initializing the guard by calling the runtime constructor exposed through the umbrella crate, passing a string literal whose contents equal the function’s identifier as written in source (not the fully qualified path, not mangled linkage names). The string literal type is `'static`, which the runtime stores by reference in per-thread stacks without per-push allocation for the label text beyond whatever the logging path does.

   2.5. **What is preserved.** No other statements, signatures, or user-authored attributes on the function are removed. Visibility, `async`, `unsafe`, constness, and ABI remain as authored. The macro does not wrap the body in a new block except for the single prepended statement inside the existing block.

   2.6. **Outputs.** The expanded function is turned back into tokens with quasi-quoting and handed to the compiler. The observable contract is: one additional statement at the front of the body, one additional lint attribute on the function item, and otherwise identical user code.

3. **Semantic contract with the runtime guard**

   3.1. The only runtime surface the macro must stay aligned with is the constructor’s contract: it must accept a `'static` string slice and return a value whose destructor performs the paired “exit” logging with the same label the constructor recorded on entry. If those drift between versions—renames, extra parameters, different drop behavior—expansion might still succeed in isolation while link errors or inconsistent logs appear at integration time.

   3.2. The binding must live for the entire function body including control-flow exits. Placing it as the first statement maximizes the chance that any `return`, `?` propagation, or tail expression still runs inside the guard’s scope. Panic unwinding also triggers the destructor unless abort-on-panic is configured at a level that skips unwinding.

   3.3. The macro does not insert stack unwinding adapters or error swallowing; failures inside the user body behave as without instrumentation.

   3.4. **Drop thread vs entry thread.** The runtime pairs pushes and pops by OS thread identifier. The macro does not prevent the returned guard type from being moved according to Rust’s auto-trait rules; if a future maintainer made the guard movable across threads and a user dropped it on a different thread than where it was created, the runtime stack would desynchronize. The intended use keeps the guard as a local binding that drops on the same thread as entry; documenting that invariant is a cross-crate responsibility shared with the runtime package.

4. **Runtime stack (behavior implied by the rewrite, not implemented here)**

   4.1. Although this crate does not implement the global map or mutex, every expansion assumes a runtime that maintains, per active OS thread, a stack of static labels mirroring nested attributed calls on that thread. The first statement’s constructor pushes; the destructor at scope exit pops. Log lines emitted by that runtime are the only user-visible “stack trace” this feature provides—there is no walk of actual machine stack frames.

   4.2. The string literal passed into the constructor is the symbol the runtime stores. Multiple activations of the same function push multiple equal strings; the runtime distinguishes depth only by position in the vector, not by unique span identifiers.

5. **What the attribute does not do**

   5.1. It does not instrument closures as distinct items, individual blocks, or foreign functions unless those appear as ordinary Rust function items with the attribute attached at the definition site.

   5.2. It does not add parameters, derive trait implementations, or wrap the body in a new async block. For `async fn`, the guard wraps the desugared body the compiler builds from the written body—poll scheduling semantics remain the runtime’s concern, and the same async limitations described in the runtime documentation apply.

   5.3. It does not accept configuration: no level selection, no target override, no feature flags in the attribute list. All such policy belongs in logging filters and crate features on the runtime side.

6. **Async and threading caveats (compile-time perspective)**

   6.1. The macro cannot see await points or task identities; it only wraps the outer function body. Cooperative scheduling that interleaves multiple logical tasks on one OS thread can reorder pushes and pops relative to any single async control flow. That is not fixable by a smarter rewrite without integrating with a specific async runtime—out of scope for this attribute.

   6.2. The attribute does not mark functions as thread-local or send boundaries; cross-thread calls simply produce separate stacks keyed by thread id in the runtime. No compile-time warning guards against misleading combined logs across threads.

7. **Performance (compile-time and emitted code)**

   7.1. **Expansion cost.** Each attributed function pays macro expansion work proportional to parsing and quoting the full function item. Very large function bodies increase compile time for that crate incrementally; there is no incremental caching beyond the compiler’s own query system.

   7.2. **Emitted code size and runtime.** Each expansion injects one constructor call and one drop hook at the LLVM/Rust MIR level for the guard. Beyond that, runtime cost is entirely in the sibling library (mutex, map, string formatting, logging). The macro does not generate per-line probes or multiple exit paths—only the single scope guard.

   7.3. **Inlining.** Whether the guard’s constructor and destructor inline into the user function depends on optimization settings and the runtime’s definitions; the macro does not force either way.

8. **Failure modes and diagnostics**

   8.1. Wrong item kind produces a panic during macro expansion—developers see a message that the attribute applies only to functions. Attaching to structs, enums, impl blocks, or use items triggers this path.

   8.2. If the parser cannot parse the item (severely malformed input inside the function), expansion fails at parse time with the compiler’s usual parse errors, not the custom panic.

   8.3. Name collisions between the generated binding and user code are mitigated by choosing a deliberately obscure binding name; extremely unusual code that macro-expands to the same identifier in the same scope could still break and would need a manual rename of user variables—an accepted limitation for minimal injection.

9. **Maintenance and extension considerations**

   9.1. Adding optional parameters to the attribute would require parsing the attribute token stream (currently discarded), validating literals or paths, and threading them into the generated statement—possibly as extra arguments to the runtime constructor or as const generic parameters if the runtime API evolves.

   9.2. Supporting additional item kinds would relax the function-only match and duplicate parts of the insertion logic for each shape, or factor a shared “prepend statement to body” helper.

   9.3. Keeping the parser feature set (full syntax coverage and extra traits for debugging) aligned with language editions used by the workspace avoids subtle parse gaps on newer Rust constructs.
