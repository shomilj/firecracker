# Seccomp BPF compiler

1. **Purpose and position in the stack**

   1.1. The component turns a declarative, JSON-encoded description of Linux seccomp policies into a compact binary artifact suitable for embedding or loading at runtime. Each policy is expressed as a set of syscall rules (optionally conditioned on register arguments) together with actions chosen from the standard seccomp vocabulary (allow, errno injection, kill, trap, log, trace). The heavy lifting of translating those rules into kernel-ready Berkeley Packet Filter (BPF) bytecode is delegated to the system’s native seccomp library; this layer orchestrates that translation, multiplexes multiple named policies in one build, and serializes the results for downstream consumers.

   1.2. The intended workflow is offline compilation: operators or build pipelines author human-readable JSON, run the command-line tool (or call the library entry point) with a chosen target CPU architecture, and produce a single file containing all compiled filters. A separate runtime in the same project can then deserialize that file under strict size limits and install the appropriate filter per thread or process. That split keeps the security-sensitive parsing and BPF generation out of the hot path while still allowing deterministic, reviewable policy sources.

2. **External dependencies and build coupling**

   2.1. The crate links against the native seccomp shared library. The build script wires the linker to search a conventional system path and to pull in the seccomp library by name. Deployment therefore assumes a development or runtime environment where that library is present and ABI-compatible; the Rust side does not vendor seccomp logic.

   2.2. A thin foreign-function interface exposes only the subset of the native API needed for this pipeline: filter context creation, architecture registration, syscall name resolution, rule insertion (both simple and argument-array forms), and BPF export to a file descriptor. Constants mirror the library’s action and architecture tokens so that JSON-derived enums can be converted without going through a higher-level Rust wrapper crate.

3. **JSON schema mental model**

   3.1. **Root document shape.** The top-level JSON value deserializes to an ordered map from string keys to filter objects. Keys are opaque names chosen by the embedding application (for example per-thread or per-role identifiers). The map type preserves lexical ordering of keys. That ordering is not required for seccomp semantics, but it matters for reproducible binary output: the same policy text always serializes to the same key order in the downstream binary envelope, which stabilizes diffs of build artifacts and avoids nondeterministic layout from hash-based map types.

   3.2. **Filter object.** Each value in the root map is a single filter configuration with three conceptual parts: a default action, a rule-match action, and a list of syscall rules. The default action applies when an invoked syscall is not matched by any rule (or when no rule’s argument conditions are satisfied—see argument matching below). The rule-match action is the seccomp action the native library attaches to every successful “add rule” call for that filter; it is the consequence of “this rule fired” for syscall rules defined in that filter. This split matches common seccomp patterns: a conservative default (for example kill or errno) with explicit allow rules, or an allow default with explicit denials.

   3.3. **Syscall rules.** Each rule names a syscall by its C ABI string (resolved to a number at compile time via the native resolver) and optionally carries a list of argument conditions. Syscall names are modeled as nul-terminated byte strings at the deserialization boundary so they can be handed to the C resolver without extra copying logic beyond what the deserializer already performs.

   3.4. **Actions in JSON.** Actions deserialize from snake_case enum tags. Variants that carry a payload—errno code for simulated failures, trace value for notification—embed that number in the JSON form. The mapping to the library’s numeric action constants is a straight table on the Rust side.

   3.5. **Comparison operators in JSON.** Operators deserialize from snake_case names. Plain masked equality exposes the mask as a nested value in the variant. The schema distinguishes “compare this argument as 32 bits” versus “64 bits” only for plain equality; other operators always use the library’s native 64-bit comparison path on the datum fields (see lowering).

   3.6. **Mental model vs. wire format.** Authors think in JSON: named filters, ordered rule lists, and per-rule argument clauses. The runtime consumer thinks in bincode: an associative container of string keys to instruction vectors. No JSON remains in the artifact; the only bridge is the compile step described later.

4. **Argument matchers**

   4.1. **Indexing.** Conditions are indexed by argument position using zero-based indices aligned with the kernel seccomp view of syscall arguments for the selected architecture. The index selects which register slot participates in every comparator in that condition.

   4.2. **Conjunction.** When a syscall rule lists multiple conditions, they are combined conjunctively: every condition must hold for the rule to match. The native library’s array-based rule API installs one rule whose comparators are chained with logical AND in the generated BPF.

   4.3. **No OR across conditions.** Expressing alternative argument patterns requires separate syscall rules (possibly repeating the same syscall name with different condition sets). First-match semantics at the seccomp layer are inherited from the underlying library and kernel behavior; the JSON lists rules in author order, but authors should consult seccomp documentation for how multiple rules on the same syscall interact with priorities.

   4.4. **Absence of conditions.** If a rule omits the argument list (or provides an empty list), the rule matches any invocation of that syscall with any arguments, subject to the deprecated “basic” compilation switch described below.

   4.5. **Operator repertoire.** The policy language exposes inequality and equality relations, plus masked equality for bitfields. Inequality operators map directly to the native comparison enumerants for greater-or-equal, greater-than, less-or-equal, less-than, and not-equal. Masked equality takes an explicit mask operand in addition to the compare value.

   4.6. **Plain equality and argument width.** Plain equality is special-cased (see semantic lowering). The policy marks each equality test as either “dword” (32-bit) or “qword” (64-bit). That flag does not change the JSON numeric type of the value (always a 64-bit unsigned integer in the model); it controls how the compiler lowers the test into BPF comparators.

5. **Architecture selection**

   5.1. The compiler is explicitly told which seccomp architecture token to add to each filter context. Only two targets are supported in the typed enum: 64-bit x86 and 64-bit ARM. The choice is case-insensitive on input. This matters because syscall numbers and argument layouts differ by architecture; the native library uses the token to emit correct BPF for the selected ABI.

   5.2. After creating a filter context with the chosen default action, the pipeline adds the selected architecture. If the architecture is already present, the library reports an “already exists” condition that is treated as success; other failures abort compilation. Rules added afterward are associated with that architecture context.

6. **Endianness and in-memory representation**

   6.1. **Binary artifact.** The serialized output uses the bincode format configured for little-endian layout of multibyte integers and fixed-width integer encoding. Any consumer that decodes the artifact must use the same endianness and fixed-integer options; mismatch would misinterpret lengths and payloads.

   6.2. **BPF instruction image.** Exported BPF is written by the native library as a byte stream to a memory-backed file descriptor, then read back into a buffer of unsigned 64-bit words. Each word corresponds to one eight-byte BPF instruction chunk as consumed by the kernel interface for installing filters; using 64-bit words satisfies alignment expectations. The byte stream from export is interpreted according to the host’s native byte order when filling those words—this compile step is always run for the same architecture class as the target token, so host and policy target are consistent in practice.

   6.3. **No cross-endian compilation.** The tool is not designed to emit BPF for an endianness different from the build host’s; cross-compilation scenarios should run the compiler on a matching host or toolchain that preserves the same assumptions.

7. **Compilation pipeline (end-to-end)**

   7.1. **Parse and validate JSON.** The entire input file is read into memory as UTF-8 text and parsed. JSON errors surface as compilation failures distinct from I/O failures opening or reading the file.

   7.2. **Per named filter: build a native filter context.** For each entry in the root map, a fresh filter handle is allocated with the deserialized default action. The target architecture is registered. Failure to initialize or add the architecture yields structured errors.

   7.3. **Per syscall rule: resolve and install.** The syscall name is resolved to a number. Failure to resolve is fatal for that compilation (typo or unsupported name). Then either:

   - **Deprecated basic mode:** A compatibility switch forces every rule to be installed without argument comparators, regardless of what the JSON described. This uses only the variadic “add rule with zero argument checks” path. It drops fine-grained argument constraints and collapses behavior to “this syscall always triggers the rule-match action when hit.” It exists for backward compatibility and is explicitly discouraged.

   - **Full mode with argument checks:** If the rule lists argument conditions, each condition is translated into the native C structure for comparators, and the array-based rule API is invoked so that all comparators apply in one rule (AND semantics).

   - **Full mode without argument checks:** If no conditions are listed, the simple rule API adds a syscall-wide rule with no argument filters—matching any call to that syscall.

   7.4. **Export BPF to an anonymous memory-backed file.** A memory-backed file descriptor is created with a fixed conventional name string suitable for memfd-style APIs. The filter context is exported as raw BPF bytecode into that descriptor. The handle is wrapped as a standard file object for convenience.

   7.5. **Measure, allocate, read.** After export, the implementation rewinds the descriptor, queries its size, and treats the bytecode as a sequence of 64-bit words. A buffer of the right length is filled by a single exact read, then the descriptor is rewound again so the same scratch space can be reused for the next named filter. This avoids persisting intermediate BPF on disk and keeps each filter’s bytecode isolated in memory before aggregation.

   7.6. **Aggregate and serialize.** Successfully compiled instruction vectors are inserted into an ordered map under their original string keys. The entire map is encoded with bincode using the public configuration (little-endian, fixed integers, and an explicit maximum byte budget on decode). The output file is created (truncating if present) and the encoded bytes are written in one shot.

8. **Libseccomp lowering (conceptual)**

   8.1. **Initialization.** A new filter state is created with the filter’s default action. This establishes the “no rule matched” behavior for syscalls not covered by explicit rules.

   8.2. **Architecture binding.** The chosen architecture token is added once per filter. Rules added later are attributed to that architecture’s syscall namespace and register layout.

   8.3. **Syscall resolution.** Human-readable syscall names are resolved to integers. Negative pseudo-numbers are legal return values for some architectures; only the dedicated error sentinel indicates resolution failure.

   8.4. **Rule installation paths.** Two insertion paths exist: a variadic form with zero comparators (syscall-wide match) and an array form carrying a contiguous list of comparator structures. The compiler selects among them based on deprecated mode and whether the JSON listed argument conditions. The action attached to every insertion is the filter’s rule-match action, not the default action.

   8.5. **Comparator structure population.** Each JSON condition becomes one C comparator: argument index, operation code, and two datum fields whose meaning depends on the operation (for example mask and value for masked equality, or a single threshold for inequalities). Plain equality lowering is special; see below.

   8.6. **BPF emission.** The native library generates BPF and writes it to the provided file descriptor. The Rust side does not transform instruction opcodes; it only transports bytes. Any optimization or architecture-specific fixups happen inside the native dependency.

9. **Binary output format (bincode contract)**

   9.1. **Logical type.** Consumers should treat the file as a bincode-encoded associative structure mapping UTF-8 strings to vectors of 64-bit unsigned machine words representing BPF instructions. The serialization is not JSON; it is a compact, deterministic binary schema meant for trusted loading alongside a matching deserialization limit.

   9.2. **Configuration parity.** The encoding uses a published configuration object so that loaders can share the exact same limits and endianness: little-endian, fixed-width integer encoding, and a byte cap on the total decoded input. Drift between encoder and decoder configurations is a correctness bug for any release artifact.

   9.3. **Length guard.** The decode byte limit mitigates memory exhaustion if a corrupted or malicious artifact claims an enormous container length. The limit is sized with domain knowledge: kernel seccomp BPF programs have a bounded instruction count, and the embedding product only tracks finitely many named filters, so the cap is a deliberate safety rail rather than a generic parser default.

   9.4. **Key ordering.** Because encoding starts from an ordered map type, the binary layout reflects sorted key order. Decoders may use unordered associative types on read; they should not rely on iteration order unless they re-sort or normalize keys according to their own policy.

   9.5. **String keys.** Keys are UTF-8 strings in the bincode stream. Any normalization (for example case folding) is the responsibility of the loader; the compiler preserves keys as deserialized from JSON.

10. **Semantic details of condition lowering**

    10.1. **Inequality and masked equality.** Inequality operators and explicit masked equality map one-to-one onto the native comparison enumerants. Masked equality passes the mask and value into the datum fields expected by the library.

    10.2. **Plain equality and argument width.** Plain equality behaves differently depending on whether the policy marks the argument as dword or qword. For quadword equality, a direct equality comparator is used on the full 64-bit value. For dword equality, the implementation deliberately lowers to a masked equality that clears the upper 32 bits in the mask and compares only the lower 32 bits against the supplied value. The rationale is pragmatic: some C library implementations have been observed to leave undefined bits in the upper half of certain syscall arguments (for example ioctl request codes on musl). Full 64-bit equality would then reject otherwise valid calls. The masked form enforces “high half must read as zero” while comparing the low half, at the cost of an extra BPF instruction that optimizers may fold away. This is a behavioral guarantee of the compiler, not merely a mechanical mapping.

    10.3. **Non-equality operators and dword.** The dword or qword annotation on non-equality comparisons is not used to branch the lowering in the same way; those operators always populate the native structure with the full 64-bit datum for the threshold-style comparisons.

11. **Deprecated modes**

    11.1. **Basic compilation switch.** A boolean flag selects deprecated “basic” behavior. When enabled, every syscall rule is installed without argument comparators even if the JSON specified them. The rule-match action still applies to those syscall-wide rules; only argument filtering is stripped.

    11.2. **Why it exists.** The mode preserves compatibility with older workflows that relied on syscall-only rules and could not express or did not want argument checks. New policies should leave this flag disabled so that JSON accurately reflects installed BPF.

    11.3. **Removal trajectory.** The source marks the flag for eventual deletion once downstream callers no longer need it. Users should migrate before it disappears from the interface.

12. **Threat model and trust boundaries**

    12.1. **Compile-time tool.** The compiler is intended to run in a trusted build environment on policy files that are reviewed or generated by trusted automation. Compromise of the JSON source or the host toolchain can produce arbitrary BPF; this component does not sandbox the policy author.

    12.2. **Native library and FFI.** Calls into the seccomp library are unsafe in Rust terms because they are extern boundaries. Correctness relies on valid handles, valid enumerants, and lifetimes that keep the filter context alive through export. A buggy or compromised native library is outside this crate’s control but would affect emitted BPF.

    12.3. **Artifact at rest and on load.** The binary artifact should be treated as trusted input to the embedding runtime only after integrity checks appropriate to the deployment (image signing, build provenance). Without the decode size limit, a malicious file could attempt denial-of-service via huge claimed lengths; the limit bounds allocation during deserialization.

    12.4. **Policy mistakes vs. implementation bugs.** A policy that is syntactically valid but overly permissive or overly strict is an operational risk, not something the compiler can fully detect. The tool fails fast on malformed JSON or resolver errors; it does not prove security properties of the policy.

    12.5. **Kernel and installation.** Installing BPF still requires appropriate privileges and correct use of kernel APIs in the runtime. This crate only produces instruction bytes; it does not enforce that the right filter is attached to the right thread.

13. **Command-line interface**

    13.1. The binary front end accepts a required target architecture string, a required policy input location, an optional output location defaulting to a conventional filename, and the deprecated basic flag described above. It delegates directly to the library compilation routine and propagates errors as process exit status via the error type’s display formatting chain.

14. **Error taxonomy and failure philosophy**

    14.1. Errors are classified for observability: file open/read, JSON parse, unknown architecture string, native library failures at each phase (context, architecture, syscall resolution, rule installation, BPF export), memory-backed descriptor lifecycle failures, output file creation, and bincode encoding. There is no partial output on failure: a problem in one named filter aborts the whole compilation.

    14.2. The design favors fail-fast semantics appropriate for build-time tooling: mis-specified policies or environment issues surface immediately rather than silently producing empty or partial artifacts.

15. **Safety and process assumptions**

    15.1. Numerous calls into the native library carry the usual FFI assumptions: valid pointers, correct enum values, and matching lifetimes (the filter context remains valid until export completes). The memory-backed descriptor is owned and closed via the file wrapper.

    15.2. Syscall name strings are carried as owned nul-terminated C strings in the deserialized types so they can be passed across the FFI boundary without additional allocation beyond what the deserializer already produced.

16. **Conceptual data-flow diagram**

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
   | native library:  | --> | add arch token   |
   | init + default   |     | (x86_64/AArch64)|
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
   | memory-backed fd |
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

17. **Relationship to runtime consumers**

    17.1. The artifact is not itself an installed seccomp policy; it is a portable bundle of precompiled BPF programs keyed by name. A runtime must select the correct entry, install it with the appropriate privileged kernel APIs, and honor the same architectural assumptions used at compile time. The deserialization configuration exposed by the library is the contract for that loading path.

    17.2. Because policies are compiled ahead of time, changes to JSON require recompilation to take effect; there is no interpreter in this crate. That trade-off favors auditability and minimizes attack surface on the virtual machine monitor’s hot path.
