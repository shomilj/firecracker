# Shared Workspace Utilities — Architecture

This document describes the architecture of the small library that centralizes cross-cutting helpers used by multiple crates in the workspace. The library is intentionally narrow in scope: it offers a bespoke command-line parsing surface, a thin abstraction over Linux clocks and timestamps, and a single shared validation rule for textual instance identifiers. Nothing here implements networking, storage, virtualization mechanics, or process lifecycle management; those concerns live in higher-level components that *call into* these utilities rather than the reverse.

---

## Purpose & Boundaries

The system exists so that several binaries and libraries can agree on how configuration is expressed at the process boundary, how certain human-readable strings are constrained before use, and how wall-clock and CPU-related measurements are taken without each crate reimplementing the same logic. Its responsibility is to provide predictable, testable building blocks with explicit failure modes rather than to encode product-specific policy beyond the few invariants that are truly universal across callers.

What falls *outside* this boundary is equally important. The utilities do not interpret semantic meaning of parsed values beyond structural rules (for example, they do not decide whether a path is safe to open or whether a socket address is reachable). They do not schedule work, persist configuration, or interact with guests. They do not own long-lived background tasks. If this component were removed or broken, every consumer that relies on uniform command-line behavior would need to fork or duplicate parsing rules, risking subtle divergence in help text, validation ordering, and error messages. Callers that depend on consistent time bases for metrics or rate limiting would likewise need to re-derive clock choices and conversion helpers. The blast radius is therefore “integration consistency and operational ergonomics” rather than “kernel correctness,” but the failure still fragments the user experience and complicates support.

---

## Interfaces & Contracts

The library exposes three conceptual surfaces to the rest of the workspace, each with a distinct contract.

The first surface is a declarative command-line description and parsing pipeline. Callers register a schema of named options before parsing. Each option can be required or optional, may consume zero or one or many trailing tokens as values, may default when omitted, and may participate in mutual dependency or exclusion rules relative to other registered names. The parser reads the host process argument vector (or an injected slice for tests), recognizes a dedicated end-of-options sentinel that partitions “passthrough” tokens, and materializes a structured view of what the user supplied. It can also emit a formatted usage synopsis derived from the same schema. The contract for callers is to build the schema once, keep string lifetimes coherent with that schema for the duration of parsing, and to query parsed results only through the provided accessors that distinguish flags, single values, and repeated values. In return, the library guarantees deterministic parsing order for a given token sequence, explicit errors for malformed input, and no silent coercion across value kinds.

The second surface is a minimal validation routine for instance identifiers. The contract is simple: a caller passes a string slice; the routine either accepts it as conforming to a documented character set and length window or rejects it with a position-aware error. There is no normalization step such as trimming or case folding; whatever the caller passes is evaluated as-is.

The third surface groups time-related helpers: conversion constants between second, millisecond, and nanosecond granularities; a selector for which Linux clock domain to query; functions that return monotonic, realtime, or CPU-time readings in various units; a combined snapshot pairing wall time and process CPU time at microsecond resolution; a fast monotonic counter path on common server architectures; and a human-oriented local timestamp type with string rendering. The contract is that these operations delegate to the platform’s standard library bindings for clocks and that arithmetic on conversions either succeeds or surfaces overflow as absence of a result rather than wrapping.

Consumers of the library include the optional instrumentation bridge, which can attach structured logging around selected operations when the corresponding feature is enabled at build time. That relationship is strictly additive: the core behavior is unchanged when the feature is off.

---

## Data Flow

Understanding how data moves clarifies where assumptions bite. At a high level, three inbound streams feed the system, and three corresponding outbound streams leave it.

The following diagram situates the library between host-provided inputs and embedding crates.

```
                    +------------------+
                    |  OS environment  |
                    |  (argv, clocks)  |
                    +--------+---------+
                             |
                             v
+------------------+  +------+------+  +------------------+
|  Caller-provided |  |   Shared    |  |  Structured      |
|  CLI schema +    |->|  utilities  |->|  parse results,  |
|  strings to      |  |  library    |  |  validated ids,  |
|  validate        |  +-------------+  |  time readings   |
+------------------+        |         +---------+----------+
                            |
                            v
                    +-------+--------+
                    |  Downstream    |
                    |  crates and    |
                    |  binaries      |
                    +----------------+
```

Before the diagram, the inputs deserve elaboration. The operating system supplies the process argument vector and the clock primitives. Callers supply the declarative schema for options, the literal strings subject to identifier validation, and implicit choices such as which clock to read when sampling duration or timestamps.

After the diagram, the outputs deserve equal attention. Structured parse results flow back as an internal map from registered names to classified values, plus a copied list of extra tokens captured after the end-of-options marker. Validation yields only success or a typed error. Time helpers yield numeric quantities or formatted strings, depending on the API path.

For the command-line path specifically, raw text tokens enter from the argument vector. The parser partitions the stream at the first bare double-hyphen token that is not introducing a long option name, treating everything to the right as opaque passthrough data for the embedding application. The left partition is scanned sequentially. Tokens beginning with the long-option prefix are interpreted as option heads; depending on schema, the next token may be consumed as a value. Help and version probes short-circuit the scan so that only a synthetic minimal result is stored, which lets callers branch to usage or version printing without fully validating unrelated requirements. When no short-circuit applies, values are attached to schema entries, defaults are applied where no user value exists, and cross-option rules are checked against the retained token list.

For validation, characters and rune indices flow through a single pass. No secondary representation is built unless the caller later copies the accepted string.

For time, kernel clock readings in seconds and nanoseconds are combined into a single unsigned nanosecond count for monotonic-style queries, or split into calendar fields for local display. Architecture-specific fast counters bypass the syscall path where available for coarse sequencing, with a fallback that still uses monotonic clock reads.

---

## Control Flow

Execution is driven by explicit calls from embedding code rather than callbacks or event loops. The critical path for configuration ingestion begins when a binary constructs a parser instance, registers options in a fluent chain, and invokes the parse entry point. That entry point pulls the live argument vector unless tests inject a synthetic slice.

The next figure outlines the decision tree inside parsing.

```
                    +-------------+
                    | Begin parse |
                    +------+------+
                           |
              +------------+------------+
              |                         |
              v                         v
     +--------+--------+       +--------+--------+
     | Help or version |       | Full option     |
     | token present?  |       | scan and attach |
     +--------+--------+       +--------+--------+
              |                         |
              v                         v
     +--------+--------+       +--------+--------+
     | Short-circuit:  |       | Validate        |
     | minimal state   |       | requirements    |
     +-----------------+       +--------+--------+
                                        |
                                        v
                               +--------+--------+
                               | Success or      |
                               | structured error|
                               +-----------------+
```

Prose before the figure: control first checks for documentation and version probes because those paths intentionally skip the heavier validation phase that enforces required fields and mutual constraints. This ordering is a product decision: a user asking for help should not be blocked by missing mandatory options that would matter only in real execution mode.

Prose after the figure: when the full scan runs, each candidate option head is validated for membership in the schema and for duplication rules before values are attached. Only after the linear pass completes does the system evaluate cross-option constraints. That two-phase split means local token grammar is enforced eagerly while global consistency is enforced once the full intent is known.

The validation routine for identifiers follows a straight-line path: length check, then per-character classification. Either the routine returns immediately on the first invalid character or completes successfully.

Time helpers follow straight-line syscall or intrinsic paths. The combined microsecond snapshot type initializes both chosen clocks when default-constructed, establishing a paired observation for latency-style measurements.

---

## State & Lifecycle

The command-line subsystem owns transient state for the duration of a parse operation on a given parser value. That state includes an ordered map from option names to their descriptors, each carrying optional user-supplied values, defaults, and metadata for help rendering. It also owns a growable list of passthrough strings captured after the end-of-options sentinel. Lifetimes tie schema strings to the caller’s static or borrowed text; the parser does not allocate copies of option names beyond what the underlying container requires.

Initialization happens through incremental registration: each fluent addition mutates the parser before any parsing occurs. Steady-state operation is the single parse call that consumes the argument vector and mutates descriptors in place. There is no concurrent access model; the expectation is one logical parse per configuration episode. Shutdown is implicit when the parser value goes out of scope.

The identifier validator is stateless. Each invocation is independent, which makes it safe to call from many threads provided inputs do not alias mutably across unsynchronized access in the caller.

Time helpers are likewise stateless aside from kernel-managed clock state. The local-time value type captures a snapshot of calendar fields and nanoseconds within a second for display; obtaining “now” performs fresh kernel queries each time.

Recovery is not a first-class concern inside this library. If parsing fails, the error is returned to the caller, which may choose to print diagnostics and exit or to retry with a different vector in tests. There is no partial commit of options that would leave the process half-configured unless the caller explicitly implements that pattern.

---

## Failure Modes

Failures are designed to be explicit and categorized rather than ambiguous.

For command-line parsing, several distinct failure families exist. Unknown tokens that resemble options but are not registered produce an unexpected-option class of error. Tokens that omit a required value where the schema demands one surface a missing-value class. Supplying the same option twice when duplicates are disallowed triggers a duplicate class. Required options absent from the vector yield a missing-required class. When mutual exclusion rules are violated, the error identifies the conflicting pair. When a dependency specifies that another option must appear alongside the first, absence triggers a missing-dependency class phrased as a missing argument for the dependent name. Positional arguments are not supported; bare tokens in the option section are treated as errors, which prevents silent misinterpretation of typos as file names.

Short-circuit paths for documentation and version probes can mask latent schema issues because validation of required fields is skipped entirely in those modes. Callers must branch on those probes before assuming production configuration is complete.

Passthrough capture after the sentinel is deliberately not validated as options; garbage in that region cannot be caught by this layer and will only fail later when downstream code interprets it.

For identifier validation, two failure kinds exist: length outside the inclusive bounds, or a character outside the allowed set with a zero-based index for quick pinpointing in logs. There is no attempt to suggest corrections.

For time conversions, checked multiplication prevents silent overflow when converting large second counts to nanoseconds; absence of a result signals overflow rather than wrapped values. Other time paths assume kernel calls succeed; a failed clock read is not translated into a rich error type in the public helpers and would surface through panics on defensive conversions where narrow casts are asserted. That is an intentional lean surface at the cost of strict recoverability from exotic platform failures.

Silent corruption is unlikely as long as callers respect value-kind contracts: interpreting a multi-value option as a single string will yield absence rather than a wrong string, which pushes mistakes toward “obvious missing data” rather than “subtly wrong data.” The largest subtlety is default merging: querying accessors merges defaults with user values, so a caller that prints “what the user typed” versus “effective configuration” must be deliberate.

---

## Operational Characteristics

Resource use is small and predictable. Parsing allocates for the passthrough list and for accumulating multiple values where allowed; the schema itself scales with the number of registered options, which is typically modest for embedded VMM style binaries. Maps keep names ordered, which trades a small CPU cost for deterministic iteration when rendering help.

Scaling bottlenecks are not central here because throughput is measured in single-digit parses per process lifetime. The heavier cost is human cognitive load when error messages diverge across binaries, which this library mitigates by centralization.

Observability is thin by design. There is no mandatory tracing; when the optional instrumentation feature is enabled, selected operations can emit structured logs via the workspace logging shim, but baseline builds rely on explicit error strings returned upward. No metrics or distributed traces are embedded in these helpers themselves.

Time calls incur syscall overhead proportional to how often callers sample. The fast counter path reduces syscall load for tight loops on supported hardware, at the expense of semantics that are not wall-clock time.

---

## Design Rationale

The overarching problem is duplication of fragile glue code across crates that must still feel like one product at the CLI and at the log line. A bespoke parser keeps dependency weight low and makes behavior legible in-tree, at the cost of maintaining more code than would be required if a general-purpose external crate were adopted. That tradeoff favors auditability and reproducibility in security-sensitive environments.

Choosing explicit error enums over stringly failures improves testability and allows callers to branch if needed, even though most binaries will simply format the display text for stderr.

Centralizing identifier rules ensures that what one component accepts another will not reject arbitrarily, assuming all call paths use the same routine. The lack of normalization is a deliberate constraint: silently changing user input can paper over security-relevant distinctions, so the validation layer stays strict and visible.

The time module standardizes clock choices so performance measurements across subsystems refer to comparable bases, while still exposing multiple clock domains because monotonic elapsed time, wall time for logs, and CPU time for utilization are genuinely different questions.

Optional instrumentation keeps the default build lean while allowing deeper diagnosis where permitted.

Taken together, the design optimizes for clarity, explicit failure, and shared conventions over maximal feature richness—a fit for a small, security-conscious utility layer that many crates touch but none should need to fork.

---

## Additional Topology View

The following ASCII sketch emphasizes that this library sits at the bottom of many dependency edges, not in the middle of request handling.

```
  +----------+     +----------+     +----------+
  | Binary A |     | Binary B |     | Library C|
  +----+-----+     +----+-----+     +----+-----+
       |                |                |
       +----------------+----------------+
                        |
                        v
                 +--------------+
                 | Shared       |
                 | utilities    |
                 +------+-------+
                        |
            +-----------+-----------+
            |                       |
            v                       v
     +--------------+      +--------------+
     | Platform     |      | Small set of |
     | clocks, argv |      | workspace    |
     |              |      | shims (opt.) |
     +--------------+      +--------------+
```

Before the figure: multiple entry points reuse the same foundation, which is why behavioral drift is the main risk this layer is meant to prevent.

After the figure: the downward edges represent operating system services and optional internal logging bridges rather than peer networking dependencies, reinforcing the library’s position as a leaf-ish dependency with a wide fan-in and narrow fan-out.
