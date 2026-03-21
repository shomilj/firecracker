# Shared utility crate

1. **Role and boundaries**

   1.1. The crate is a small, dependency-light Rust library that centralizes cross-cutting concerns for a VMM-style process: reading Linux kernel clocks, formatting human-readable local timestamps, validating identifiers used across components, and parsing long-form command-line options with relational constraints (required, mutually exclusive, and co-requisite flags).

   1.2. Design emphasis is on predictable, POSIX-aligned behavior on Linux: time reads go through standard monotonic, realtime, and process/thread CPU-time clocks where appropriate; local calendar rendering uses the C library’s local-time conversion from realtime seconds; CLI parsing follows GNU-style long options and a dedicated end-of-options sentinel.

   1.3. An optional compile-time feature wires in lightweight instrumentation hooks from a sibling logging crate. When disabled, the library has no tracing-specific surface area beyond ordinary dependencies.

2. **Time and measurement architecture**

   2.1. **Kernel clock abstraction**

   2.1.1. Four logical clock kinds are exposed as a small enumeration mapped one-to-one to POSIX clock identifiers on Linux: monotonic wall time (steady progression suitable for measuring elapsed intervals without calendar meaning), realtime wall time (the usual “wall clock” subject to administrator adjustments and meaningful for civil time), process-scoped CPU time (time charged to the process across its threads), and thread-scoped CPU time (time charged to the calling thread only).

   2.1.2. Downstream code selects a clock kind when asking for nanosecond, microsecond, or millisecond granularities; all granularities derive from a single nanosecond read so fractional units are consistent and not mixed from separate kernel calls that could drift relative to each other within one logical sample.

   2.2. **Nanosecond pipeline**

   2.2.1. Reading time fills a standard POSIX time structure, then combines whole seconds (converted to nanoseconds with checked multiplication) with the fractional nanosecond field. The public second-to-nanosecond conversion returns absence on overflow; the primary clock readers assume the conversion succeeds and aggregate into an unsigned nanosecond total. Extremely large second counts that overflow the checked multiply surface as conversion failure only on the dedicated conversion helper, not on the main read path—so the documented overflow behavior differs between the thin conversion API and the clock readers, which treat impossible kernel values as a hard failure at the point of aggregation.

   2.2.2. Public conversion constants fix the scaling factors between seconds, milliseconds, and nanoseconds so all unit transforms share one definition of “how large a second is” in fixed-point nanoseconds.

   2.2.3. Microsecond and millisecond results are obtained by integer division from the nanosecond total, so sub-unit precision is truncated toward zero rather than rounded; callers comparing microsecond output to nanosecond output divided by hand should expect exact alignment only where the division boundary allows it.

   2.3. **High-resolution cycle counter vs. portable clocks**

   2.3.1. On x86-64, a “timestamp in cycles” path returns the raw time-stamp counter from the CPU. That is not translated to wall time in this layer—it is a cheap, high-frequency ordinal useful for relative measurements on that architecture, with no guarantee of correspondence to seconds across cores, power states, or virtualization unless the platform provides such guarantees outside this crate.

   2.3.2. On non-x86-64 targets, the same entry point transparently uses monotonic nanoseconds from the kernel instead, so callers always get a monotonic increasing value without architecture-specific branching at the call site.

   2.4. **Local time with nanosecond display**

   2.4.1. “Now” in local civil time is obtained by reading realtime into a POSIX time structure, then passing the second count through the re-entrant local-time breakdown routine. The result keeps separate fields for second, minute, hour, month day, month index, year offset since 1900, plus that structure’s subsecond nanoseconds. The effective zone and daylight-saving interpretation follow the process environment and system zone database at the moment of the call, not a caller-supplied zone.

   2.4.2. Display formatting as a string follows a fixed pattern: ISO-like ordering with zero-padded month/day/hour/minute/second and a nine-digit fractional part. Month and year fields are adjusted in the formatter (month +1, year +1900) because the underlying breakdown uses zero-based month and years-since-1900, matching traditional broken-down civil-time conventions.

   2.4.3. Negative or leap-second edge cases are delegated entirely to the C library breakdown; the display path does not synthesize its own notion of civil time beyond formatting the fields returned.

   2.5. **Paired real vs. CPU microsecond snapshot**

   2.5.1. A small aggregate type pairs two microsecond-scale quantities: monotonic real time and process CPU time, both obtained via the same microsecond helper. Instantiating default state samples both at construction, giving a compact “where was the process on the wall clock vs. how much CPU it had used” checkpoint for metrics or logging. The two samples are not atomic with respect to each other beyond occurring back-to-back in the same default initializer.

   2.6. **Testing stance**

   2.6.1. Unit tests assert monotonicity (or at least non-decreasing behavior) for repeated reads of monotonic and CPU clocks, sanity on realtime nonzero values, consistency between nanosecond, microsecond, and millisecond derivations, display formatting for hand-crafted civil times, and overflow behavior on extreme second-to-nanosecond conversion.

   2.7. **Behavioral contract and observable quirks**

   2.7.1. The nanosecond read is the canonical sample: microsecond and millisecond views are pure views over that total, so a caller who records nanoseconds and later derives milliseconds will see the same truncation relationship the library uses internally—there is no hidden rounding mode or floating-point path.

   2.7.2. Monotonic time is the appropriate default for elapsed-interval and ordering questions; realtime is appropriate when the question is “what civil instant is it” or when correlating with external wall-clock timestamps. CPU-time clocks answer “how much execution resource was consumed,” which can diverge sharply from wall time under load or when blocked.

   2.7.3. The dedicated second-to-nanosecond conversion is the only place overflow is surfaced as a recoverable outcome; the main readers assume success and will abort the process if aggregation encounters a representation failure. That split matters for embedders who wrap or fuzz time APIs: tests may exercise the conversion helper’s edge cases without expecting the same outcomes from the fast clock readers unless the kernel could actually return such magnitudes.

   2.7.4. Local-time display is not timezone-stable across calls: two successive reads can straddle a zone transition and produce non-monotonic or surprising civil fields even while monotonic nanoseconds advance smoothly—callers mixing the two kinds of output should not assume one implies the other.

3. **Identifier validation**

   3.1. **Policy**

   3.1.1. A dedicated validator enforces a strict character set and length window for an “instance id” string: the UTF-8 byte length must fall between fixed inclusive bounds (short enough for practical UI and long enough to reject empty identifiers). The check uses byte length, not grapheme or Unicode scalar count, so a string that fits the byte budget may still contain fewer logical characters if those characters use multi-byte encodings.

   3.1.2. After length passes, the implementation scans Unicode scalar values one at a time. A character is allowed if it is a hyphen or if it satisfies the host language’s standard alphanumeric predicate for scalar values. That predicate is broader than ASCII-only letters and digits: many non-ASCII letters and numeric symbols are accepted. Callers who intend strictly host-name–style or ASCII-only labels should not rely on this validator alone for that narrower policy.

   3.1.3. Hyphens are explicitly allowed and are the only non-alphanumeric punctuation guaranteed by this path; underscores, spaces, slashes, colons, and other punctuation fail with a structured error naming the first offending character.

   3.2. **Error model**

   3.2.1. Errors are a small closed set of variants with structured payloads: invalid length carries actual length plus the configured min/max; invalid character carries the character and a zero-based index in the scalar iteration (character index, not UTF-8 byte offset). Display strings are generated automatically from those variants so user-facing messages stay stable and localized formatting can be layered consistently.

   3.2.2. An empty string fails the length rule before any per-character scan, so no character-level error is reported for emptiness.

   3.2.3. A string whose UTF-8 byte length exceeds the maximum fails on the length check without scanning every character, while a string that is within the length window but contains a disallowed character fails on first disallow with position information.

   3.3. **Operational characteristics**

   3.3.1. Validation is O(n) over Unicode scalar values with early exit on the first disallowed character; length is checked before scanning so overlong strings fail without a full pass when length alone violates policy.

   3.3.2. Because validation accepts only well-formed UTF-8 text at the boundary, ill-formed encodings cannot appear as input; callers supply valid Unicode sequences.

   3.4. **Edge cases and policy mismatches**

   3.4.1. A single-byte string that is one printable ASCII character passes both length and character rules; the minimum length bound is therefore one byte, not one grapheme, so a lone combining mark encoded in multiple bytes could satisfy length while still being an odd standalone label depending on higher-level policy.

   3.4.2. The reported position for invalid characters is the zero-based index in the scalar iteration, not the byte offset in the UTF-8 encoding. Diagnostics that show both the character and the index are therefore aligned with “which logical character failed,” not “at which byte offset,” which matters when correlating errors with hex dumps or network frames.

   3.4.3. Characters that normalize to different byte sequences in different Unicode normalization forms are not canonicalized here; two visually identical labels might compare unequal upstream while both passing this validator, or vice versa, depending on how peers perform equality.

   3.4.4. Underscores are rejected even though some naming conventions treat them as word separators; only hyphen is admitted among non-alphanumeric punctuation, so migration from underscore-heavy identifiers requires translation or a different validator.

4. **Command-line parsing architecture**

   4.1. **Overall shape**

   4.1.1. The parser is built from a registry of declared options and a separate bag for “extra” tokens that appear after an end-of-options sentinel. The registry keeps a stable ordering (lexicographic by option name) because it is backed by an ordered map—this affects help text ordering, not parsing semantics.

   4.1.2. Options are long-form only, prefixed with a double-dash; short single-dash tokens are not first-class options except that a conventional short help alias is recognized as a special case during the early scan.

   4.2. **Phased parse pipeline**

   The following ASCII diagram summarizes control flow from raw process arguments to validated state:

   ```
   program name dropped
          |
          v
   split at first bare "--"  ---------->  forwarded tail (opaque)
          |
          v
   help/version present? ----yes---->  inject synthetic flag, return OK
          |
          no
          v
   linear scan: for each token
          |
          +-- must start with "--" else Unexpected
          +-- must be a known name else Unexpected
          +-- if value-taking: next token must exist and must NOT look like a new option
          +-- accumulate into flag, single value, or multiple values
          |
          v
   validate_requirements:
          +-- every required option has a user value
          +-- if an option with co-requisites appeared, the left segment must contain the co-requisite’s long token
          +-- if an option forbids others, none of the forbidden long tokens may appear
   ```

   4.2.1. The first element of the process argument vector is always ignored as the program name. Everything after that is split at the first standalone end-of-options token: the left side is parsed as structured options; the right side is copied verbatim into an auxiliary list for the embedding application (for example forwarded guest arguments). Only the first such separator participates in splitting; any additional separators in the remainder stay inside the forwarded material as ordinary arguments.

   4.2.2. If either standard help token appears anywhere in the left segment, parsing short-circuits: only a synthetic help flag is recorded and the rest of the line is not interpreted. The same pattern applies to a standard version token, which records only a version flag. These shortcuts guarantee help and version never trip “missing value” or “unknown option” errors on the remainder of the left segment. Presence is detected by membership in the left segment, not by position, so help may appear after other tokens and still preempt full parsing.

   4.2.3. Otherwise, the left segment is walked sequentially. Each option must begin with the long-option prefix; the name is looked up in the registry. Duplicate appearance is rejected unless the schema explicitly allows multiple values for that name.

   4.2.4. Value-taking options consume the next argument element as their value unless that element itself begins with the long-option prefix, in which case the value is considered missing—this enforces that values cannot be omitted in favor of silently treating the next flag as a value. A value cannot be another option token even if that token would have been invalid; the parser commits to “missing value” first.

   4.2.5. For repeatable value-taking options, each occurrence consumes its own value; repeating the name without a following non-option token yields a missing-value error rather than a duplicate-name error in that situation.

   4.2.6. After structural parsing, a second pass enforces relational constraints: required options must have been assigned a user value; conditional requirements (if A appears, B’s flag must also appear) are checked by exact token equality against a synthesized expected token in the original left segment; mutual exclusions are enforced symmetrically by scanning for forbidden names when a trigger option is present. The co-requisite and forbid checks compare whole argument entries in the left segment, not substrings inside values.

   4.2.7. Boolean flags do not consume a following token; any token that looks like a bare value immediately after a flag in the linear scan is therefore interpreted as starting a new option or as an unexpected bare token, depending on form.

   4.3. **Value domain**

   4.3.1. Parsed values classify into three shapes: a boolean flag (presence means true), a single string, or an ordered list of strings. Multi-value mode is only available when the schema marks an option as repeatable; turning that on forces value-taking behavior because accumulation requires per-occurrence values.

   4.3.2. Defaults are represented as pre-seeded single string values; when serving callers, user-provided values override defaults. Flags do not use defaults in the same way—presence is explicit.

   4.3.3. Querying a value merges user value with default for value-taking options; flags are detected by stored value kind. A flag is considered present only when the stored value is the flag kind, not when a default might exist for other kinds.

   4.3.4. A schema-marked required option is satisfied only when the user actually supplied a value or flag on the command line; pre-seeded defaults do not count toward that requirement during validation, even though lookups can still resolve to the default when the option is optional. Practically, combining “required” with “default” forces the user to repeat an explicit assignment if the schema author intended the default to be a documentation hint only.

   4.4. **Query API on parsed state**

   4.4.1. Callers retrieve a single string, test flag presence, or obtain a shared slice of multiple strings by stable option name. Internally, resolution merges user value with default for value-taking options; flags are detected by stored value kind.

   4.4.2. Extra arguments after the end-of-options marker are exposed as a cloned vector so embedders can forward them without aliasing the parser’s internal storage.

   4.5. **Help generation**

   4.5.1. Help text is assembled by partitioning the schema into required vs. optional entries, then rendering each partition with aligned columns: the longest formatted name sets the column width so descriptions line up.

   4.5.2. Each option formats as an indented long name, with angle-bracket placeholders repeating the logical name for value-taking options. Descriptions append after a fixed triple-space gap; if both help prose and a default exist, the prose appears first followed by a bracketed default annotation; if only default exists, the bracketed default stands alone.

   4.6. **Error taxonomy and message semantics**

   4.6.1. **Forbidden combination** — Triggered when an option that forbids others was seen with a user value, and a forbidden option’s token appears in the left segment. The message names the two participants; ordering in the template reflects internal argument order, not command-line order.

   4.6.2. **Missing required option** — Either a schema-required option never received a user value, or an option with a co-requisite appeared but the co-requisite’s token was absent from the left segment. The same variant covers both “globally required” and “required because something else appeared.”

   4.6.3. **Missing value** — A value-taking option was recognized but the next token was missing or began with the long-option prefix.

   4.6.4. **Unexpected token** — A token did not begin with the long-option prefix on the left segment, or the name after the prefix was unknown, or an internal inconsistency occurred while merging repeatable values.

   4.6.5. **Duplicate** — The same option appeared twice when repetition was not allowed.

   4.6.6. Messages are stable, sentence-style strings suitable for CLI output; they are suitable for logs but are not themselves machine-parseable codes beyond the typed error variants exposed to callers.

   4.7. **Combinations and ordering caveats**

   4.7.1. Co-requirement checks require an exact token matching the conventional long form built from the related name; options must be declared with names consistent with that equality test.

   4.7.2. Short options are not generalized—only the help alias is special-cased; all other single-dash tokens on the left segment are unexpected unless they appear after the end-of-options marker in the extra-args bucket.

   4.7.3. Ordering of duplicate multi-value accumulation follows command-line order; the parser does not sort or deduplicate unless duplicate appearance itself is illegal.

   4.7.4. A trailing end-of-options marker with nothing after it yields an empty forward list; a missing value for a value-taking option remains an error even if the next token would have been the separator, because the separator is consumed only during the initial split, not during per-option value consumption.

   4.8. **Further interaction scenarios**

   4.8.1. If the first token after the program name is the end-of-options sentinel, the left segment is empty: no long options are parsed, required-option validation fails if the schema demands any, and the entire remainder (if any) becomes forwarded material—useful only when the embedding application intends “all guest args, no host flags.”

   4.8.2. Multiple end-of-options tokens can appear only if the first one is not followed immediately by another in the left segment; after the split, every token in the forwarded tail is opaque, including additional double-dash tokens, so a guest may pass its own long-looking tokens without the host parser interpreting them.

   4.8.3. Help and version preempt validation of required options: a minimal invocation that only requests help succeeds even when mandatory host options were declared in the schema, because the early exit records only the synthetic help or version state.

   4.8.4. A value-taking option whose value is intended to be a literal string beginning with the long-option prefix cannot express that on the left segment under the current rules—the next token would be rejected as a missing value. Such values must be placed after the end-of-options marker in the forwarded tail or carried through a different configuration channel.

   4.8.5. Repeatable value-taking options accumulate in encounter order; interleaving with other options preserves a total order over the collected list that mirrors the host segment’s token order, which matters when order encodes precedence (for example layered configuration).

   4.8.6. Relational checks (co-requisites and mutual exclusions) consider only arguments that successfully bound a user value in the first pass, including boolean flags. Options that never completed parsing—because of unknown names, missing values, or duplicates where disallowed—do not participate in that second pass, so ordering of errors can surface structural problems before relational ones.

5. **Security and trust considerations**

   5.1. **Argument handling**

   5.1.1. Parsing operates on a vector of strings already split by the runtime; it does not invoke a shell and does not perform glob expansion or variable interpolation. Risk from metacharacters is therefore delegated to whatever program constructs the argument vector (parent process, service manager, or remote API), not to this layer.

   5.1.2. Extra arguments are forwarded verbatim; embedders that pass those strings to guests or subprocesses inherit responsibility for quoting, capability boundaries, and injection into other languages’ contexts.

   5.1.3. Relational checks use equality of whole argument entries for forbidden and required companion tokens, which avoids naive substring spoofing on a single token but also means the policy is sensitive to exact spelling and duplicate-dash forms as emitted—callers should treat the left segment as structured data, not as free text.

   5.2. **Identifier validation**

   5.2.1. The validator reduces risk of pathological or confusing identifiers in cross-component protocols by bounding length and excluding most punctuation. It is not a full Unicode normalization or confusable-character audit; visually similar identifiers or homoglyphs are out of scope.

   5.2.2. Because alphanumeric classification is Unicode-aware, internationalized labels are accepted when they satisfy length and character rules; security policies that require ASCII-only labels need an additional check upstream or downstream.

   5.3. **Time and FFI**

   5.3.1. Wall-clock and CPU-time reads use small unsafe blocks around POSIX calls with stack-allocated structures and fixed clock identifiers; the high-resolution x86 path uses an intrinsic read of the cycle counter. Trust is placed in the kernel and CPU for sane return values; the nanosecond aggregation path assumes conversions succeed for realistic clock readings.

   5.3.2. Local time formatting inherits the platform C library’s notion of time zone and DST; incorrect or malicious zone configuration affects displayed civil time but not the monotonic measurement APIs.

   5.4. **Resource use and sensitive data**

   5.4.1. Parsing is linear in the number of tokens on the left segment and linear in the length of strings stored; there is no deliberate quadratic behavior, but extremely large argument lists produced by a hostile parent could still stress memory. Embeddings that accept remote configuration should cap argument count and length outside this crate.

   5.4.2. Error messages and help text may echo option names and values supplied by the user; logs that include parse errors should be treated as potentially containing secrets (tokens, paths) unless redacted upstream.

   5.4.3. Forwarded arguments are copied into an owned buffer for the embedder; that duplication is a deliberate isolation choice so later mutation of the parser’s internal state cannot silently change what will be passed along, at the cost of extra allocations proportional to forwarded token count and length.

6. **Cross-cutting engineering properties**

   6.1. **Safety and FFI**

   6.1.1. Unsafe blocks are limited to well-scoped POSIX calls with documented preconditions (valid pointers to stack-allocated structs, correct clock IDs) and architecture intrinsics where applicable.

   6.2. **Dependencies**

   6.2.1. Error handling uses conventional attribute-driven error typing for human-readable messages; the C library is accessed through the platform FFI bindings for clocks and local time.

   6.3. **Evolution**

   6.3.1. New validation rules should extend the small validator error taxonomy rather than ad-hoc boolean APIs; new CLI behaviors should extend the schema model (defaulting, repetition, forbids/requires) to keep the relational validation in one place.
