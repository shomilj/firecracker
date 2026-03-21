# log-instrument

1. **Purpose and scope**

   1.1. The crate layers optional, very low-level call-structure visibility on top of the `log` facade. It emits paired events at TRACE severity around the dynamic extent of each attributed callable, so that a downstream logger (for example one configured to show TRACE for the crate’s target) can reconstruct which functions entered and exited, in what order, and how nesting relates calls across the same OS thread.

   1.2. Nothing here participates in distributed tracing standards, span IDs, or sampling policies from the wider “tracing” ecosystem. The design is deliberately minimal: static labels taken from source, thread identity from the standard library, and a single global mutex protecting a small amount of per-thread state. It is suitable for debugging and coarse profiling of synchronous call trees, not for production-grade observability pipelines that require correlation across threads, processes, or async task boundaries without extra care.

2. **Architectural split: compile-time hook vs runtime bookkeeping**

   2.1. Responsibility is divided across two publishable units that are versioned and depended on separately at the manifest level but intended to be consumed as one logical feature. One unit is a procedural-macro library that runs only at compile time; the other is an ordinary library that is linked into the final artifact and executes at runtime.

   2.2. The macro layer’s only job is to recognize attributed functions and rewrite their bodies so that the first executable statement in each body acquires a small guard value whose destructor runs when the function’s scope ends. The runtime library defines that guard type and the global state the destructor consults. The macro crate depends on the runtime crate’s public path for the constructor it injects; the application-facing crate re-exports the macro so callers typically depend on a single package name and apply one attribute.

   2.3. This split keeps the procedural macro crate free of runtime dependencies beyond what the compiler already pulls for macro expansion (`syn`, `quote`, etc.), while the runtime crate only needs the logging facade and synchronization primitives. It also makes it explicit that instrumentation is an additive transform: if the macro is not applied, no extra code from this feature runs.

3. **Compile-time behavior (what the attribute changes)**

   3.1. The attribute accepts no parameters in its current form. When it is placed on a function item, the macro parser consumes the entire function and rejects any other item kind with a hard failure at expansion time, so misuse is caught early rather than silently ignored.

   3.2. The transformation prepends one statement to the function body’s statement list: a `let` binding that initializes the guard by passing a string literal equal to the function’s identifier. That string is stored as a `'static str` inside the runtime’s per-thread stack, which is why the label does not reflect generic parameters, overloads, or module path—only the bare name.

   3.3. Because the binding is the first statement, the guard’s destructor runs after the rest of the body completes for every normal or early return path, and Rust’s drop rules also run it during unwinding when a panic crosses the function boundary (unless the process aborts first). The guard variable is intentionally named to reduce the chance of colliding with user bindings; the macro also injects a lint allowance related to statement ordering so that valid patterns mixing inner items and statements in the same block still compile under pedantic style checks.

   3.4. The macro does not wrap parameters, alter signatures, or add `async`/`await`. Async functions are still functions; the guard’s lifetime is tied to the outer synchronous scope of the generated state machine glue, not to individual poll points—see §6 on concurrency.

4. **Runtime model: global map, mutex, and per-thread stacks**

   4.1. At the center is a lazily initialized global container holding, for each active thread identifier, an ordered list of the static labels currently “open.” Conceptually this is a stack: the innermost attributed call corresponds to the last element. The container is wrapped in a mutex so that all pushes and pops serialize across threads when updating the map, even though each logical stack is only ever mutated by one thread’s identifier at a time under correct synchronous use.

   4.2. When the guard’s constructor runs, it locks the mutex, looks up or creates the vector for the current thread, pushes the new label, and computes a prefix string by folding over the stack *below* the newly pushed entry—each prior label contributes a segment separated by a fixed delimiter. That prefix represents the chain of outer attributed calls still active. It then emits a TRACE log line encoding the thread id, the prefix (possibly empty on the outermost call), a direction marker meaning “enter,” and the new label.

   4.3. When the destructor runs, it locks again, pops the top label (which must match what was pushed for that activation), recomputes the prefix from what remains in the stack, and emits a second TRACE line with an “exit” marker. If the mutex is poisoned from a panicking critical section, the usual lock poison rules apply and the instrumentation may panic as well—this is consistent with failing loudly rather than logging inconsistent state.

   4.4. The prefix folding uses the same delimiter between ancestors, so nested calls produce log lines where the textual path mirrors the static nesting of attributed functions along that thread, independent of how many non-instrumented frames sit between them in the real stack.

5. **Log format and how to read it**

   5.1. Enter lines conceptually follow: thread identity, optional ancestor path built from outer labels, a visual “down” or “into” marker, then the label being entered. Exit lines mirror that with an “up” or “out of” marker. Together they bracket the dynamic extent of that invocation.

   5.2. Because the logger target and module path come from the `log` crate’s configuration, these lines typically appear under the runtime crate’s target unless the application remaps targets. Application code using `debug!` or `info!` on other targets will interleave; the useful invariant is ordering within the TRACE stream for the instrumentation target on a single thread.

   5.3. The following ASCII sketch shows two nested attributed calls on one thread; indentation is logical, not part of the real output:

   ```
        Thread A outer enters
            |
            v
        Thread A outer::inner enters
            |
            (body work)
            |
            v
        Thread A outer::inner exits
            |
            v
        Thread A outer exits
   ```

6. **Threading, async, and correctness boundaries**

   6.1. **Synchronous nesting.** When every attributed function calls the next on the same thread without interleaving unrelated work that also manipulates the same global stacks, the logged prefix accurately reflects the static nesting of those functions.

   6.2. **Cross-thread calls.** A callee running on another thread has its own thread-local key in the map. Logs remain correct per thread but no single prefix ties parent on thread T1 to child on thread T2; correlation requires external clues (for example explicit IDs in other log fields).

   6.3. **Async and cooperative scheduling.** Tasks that `.await` may yield the thread to other tasks. If another task runs attributed code on the same thread before the first task resumes, pushes and pops interleave in the same vector keyed by that thread, corrupting the logical stack relative to any single async flow. The facility assumes synchronous call/return nesting aligned with thread stack behavior; async runtimes that multiplex many logical tasks on few threads can produce misleading paths unless only one such task executes attributed sections at a time on a given thread.

   6.4. **Re-entrancy and recursion.** A function that calls itself or is part of a recursive cycle pushes a new copy of the same static label each time. Prefixes distinguish depth by repetition in the path segments, not by distinguishing separate activations with unique IDs.

7. **Performance and observability trade-offs**

   7.1. Every enter/exit pair takes a global mutex twice and allocates or updates a small `String` for the prefix on each event. High-frequency hot paths will pay this cost whenever TRACE is enabled for the target; if the logger filters TRACE out at compile time or runtime, the cost may be reduced depending on how the `log` macros expand in the build.

   7.2. The mutex serializes all threads’ instrumentation updates. Under heavy parallel load, contention on that lock could dominate; the feature is aimed at diagnostic sessions rather than always-on production tracing at extreme QPS.

8. **Relationship to the companion procedural-macro package**

   8.1. The application-facing crate exists to own the stable user path: one dependency line, one attribute name, and a re-export that merges macro and runtime symbols. The macro-only package is the place where `syn` parses Rust items and `quote` rebuilds the function with the extra statement; it does not itself log or maintain state.

   8.2. Version coupling is expressed via path dependencies in the workspace: the macro crate pins to a compatible runtime version so the injected constructor call and the guard type stay in agreement. Consumers should not need to depend on the macro crate directly unless they split their build graph unusually.

   8.3. For a deeper treatment of the compile-time rewrite pipeline, constraints, and failure modes of the attribute itself, see the dedicated document shipped alongside the macro crate. The runtime behavior above is authoritative for how those rewrites behave once compiled.

9. **Runnable examples**

   9.1. Packaged examples enable TRACE on a simple env-backed logger and exercise nested and non-nested calls. They are the most direct way to see interleaved `debug!` lines from application code and TRACE bracket lines from this layer on one stream.
