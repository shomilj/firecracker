# Seccomp BPF compiler

1. **Purpose and position in the stack**

   1.1. The component turns a declarative, JSON-encoded description of Linux seccomp policies into a compact binary artifact suitable for embedding or loading at runtime. Each policy is expressed as a set of syscall rules (optionally conditioned on register arguments) together with actions chosen from the standard seccomp vocabulary (allow, errno injection, kill, trap, log, trace). The heavy lifting of translating those rules into kernel-ready Berkeley Packet Filter (BPF) bytecode is delegated to the system’s libseccomp implementation; this layer orchestrates that translation, multiplexes multiple named policies in one build, and serializes the results for downstream consumers.

   1.2. The intended workflow is offline compilation: operators or build pipelines author human-readable JSON, run the command-line tool (or call the library entry point) with a chosen target CPU architecture, and produce a single file containing all compiled filters. A separate runtime in the same project can then deserialize that file under strict size limits and install the appropriate filter per thread or process. That split keeps the security-sensitive parsing and BPF generation out of the hot path while still allowing deterministic, reviewable policy sources.

2. **External dependencies and build coupling**

   2.1. The crate links against the native libseccomp shared library. The build script wires the linker to search a conventional system path and to pull in the `seccomp` library by name. Deployment therefore assumes a development or runtime environment where that library is present and ABI-compatible; the Rust side does not vendor seccomp logic.

   2.2. A thin foreign-function interface exposes only the subset of libseccomp needed for this pipeline: filter context creation, architecture registration, syscall name resolution, rule insertion (both simple and argument-array forms), and BPF export to a file descriptor. Constants mirror libseccomp’s action and architecture tokens so that JSON-derived enums can be converted without going through a higher-level Rust wrapper crate.

3. **Input language (JSON model)**

   3.1. The top-level JSON value deserializes to an ordered map from string keys to filter objects. Keys are opaque names (for example per-thread identifiers in the embedding application); their lexical ordering is preserved through the map type used, which matters for reproducible binary output and stable diffing of artifacts.

   3.2. Each filter object carries three conceptual parts: a default action, a rule-match action, and a list of syscall rules. The default action applies when an invoked syscall is not matched by any rule (or not matched strongly enough—see argument matching below). The rule-match action is the seccomp action libseccomp attaches to every `rule_add` invocation for that filter; it is the consequence of “this rule fired.” That split matches common seccomp patterns: a conservative default (e.g., kill or errno) with explicit allow rules, or an allow default with explicit denials.

   3.3. Each syscall rule names a syscall by its C ABI string (resolved to a number at compile time via libseccomp) and optionally carries a list of argument conditions. When conditions are present, they are combined conjunctively: every condition must hold for the rule to match. When they are absent, the rule matches any invocation of that syscall with any arguments (subject to the “basic” compilation mode—see below).

   3.4. Conditions are indexed by argument position (zero-based, matching kernel seccomp’s view of syscall arguments). Each condition specifies a comparison operator, a value, and whether the comparison should treat the argument as a 32-bit or 64-bit quantity. Operators cover inequality and equality relations plus masked equality for bitfields.

4. **Architecture selection**

   4.1. The compiler is explicitly told which seccomp architecture token to add to each filter context. Only two targets are supported in the typed enum: 64-bit x86 and 64-bit ARM. The choice is case-insensitive on input. This matters because syscall numbers and argument layouts differ by architecture; libseccomp uses the token to emit correct BPF for the selected ABI.

   4.2. After creating a filter context with the chosen default action, the pipeline adds the selected architecture. If the architecture is already present, libseccomp reports an “already exists” condition that is treated as success; other failures abort compilation. Rules added afterward are associated with that architecture context.

5. **Compilation pipeline (end-to-end)**

   5.1. **Parse and validate JSON.** The entire input file is read into memory as UTF-8 text and parsed. JSON errors surface as compilation failures distinct from I/O failures opening or reading the file.

   5.2. **Per named filter: build a libseccomp context.** For each entry in the map, a fresh filter handle is allocated with the deserialized default action. The target architecture is registered. Failure to initialize or add the architecture yields structured errors.

   5.3. **Per syscall rule: resolve and install.** The syscall name is resolved to a number. Failure to resolve is fatal for that compilation (typo or unsupported name). Then either:

   - **Basic mode (deprecated):** A compatibility switch forces every rule to be installed without argument comparators, regardless of what the JSON described. This uses the variadic “add rule with zero argument checks” API only. It effectively drops fine-grained argument constraints and collapses behavior to “this syscall always triggers the rule action.” It exists for backward compatibility and is explicitly discouraged.

   - **Full mode with argument checks:** If the rule lists argument conditions, each condition is translated into libseccomp’s C structure for comparator chains, and the array-based rule API is invoked so that all comparators apply in one rule (AND semantics).

   - **Full mode without argument checks:** If no conditions are listed, the simple rule API adds a syscall-wide rule with no argument filters—matching any call to that syscall.

   5.4. **Export BPF to an anonymous memory-backed file.** A memory-backed file descriptor is created with a fixed name string suitable for memfd APIs. The filter context is exported as raw BPF bytecode into that descriptor. The handle is wrapped as a standard file object for convenience.

   5.5. **Measure, allocate, read.** After export, the implementation rewinds the memfd, queries its size, and treats the bytecode as a sequence of 64-bit words (matching BPF instruction width expectations in this toolchain). A buffer of the right length is filled by a single exact read, then the memfd is rewound again so the same scratch space can be reused for the next named filter. This avoids persisting intermediate BPF on disk and keeps each filter’s bytecode isolated in memory before aggregation.

   5.6. **Aggregate and serialize.** Successfully compiled instruction vectors are inserted into an ordered map under their original string keys. The entire map is encoded with bincode using little-endian fixed-integer encoding and an explicit maximum byte budget on decode. That configuration is part of the public API so that loaders can share the exact same limits and endianness. The output file is created (truncating if present) and the encoded bytes are written in one shot.

6. **Binary output format (conceptual contract)**

   6.1. Consumers should treat the file as a bincode-encoded associative structure from UTF-8 strings to vectors of 64-bit unsigned machine words representing BPF instructions. The serialization is not JSON; it is a compact, deterministic binary schema meant for trusted loading alongside the matching deserialization limit.

   6.2. The byte limit on deserialization exists to mitigate memory exhaustion if a corrupted or malicious artifact claims an enormous container length. The limit is sized with domain knowledge: kernel seccomp BPF programs have a bounded instruction count, and the embedding product only tracks finitely many threads, so the cap is a deliberate safety rail rather than a generic parser limit.

7. **Semantic details of condition lowering**

   7.1. Most comparison operators map one-to-one onto libseccomp’s comparison enum: greater-or-equal, greater-than, less-or-equal, less-than, not-equal, and masked equality with an explicit mask operand carried in the JSON variant.

   7.2. **Equality and argument width.** Plain equality behaves differently depending on whether the policy marks the argument as dword or qword. For quadword equality, a direct equality comparator is used on the full 64-bit value. For dword equality, the implementation deliberately lowers to a masked equality that clears the upper 32 bits in the mask and compares only the lower 32 bits against the supplied value. The rationale is pragmatic: some libc implementations have been observed to leave undefined bits in the upper half of certain syscall arguments (for example ioctl request codes on musl). Full 64-bit equality would then reject otherwise valid calls. The masked form enforces “high half must read as zero” while comparing the low half, at the cost of an extra BPF instruction that optimizers may fold away. This is a behavioral guarantee of the compiler, not merely a mechanical mapping.

8. **Command-line interface**

   8.1. The binary front end accepts a required target architecture string, a required input JSON path, an optional output path defaulting to a conventional filename, and the deprecated “basic” flag described above. It delegates directly to the library compilation routine and propagates errors as process exit status via the error type’s display formatting chain.

9. **Error taxonomy and failure philosophy**

   9.1. Errors are classified for observability: file open/read, JSON parse, unknown architecture string, libseccomp failures at each phase (context, arch, syscall resolution, rule installation, BPF export), memfd lifecycle failures, output file creation, and bincode encoding. There is no partial output on failure: a problem in one named filter aborts the whole compilation.

   9.2. The design favors fail-fast semantics appropriate for build-time tooling: mis-specified policies or environment issues surface immediately rather than silently producing empty or partial artifacts.

10. **Safety and process assumptions**

    10.1. Numerous calls into libseccomp are `unsafe` in Rust terms because they are extern C APIs. Call sites document the conventional assumptions: valid pointers, correct enum values, and matching lifetimes (the filter context remains valid until export completes). The memfd file descriptor is owned and closed via the file wrapper.

    10.2. Syscall name strings are carried as owned NUL-terminated C strings in the deserialized types so they can be passed to C APIs without additional allocation at the FFI boundary beyond what the deserializer already produced.

11. **Conceptual data-flow diagram**

```
  JSON file (UTF-8)
        |
        v
   +------------------+
   | Parse to in-mem |
   | policy: map of   |
   | named filters    |
   +------------------+
        |
        | for each named filter
        v
   +------------------+     +------------------+
   | libseccomp: init | --> | add arch token   |
   | default action   |     | (x86_64/AArch64) |
   +------------------+     +------------------+
        |
        | for each syscall rule
        v
   +------------------+     +------------------+
   | resolve syscall  |     | add rule(s) with |
   | name -> number   |     | optional AND of  |
   +------------------+     | arg comparators  |
        |                  +------------------+
        v
   +------------------+
   | export BPF ->    |
   | memfd stream     |
   +------------------+
        |
        v
   +------------------+
   | read u64[]       |
   | (instruction img)|
   +------------------+
        |
        +-----> accumulate in map: name -> instructions
        |
        v
   +------------------+
   | bincode encode   |
   | (little-endian,  |
   |  fixed int, cap) |
   +------------------+
        |
        v
   output binary file
```

12. **Relationship to runtime consumers**

    12.1. The artifact is not itself an installed seccomp policy; it is a portable bundle of precompiled BPF programs keyed by name. A runtime must select the correct entry, install it with `prctl`/`seccomp` APIs appropriate to the host kernel, and honor the same architectural assumptions used at compile time. The deserialization configuration exposed by the library is the contract for that loading path.

    12.2. Because policies are compiled ahead of time, changes to JSON require recompilation to take effect; there is no interpreter in this crate. That trade-off favors auditability and minimizes attack surface on the VMM’s hot path.
