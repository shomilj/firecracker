# CPU template helper

1. **Purpose and operational model**

   1.1. The helper is a standalone command-line tool that supports authoring, deduplicating, validating, and fingerprinting *custom CPU templates* for a microVM stack. It reuses the same CPU configuration types and VMM construction logic as the main hypervisor, so what it dumps, strips, or verifies is semantically aligned with what the VMM would apply at boot—not an independent approximation.

   1.2. Work is organized into two command families: *template* operations (dump the effective guest CPU configuration as template-shaped data, strip redundancy across multiple template documents, verify that a template’s intent matches the configuration the VMM actually materialized) and *fingerprint* operations (capture host plus guest-CPU state into a single JSON artifact, and compare two such artifacts with selectable fields).

   1.3. On success the process exits with a zero status; failures print a human-readable error to standard error and exit non-zero. Output artifacts are pretty-printed JSON for review and diffing in version control or CI.

2. **Relationship to the VMM and the “microVM build”**

   2.1. Nearly every substantive operation begins by constructing a virtual machine monitor instance and associated machine resource bundle from a Firecracker-style JSON configuration, optionally augmented with a deserialized custom CPU template. The construction path is the same *boot-oriented* builder used elsewhere: an event loop handle is created, seccomp filters are explicitly set to an empty set (the tool is not a long-running sandboxed service), and the microVM is prepared *without* requiring a full guest workload—only enough structure exists for CPU state to be introspected.

   2.2. When the operator does not supply a configuration path, the tool synthesizes a minimal valid configuration: a tiny kernel image is materialized from bytes embedded at build time, paired with an empty block device acting as root, and referenced from generated JSON. On one architecture the embedded kernel is a statically linked, minimal stub that spins forever; on another it is a header-sized blob satisfying the platform’s expected boot image layout. This keeps template and fingerprint workflows usable in CI or developer machines without mandating real guest images.

   2.3. When a configuration *is* supplied, it is read as text and parsed through the same resource loader as production, including payload size limits carried over from the HTTP API defaults. An optional template path is merged into machine resources so the effective template is whatever the configuration implies, optionally overridden by the explicit template file.

   2.4. The tool’s own version string is threaded into instance metadata as the VMM version and application name, so diagnostics and fingerprints can be tied to the helper build.

3. **Template dump: from live CPU state to template-shaped JSON**

   3.1. The dump path locks the constructed monitor, asks it to export the current CPU configuration for all vCPUs, and takes the first CPU’s snapshot as representative. That snapshot is an architecture-specific CPU configuration snapshot: on Intel/AMD-style hosts it comprises a normalized CPUID table and a map of MSR index to 64-bit value; on AArch64 hosts it comprises the guest register vector exposed by KVM for the vCPU.

   3.2. The conversion into the JSON template schema is intentionally mechanical: every exported leaf/subleaf/register or MSR index becomes a modifier entry whose “value and mask” semantics match the template format (bitmaps describe which bits are relevant and what values they should take after masking).

   3.3. **x86_64-specific shaping.** CPUID entries are emitted as one leaf modifier per `(leaf, subleaf, flags)` group, with up to four register modifiers inside (EAX/EBX/ECX/EDX), preserving KVM’s per-leaf flags. The MSR list is filtered before serialization: ranges associated with time-varying counters, machine-check and PMU-related MSRs that the VMM does not meaningfully expose, and other high-churn or unsupported values are dropped so dumps focus on stable, comparable state. On AMD hosts an additional exclusion applies to an architecture-capability MSR that KVM emulates on Intel-class systems but which the product stack deliberately hides on AMD via CPUID policy—dumping it would add noise. Remaining MSRs are sorted by index for deterministic output.

   3.4. **AArch64-specific shaping.** Registers are converted according to their encoded width (32, 64, or 128 bits); wider encodings are skipped with a warning because the template format does not model them. Timer-related system registers and the program counter are excluded: the former fluctuate with time, the latter is determined by the loaded kernel image rather than a portable “CPU model” concern. The remainder are sorted by register id.

4. **Template strip: factoring out what templates agree on**

   4.1. Strip takes *two or more* template documents and produces one output per input. The goal is to remove modifiers (or portions of modifiers) that are identical across *all* inputs, leaving only what differentiates each file—useful when several CPU models share a large common baseline and only a small delta should be maintained per variant.

   4.2. Internally each template is normalized into one hash map per modifier family (on x86, separate maps for CPUID register bits and for MSRs; on AArch64, a single map for guest registers). Keys are fully qualified identities (for CPUID: leaf, subleaf, KVM flags, and which register within the leaf; for MSRs and AArch64 registers: the numeric index/id). Values are *register value filters*: a value plus a bitmask indicating which bits participate.

   4.3. The core algorithm walks the key set of the first map as a candidate “common” set. For each key present in the first map, every other map must also contain that key; if any map lacks it, the key is dropped from consideration entirely—partial overlap does not count as “common.” For surviving keys, the algorithm compares the *masked* values across maps (value AND mask). Bits that differ anywhere are accumulated into a difference mask. After this pass, the “common” map holds, for each key, the aggregate XOR of all pairwise masked differences.

   4.4. The second phase subtracts that common information from every input map. Where the common difference mask is zero for a key, the filtered values were identical across all inputs, so that key is removed from every map. Where the mask is nonzero, each map keeps the key but narrows each modifier’s filter to the intersection of its previous filter and the common difference mask—so only bits that actually vary across templates remain targeted. Intuitively: shared bits are stripped; disputed bits stay.

   4.5. The maps are converted back to the template vector representation with deterministic ordering (sorted leaves, sorted registers within a leaf, sorted MSR indices, sorted AArch64 register ids).

5. **Template verify: does the template match reality?**

   5.1. Verify combines three inputs in memory: the machine configuration (to recover which custom template the operator believes should apply), the explicit template file if provided (merged earlier), and a fresh dump of the effective guest CPU configuration using the same conversion as the dump command.

   5.2. The check is *template-driven*: every modifier that exists in the template must have a corresponding entry in the dumped configuration, and for each such entry the masked template value must equal the masked configuration value under the template’s mask. Extra state present in the dump but not covered by the template is not a failure—the template may intentionally specify only a subset of bits or leaves.

   5.3. The equality test uses the same hash-map representation as strip, so CPUID is compared per `(leaf, subleaf, flags, register)` tuple and MSRs or AArch64 registers per numeric id. Mismatches produce errors that include a bit-aligned ASCII visualization: three lines showing the template bits, the configuration bits, and a row of carets marking differing bit positions—making off-by-one bitmask authoring mistakes obvious.

6. **Fingerprint dump: anchoring templates to host context**

   6.1. A fingerprint extends the guest CPU template snapshot with host metadata so operators can tell whether a template was produced on a comparable software and firmware stack. Fields include: the helper’s version (as a stand-in for the tool lineage), the running kernel release from `uname`, a microcode or CPU revision string read from sysfs (the exact sysfs node depends on architecture—one path exposes x86 microcode version, another exposes AArch64 REVIDR), and DMI-derived BIOS version and BIOS release strings read from the virtual sysfs tree for firmware identification.

   6.2. Trailing newlines from sysfs reads are stripped for stable string equality. Guest CPU configuration inside the fingerprint is populated by the same dump pipeline as template dump, so the JSON nested under the host metadata is template-shaped and comparable to stripped template outputs.

7. **Fingerprint compare: diffing two snapshots with filters**

   7.1. Compare loads two JSON fingerprints, deserializes them into a strongly typed structure, and evaluates a caller-selected list of top-level fields (defaulting to *all* fields). For scalar host metadata fields the comparison is strict string inequality.

   7.2. For the embedded guest CPU configuration field, a naive inequality would be noisy when many bits are equal. If that field differs, the helper clones both guest configurations into a two-element list and runs the *same strip algorithm* used for multi-file template maintenance. Strip with two inputs removes identical masked values and leaves only differing bits per modifier. The resulting pair is what gets serialized into the diff report, so the operator sees a minimized delta rather than two full templates.

   7.3. If any compared field differs, the command fails and concatenates pretty-printed JSON diff objects (each records the field name, previous value, current value). If all selected fields match, it succeeds with no output.

8. **Cross-cutting design: modifier maps, bitmaps, and display**

   8.1. Throughout, CPUID on x86 is treated as a flat map keyed by the full qualifier (leaf, subleaf, flags, register) because the hypervisor’s leaf structures group registers together; flattening enables uniform algorithms. MSRs and AArch64 registers use a single numeric key with an opaque id formatting for error messages.

   8.2. Modifier entries pair a value with a bitmask (“filter”) in a consistent way: comparisons always apply bitwise AND of value and mask on both sides, so partial-bit templates only assert the bits they care about.

   8.3. A small trait layer abstracts how numeric types render bit-diff strings for error messages, walking bit positions from most to least significant and emitting spaces where bits match and carets where they differ.

9. **Build-time concerns**

   9.1. A build script regenerates the embedded mock kernel artifact whenever the tiny C source changes (on the architecture that compiles it), linking statically without unwind tables and stripping symbols for a minimal binary. On the other architecture it writes only the required boot header bytes. The build marks the output for rerun when inputs change so incremental builds stay correct.

10. **Testing strategy**

    10.1. Unit tests cover the strip core on synthetic small maps, verify behavior for missing keys and mismatched bits, architecture-specific conversion from synthetic CPU configurations, fingerprint sysfs helpers where feasible, and CLI-level smoke tests that exercise dump/strip/verify/fingerprint commands with temporary files.

11. **Architecture overview diagram**

```
                    +-------------------+
                    | CLI (template /   |
                    | fingerprint cmds) |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | Load JSON config  |
                    | + optional        |
                    |   template JSON   |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | Build monitor +   |
                    | machine resources |
                    | (empty seccomp)   |
                    +---------+---------+
                              |
          +-------------------+-------------------+
          |                                       |
          v                                       v
 +----------------+                    +----------------------+
 | export CPU     |                    | fingerprint extras   |
 | config -> tmpl |                    | (uname, sysfs, DMI)  |
 |   conversion   |                    +----------+-----------+
 +--------+-------+                               |
          |                                       v
          |                              +--------------------+
          |                              | merge into         |
          |                              | fingerprint JSON   |
          |                              +--------------------+
          v
 +------------------+
 | strip / verify / |
 | compare paths    |
 +------------------+
```

12. **Data-flow diagram (template verify)**

```
  Firecracker JSON ------> machine resources ------> merged custom template
        |                                                   |
        |                                                   |
        v                                                   v
   boot-time microVM build -----------------------> export CPU configuration
                                                        |
                                                        v
                                                 template-shaped dump
                                                        |
                                                        v
                    verify: template masked values  ==  dump masked values
                            (per CPUID / MSR / reg map)
```
