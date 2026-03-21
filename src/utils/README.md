# Shared utility crate

1. **Role and boundaries**

   1.1. The crate is a small, dependency-light Rust library that centralizes cross-cutting concerns for a VMM-style process: reading Linux kernel clocks, formatting human-readable local timestamps, validating identifiers used across components, and parsing long-form command-line options with relational constraints (required, mutually exclusive, and co-requisite flags).

   1.2. Design emphasis is on predictable, POSIX-aligned behavior on Linux: time reads go through the standard monotonic/realtime/CPU-time clocks where appropriate; local calendar rendering uses the C library’s local-time conversion from realtime seconds; CLI parsing follows GNU-style long options and a dedicated end-of-options sentinel.

   1.3. An optional compile-time feature wires in lightweight instrumentation hooks from a sibling logging crate. When disabled, the library has no tracing-specific surface area beyond ordinary dependencies.

2. **Time and measurement architecture**

   2.1. **Kernel clock abstraction**

   2.1.1. Four logical clock kinds are exposed as a small enumeration mapped one-to-one to Linux `clockid_t` values: monotonic wall time (not subject to NTP step adjustments in the usual sense), realtime wall time (calendar-related), process-scoped CPU time, and thread-scoped CPU time.

   2.1.2. Downstream code selects a clock kind when asking for nanosecond, microsecond, or millisecond granularities; all granularities derive from a single nanosecond read to avoid inconsistent rounding across units.

   2.2. **Nanosecond pipeline**

   2.2.1. Reading time fills a POSIX `timespec`, then combines whole seconds (converted to nanoseconds with checked multiplication) with the fractional nanosecond field. Overflow on the second→nanosecond step is treated as a conversion failure at the API boundary for the helper that only converts seconds; the primary clock readers assume the kernel returns values that fit the expected ranges and bridge into unsigned nanosecond counts for the common case.

   2.2.2. Public conversion constants fix the scaling factors between seconds, milliseconds, and nanoseconds so all unit transforms share one definition of “how large a second is” in fixed-point nanoseconds.

   2.3. **High-resolution cycle counter vs. portable clocks**

   2.3.1. On x86-64, a “timestamp in cycles” path returns the raw time-stamp counter from the CPU. That is not translated to wall time in this layer—it is a cheap, high-frequency ordinal useful for relative measurements on that architecture.

   2.3.2. On non-x86-64 targets, the same entry point transparently uses monotonic nanoseconds from the kernel instead, so callers always get a monotonic increasing value without architecture-specific branching at the call site.

   2.4. **Local time with nanosecond display**

   2.4.1. “Now” in local civil time is obtained by reading realtime into a timespec, then passing the second count through the re-entrant local-time breakdown routine. The result keeps separate fields for second, minute, hour, month day, month index, year offset since 1900, plus the timespec’s subsecond nanoseconds.

   2.4.2. Display formatting as a string follows a fixed pattern: ISO-like ordering with zero-padded month/day/hour/minute/second and a nine-digit fractional part. Month and year fields are adjusted in the formatter (month +1, year +1900) because the underlying breakdown uses zero-based month and years-since-1900, matching traditional `tm` conventions.

   2.5. **Paired real vs. CPU microsecond snapshot**

   2.5.1. A small aggregate type pairs two microsecond-scale quantities: monotonic real time and process CPU time, both obtained via the same microsecond helper. Instantiating default state samples both at construction, giving a compact “where was the process on the wall clock vs. how much CPU it had used” checkpoint for metrics or logging.

   2.6. **Testing stance**

   2.6.1. Unit tests assert monotonicity (or at least non-decreasing behavior) for repeated reads of monotonic and CPU clocks, sanity on realtime nonzero values, consistency between ns/us/ms derivations, display formatting for hand-crafted civil times, and overflow behavior on extreme second→nanosecond conversion.

3. **Identifier validation**

   3.1. **Policy**

   3.1.1. A dedicated validator enforces a strict character set and length window for an “instance id” string: length must fall between fixed inclusive bounds (short enough for practical UI and long enough to reject empty identifiers).

   3.1.2. Allowed characters are ASCII letters and digits plus hyphen; any other character fails validation and reports the offending character and its byte/character index.

   3.2. **Error model**

   3.2.1. Errors are a small closed enum with structured payloads: invalid length carries actual length plus the configured min/max, invalid character carries the character and position. Display strings are generated via derive macros so user-facing messages stay stable and localized formatting can be layered consistently.

   3.3. **Operational characteristics**

   3.3.1. Validation is O(n) over Unicode scalar values with early exit on the first disallowed character; length is checked before scanning so overlong strings fail without a full pass when length alone violates policy.

4. **Command-line parsing architecture**

   4.1. **Overall shape**

   4.1.1. The parser is built from a registry of declared options and a separate bag for “extra” tokens that appear after an end-of-options sentinel. The registry keeps a stable ordering (lexicographic by option name) because it is backed by an ordered map—this affects help text ordering, not parsing semantics.

   4.1.2. Options are long-form only, prefixed with a double-dash; short single-dash tokens are not first-class options except that a conventional short help alias is recognized as a special case during the early scan.

   4.2. **Phased parse pipeline**

   The following ASCII diagram summarizes control flow from raw argv to validated state:

   ```
   argv[0] dropped
          |
          v
   split at first bare "--"  ---------->  extra_args[] (opaque forward)
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
          +-- accumulate into Flag / Single / Multiple
          |
          v
   validate_requirements:
          +-- every required option has a user value
          +-- if an option with co-requisites appeared, argv must contain the co-requisite’s "--name"
          +-- if an option forbids others, none of the forbidden "--name" tokens may appear
   ```

   4.2.1. The first element of the process argument vector is always ignored as the program name. Everything after that is split at the first standalone `--` token: left side is parsed as structured options; right side is copied verbatim into an auxiliary list for the embedding application (for example forwarded guest arguments).

   4.2.2. If either standard help token appears anywhere in the left segment, parsing short-circuits: only a synthetic help flag is recorded and the rest of the line is not interpreted. The same pattern applies to a standard version token, which records only a version flag. These shortcuts guarantee help/version never trip “missing value” or “unknown option” errors on the remainder of the argv slice.

   4.2.3. Otherwise, the left segment is walked sequentially. Each option must begin with the long-option prefix; the name is looked up in the registry. Duplicate appearance is rejected unless the schema explicitly allows multiple values for that name.

   4.2.4. Value-taking options consume the next argv element as their value unless that element itself begins with the long-option prefix, in which case the value is considered missing—this enforces that values cannot be omitted in favor of silently treating the next flag as a value.

   4.2.5. After structural parsing, a second pass enforces relational constraints: required options must have been assigned a user value; conditional requirements (if A appears, B’s flag must also appear) are checked by substring presence of the required `--b` token in the original left segment; mutual exclusions are enforced symmetrically by scanning for forbidden names when a trigger option is present.

   4.3. **Value domain**

   4.3.1. Parsed values classify into three shapes: a boolean flag (presence means true), a single string, or an ordered list of strings. Multi-value mode is only available when the schema marks an option as repeatable; turning that on forces value-taking behavior because at least one value is required in practice.

   4.3.2. Defaults are represented as pre-seeded single string values; when serving callers, user-provided values override defaults. Flags do not use defaults in the same way—presence is explicit.

   4.4. **Query API on parsed state**

   4.4.1. Callers retrieve a single string, test flag presence, or obtain a shared slice of multiple strings by stable option name. Internally, resolution merges user value with default for value-taking options; flags are detected by discriminant.

   4.4.2. Extra arguments after `--` are exposed as a cloned vector so embedders can forward them without aliasing the parser’s internal storage.

   4.5. **Help generation**

   4.5.1. Help text is assembled by partitioning the schema into required vs. optional entries, then rendering each partition with aligned columns: the longest formatted name sets the column width so descriptions line up.

   4.5.2. Each option formats as a indented long name, with `<name>` placeholders for value-taking options. Descriptions append after a fixed triple-space gap; if both help prose and a default exist, the prose appears first followed by a bracketed default annotation; if only default exists, the bracketed default stands alone.

   4.6. **Error taxonomy**

   4.6.1. Errors distinguish: unknown or misplaced tokens, missing required options, missing values for value-taking options, duplicates where disallowed, and forbidden pairwise combinations. Messages are stable, sentence-style strings suitable for CLI output.

   4.7. **Limitations implied by the design**

   4.7.1. Co-requirement checks look for literal `--name` substrings in the token list; options must be declared in a consistent naming scheme for that check to match.

   4.7.2. Short options are not generalized—only the help alias is special-cased; all other single-dash tokens are unexpected unless they appear after `--` in the extra-args bucket.

   4.7.3. Ordering of duplicate multi-value accumulation follows argv order; the parser does not sort or deduplicate unless duplicate appearance itself is illegal.

5. **Cross-cutting engineering properties**

   5.1. **Safety and FFI**

   5.1.1. Unsafe blocks are limited to well-scoped POSIX calls with documented preconditions (valid pointers to stack-allocated structs, correct clock IDs).

   5.2. **Dependencies**

   5.2.1. Error handling uses a derive-based error crate and displaydoc for human-readable messages; the C library is accessed through the libc crate for clocks and local time.

   5.3. **Evolution**

   5.3.1. New validation rules should extend the small validator enum rather than ad-hoc boolean APIs; new CLI behaviors should extend the schema model (defaulting, repetition, forbids/requires) to keep the relational validation in one place.
