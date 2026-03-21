# log-instrument-macros

1. **Role in the overall design**

   1.1. This package implements the compile-time half of optional call instrumentation: an attribute procedural macro that rewrites attributed function bodies so they cooperate with a small runtime guard type defined in the sibling runtime crate. The macro crate does not perform logging, allocate thread-local storage, or execute at application run time beyond the compiler’s expansion phase.

   1.2. The split exists so that crates which only need to *emit* the macro (the compiler’s proc-macro harness) depend on a minimal graph (`syn`, `quote`, `proc-macro2`) while the final binary links the runtime instrumentation logic and the `log` facade. End users normally depend on the umbrella crate that re-exports this macro under a single name; understanding this package is relevant for maintainers extending the attribute or diagnosing expansion failures.

2. **Pipeline from tokens to rewritten function**

   2.1. Expansion begins with the attribute token stream (currently ignored) and the item token stream that follows. The item must parse as a full Rust item; the implementation uses a complete function AST parse so that nested modules, generics, where-clauses, and bodies are all represented.

   2.2. If the item is not a function item, expansion stops with an explicit panic carrying a narrow message. Procedural macros cannot return a structured diagnostic in this path without additional machinery, so misuse surfaces as a compile-time error whose span is tied to the macro invocation rather than a refined secondary label.

   2.3. For a function item, the body is a block whose statements are held in an ordered vector. The transform clones or mutates that function item in place: it appends a lint-related attribute requested for pedantic style (so that certain valid block shapes—where inner items and statements coexist—remain acceptable after insertion), then inserts a new first statement into the statement vector.

   2.4. That statement is a `let` binding initializing the guard by calling the runtime constructor with a string literal whose contents equal the function’s identifier as written in source (not the fully qualified path, not mangled symbols). The string literal type becomes `'static`, which the runtime stores in per-thread stacks without allocation per push beyond whatever the logging path does.

   2.5. The expanded function is turned back into tokens with quasi-quoting and handed to the compiler. No other statements, signatures, or attributes on the function are removed; visibility, `async`, `unsafe`, constness, and ABI remain as authored.

3. **Semantic contract with the runtime guard**

   3.1. The only runtime surface the macro must stay aligned with is the constructor’s signature: it must accept a `'static str` and return a value whose destructor performs the paired “exit” logging with the same label the constructor recorded on entry. If those drift between versions—renames, extra parameters, different drop behavior—expansion would still succeed but logs could become inconsistent or fail at link time.

   3.2. The binding must live for the entire function body including control-flow exits. Placing it as the first statement maximizes the chance that any `return`, `?` propagation, or tail expression still runs inside the guard’s scope. Panic unwinding also triggers the destructor unless abort-on-panic is enabled at a level that skips unwinding.

   3.3. The macro does not insert `catch_unwind` or swallow errors; failures inside the user body behave as without instrumentation.

4. **What the attribute does not do**

   4.1. It does not instrument closures, individual blocks, trait methods separately from their impl items, or foreign functions unless those appear as ordinary Rust `fn` items with the attribute attached at definition site.

   4.2. It does not add parameters, derive trait impls, or wrap the body in a new async block; for `async fn`, the guard wraps the desugared body the compiler builds from the written body—poll scheduling semantics remain the runtime’s concern, and the same async limitations described in the runtime documentation apply.

   4.3. It does not accept configuration: no level selection, no target override, no feature flags in the attribute list. All such policy belongs in `log` filters and crate features on the runtime side.

5. **Failure modes and diagnostics**

   5.1. Wrong item kind produces a panic during macro expansion—developers see a message that the attribute applies only to functions. Attaching to structs, enums, impl blocks, or use items triggers this path.

   5.2. If `syn` cannot parse the item (severely malformed input inside the function), expansion fails at parse time with the compiler’s usual parse errors, not the custom panic.

   5.3. Name collisions between the generated binding and user code are mitigated by choosing a deliberately obscure binding name; extremely unusual code that macro-expands to the same identifier in the same scope could still break and would need a manual rename of user variables—an accepted limitation for minimal injection.

6. **Maintenance and extension considerations**

   6.1. Adding optional parameters to the attribute would require parsing the attribute token stream (currently discarded), validating literals or paths, and threading them into the generated statement—possibly as extra arguments to the runtime constructor or as const generic parameters if the runtime API evolves.

   6.2. Supporting additional item kinds (for example inherent methods are already functions; associated trait defaults are trickier) would relax the `Item::Fn` match and duplicate parts of the insertion logic for each shape, or factor a shared “prepend statement to body” helper.

   6.3. Keeping the macro’s `syn` feature set (`full`, `extra-traits`) aligned with language editions used by the workspace avoids subtle parse gaps on newer Rust constructs.
