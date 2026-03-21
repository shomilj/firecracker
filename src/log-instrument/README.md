# log-instrument

1. **Purpose and scope**

   1.1. The crate layers optional, very low-level call-structure visibility on top of the standard logging facade. It emits paired events at TRACE severity around the dynamic extent of each attributed callable, so that a downstream logger (for example one configured to show TRACE for the instrumentation target) can reconstruct which functions entered and exited, in what order, and how nesting relates calls across the same OS thread.

   1.2. Nothing here participates in distributed tracing standards, span IDs, or sampling policies from the wider observability ecosystem. The design is deliberately minimal: static labels taken from source, thread identity from the standard library, and a single global mutex protecting shared state keyed by thread. It is suitable for debugging and coarse profiling of synchronous call trees, not for production-grade observability pipelines that require correlation across threads, processes, or async task boundaries without extra care.

2. **Architectural split: compile-time hook vs runtime bookkeeping**

   2.1. Responsibility is divided across two publishable units that are versioned and depended on separately at the manifest level but intended to be consumed as one logical feature. One unit is a procedural-macro library that runs only at compile time; the other is an ordinary library that is linked into the final artifact and executes at runtime.

   2.2. The macro layer’s only job is to recognize attributed functions and rewrite their bodies so that the first executable statement in each body acquires a small guard value whose destructor runs when the function’s scope ends. The runtime library defines that guard type and the global state the destructor consults. The macro crate depends on the runtime crate’s public surface for the constructor it injects; the application-facing crate re-exports the macro so callers typically depend on a single package name and apply one attribute.

   2.3. This split keeps the procedural macro crate free of runtime dependencies beyond what the compiler already pulls for macro expansion, while the runtime crate only needs the logging facade and synchronization primitives. It also makes explicit that instrumentation is an additive transform: if the macro is not applied, no extra code from this feature runs.

3. **Macro expansion contract (what the attribute guarantees at compile time)**

   3.1. The attribute accepts no parameters in its current form. When it is placed on a function item, the macro parser consumes the entire function and rejects any other item kind with a hard failure at expansion time, so misuse is caught early rather than silently ignored.

   3.2. The transformation prepends one statement to the function body’s statement list: a `let` binding that initializes the guard by passing a string literal equal to the function’s identifier as written in source. That string is stored as a `'static` reference inside the runtime’s per-thread stack, which is why the label does not reflect generic parameters, overloads, or module qualification—only the bare name.

   3.3. Because the binding is the first statement, the guard’s destructor runs after the rest of the body completes for every normal or early return path, and Rust’s drop rules also run it during unwinding when a panic crosses the function boundary (unless the process aborts before unwinding). The generated binding name is deliberately unusual to reduce collision with user code; the macro also injects a lint allowance so that valid patterns mixing inner items and statements in the same block still compile under pedantic style checks.

   3.4. The macro does not wrap parameters, alter signatures, or add `async`/`await` at the source level. Async functions remain ordinary function items from the macro’s perspective; the guard’s lifetime is tied to the outer synchronous scope of the state machine the compiler builds, not to individual poll points—see §7.

   3.5. **Contract between expansion and runtime.** The injected constructor must receive a `'static` string slice naming the current activation. The returned value must implement drop semantics that log exactly one matching exit for that activation on the same conceptual stack as the entry, assuming single-threaded drop on the same OS thread as the entry (see §7.4 for when that assumption breaks). If the macro and runtime versions drift—different constructor arity, different drop behavior, renamed types—link errors or inconsistent logs can result even when compilation succeeds for one half of the pair.

4. **Runtime model: global map, mutex, and per-thread stacks**

   4.1. At the center is a lazily initialized global container: a mutex wrapping a map from the current standard-library thread identifier to an ordered vector of static label references. Conceptually each vector is a stack: the innermost attributed call corresponds to the last element. The map is shared across all threads; every push, pop, and prefix computation acquires the same mutex, so instrumentation updates serialize globally even though each logical stack is only semantically owned by one thread under correct use.

   4.2. When the guard’s constructor runs, it locks the mutex, resolves the current thread’s key, and either creates a new vector containing the new label or pushes onto an existing vector. Before pushing, it computes a *prefix string* by folding over the stack *below* the newly pushed entry: each prior label contributes a segment prefixed by a fixed delimiter. That prefix represents the chain of outer attributed calls still active. It then emits a TRACE log line encoding the thread’s debug representation, the prefix (possibly empty at the outermost call), a two-character “into” marker, and the new label.

   4.3. When the destructor runs, it locks again, looks up the same thread’s vector, pops the top label, recomputes the prefix from what remains, and emits a second TRACE line with a matching “out of” marker and the popped label. If the mutex is poisoned from a panicking critical section, the usual lock poison rules apply: the instrumentation path may panic as well rather than silently emit inconsistent logs.

   4.4. The prefix folding uses the same delimiter between ancestors, so nested calls produce log lines where the textual path mirrors the static nesting of attributed functions along that thread, independent of how many non-instrumented frames sit between them in the real call stack.

   4.5. **Runtime stack integrity.** The destructor assumes the thread’s vector exists and is non-empty and that the top element corresponds to this activation. Those assumptions hold when every constructor is paired with a destructor on the same thread and no foreign code manipulates the map. They do not hold if a guard value were moved to another thread and dropped there (push on one thread id, pop on another), or if the runtime API were misused manually alongside attributed code. Normal attributed functions do not expose the guard to user control, so the common case stays consistent.

5. **Log format and how to read it**

   5.1. Enter lines conceptually follow: thread identity (debug-formatted), optional ancestor path built from outer labels with a fixed delimiter between segments, a visual “down” or “into” marker (two consecutive angle brackets in the forward direction), then the label being entered. Exit lines mirror that with an “up” or “out of” marker (two consecutive angle brackets in the backward direction). Together they bracket the dynamic extent of that invocation.

   5.2. Because the logger target and module path come from the logging facade’s configuration, these lines typically appear under the runtime crate’s target unless the application remaps targets. Application code using other macros on other targets will interleave; the useful invariant is ordering within the TRACE stream for the instrumentation target on a single thread.

   5.3. The following ASCII sketch shows two nested attributed calls on one thread; indentation is logical, not part of the real output:

   ```
        Thread A outer enters
            |
            v
        Thread A (outer, inner) enters
            |
            (body work)
            |
            v
        Thread A (outer, inner) exits
            |
            v
        Thread A outer exits
   ```

6. **Threading, async, and correctness boundaries**

   6.1. **Synchronous nesting.** When every attributed function calls the next on the same thread without interleaving unrelated work that also manipulates the same global stacks, the logged prefix accurately reflects the static nesting of those functions.

   6.2. **Cross-thread calls.** A callee running on another thread has its own key in the map. Logs remain correct per thread but no single prefix ties parent on one thread to child on another; correlation requires external clues (for example explicit IDs in other log fields).

   6.3. **Async and cooperative scheduling.** Tasks that `.await` may yield the thread to other tasks. If another task runs attributed code on the same thread before the first task resumes, pushes and pops interleave in the same vector keyed by that thread, corrupting the logical stack relative to any single async flow. The facility assumes synchronous call/return nesting aligned with OS-thread stack behavior; async runtimes that multiplex many logical tasks on few threads can produce misleading paths unless only one such task executes attributed sections at a time on a given thread.

   6.4. **Where the guard lives for async functions.** The compiler lowers an async function to a state-machine type that runs the user body in fragments across scheduling points. The injected guard is still a synchronous `let` at the outer block scope of that body: it is constructed when an invocation of the async function begins (when the future starts running), and dropped when that outer scope ends—typically when the future is dropped or completes, not on every await point. Nested async calls or recursive async patterns still follow the same thread-keyed stack rules; the mismatch is between *logical* async tasks and *physical* thread interleaving, not a special case in the guard’s placement.

   6.5. **Re-entrancy and recursion.** A function that calls itself or is part of a recursive cycle pushes a new copy of the same static label each time. Prefixes distinguish depth by repetition in the path segments, not by distinguishing separate activations with unique IDs.

   6.6. **Panic and unwinding.** If the user body panics, the guard’s destructor still runs during unwinding unless the runtime aborts without unwinding. If instrumentation itself panics (for example on poisoned mutex or violated stack assumptions), that secondary panic can complicate diagnostics; the design favors failing loudly over emitting impossible enter/exit pairs.

7. **Performance characteristics and when to avoid hot paths**

   7.1. **Per-event work.** Every enter performs: one mutex lock, a hash map lookup or insert for the current thread id, push onto a vector of static references, allocation and formatting of a new string for the full prefix (fold over all ancestors), and one TRACE log call. Every exit performs: lock, lookup, pop, another full prefix string allocation from the remaining stack, and another TRACE log call. There is no attempt to reuse prefix buffers across events or to shard locks by thread.

   7.2. **Contention.** A single mutex serializes all threads’ instrumentation. Under high parallel load, threads may queue on that lock even when their logical stacks are independent; contention can dominate if many cores simultaneously enter or exit attributed regions.

   7.3. **Logging backend.** Whether TRACE calls compile away or incur dynamic checks depends on the logging facade configuration and crate features. Even when disabled at runtime, the guard constructor and destructor still execute the mutex and stack manipulation unless the optimizer can prove the entire attributed call away—which is unlikely when the body has side effects.

   7.4. **Algorithmic cost of depth.** Prefix strings grow linearly with nesting depth; each enter and exit walks the entire current stack to build the prefix. Deeply nested attributed call chains therefore pay proportional to depth twice per boundary crossing.

   7.5. **Intended use.** The feature targets diagnostic sessions, bug hunts, and coarse “who called whom” narratives—not sustained always-on tracing at extreme QPS. For production systems, consider whether the mutex, string churn, and TRACE volume are acceptable before leaving instrumentation enabled.

8. **Relationship to the companion procedural-macro package**

   8.1. The application-facing crate exists to own the stable user path: one dependency line, one attribute name, and a re-export that merges macro and runtime capabilities. The macro-only package parses Rust items and rebuilds the function with the extra statement; it does not itself log or maintain state.

   8.2. Version coupling is expressed via path dependencies in the workspace: the macro crate pins to a compatible runtime version so the injected constructor call and the guard type stay in agreement. Consumers should not need to depend on the macro crate directly unless they split their build graph unusually.

   8.3. For a deeper treatment of the compile-time rewrite pipeline, constraints, and failure modes of the attribute itself, see the dedicated document shipped alongside the procedural-macro package. The runtime behavior described here is authoritative for how those rewrites behave once compiled.

9. **Runnable examples**

   9.1. Packaged examples enable TRACE on a simple environment-backed logger and exercise nested and non-nested calls. They are the most direct way to see interleaved diagnostic lines from application code and TRACE bracket lines from this layer on one stream.
