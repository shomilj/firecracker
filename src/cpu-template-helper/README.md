# CPU template helper

1. **Purpose and operational model**

   1.1. A standalone command-line companion for authoring, deduplicating, validating, and fingerprinting *custom CPU templates* used with a Firecracker-style microVM stack. It reuses the same CPU configuration types and the same boot-time monitor construction path as the main hypervisor, so exported JSON reflects what the VMM would actually apply—not a parallel model.

   1.2. Commands fall into two families: *template* (dump effective guest CPU state as template-shaped JSON, strip redundancy across several template documents, verify that a template’s assertions hold against a live export) and *fingerprint* (record host context plus the same guest CPU snapshot in one JSON artifact, then compare two artifacts with per-field filters).

   1.3. Success exits zero; failures write a concise message to standard error and exit non-zero. Outputs are pretty-printed JSON for review, diffing, and CI gates.

2. **Guest construction: how every “live” pipeline is anchored**

   2.1. Dump, verify, and fingerprint dump all need a constructed monitor so the stack can call into the same export hook the product uses. Construction begins by turning a JSON machine description into resource bundles, optionally overlaying a deserialized custom CPU template so the effective template is “what the config says” plus any explicit template file passed on the command line.

   2.2. When no configuration text is supplied, the tool synthesizes a minimal valid description: a tiny kernel image is materialized from bytes embedded at build time, paired with an empty block device as root, and referenced from generated JSON. On 64-bit x86 the embedded image is a statically linked stub produced by compiling a minimal C program with a freestanding link (no standard libraries, no unwind tables, symbols stripped) so the blob is small and deterministic. On 64-bit AArch64 the embedded artifact is not a runnable loop but a correctly formed boot image header matching the platform’s documented layout (magic and reserved fields) so the loader accepts it; only the header bytes are required for the helper’s purposes.

   2.3. When configuration text *is* supplied, it is parsed with the same resource loader and the same maximum payload size bound as the HTTP-facing API, so pathological inputs are rejected consistently with production.

   2.4. Instance metadata is filled with the helper’s package version as both the reported VMM version and application name, and an anonymous instance identifier. An event loop handle is created, seccomp is explicitly set to empty filters (the tool is a short-lived utility, not a sandboxed long-running service), and the boot-oriented builder runs to obtain a locked monitor handle and the resolved machine resources.

   2.5. Nothing in this path requires a meaningful guest workload: only enough of the machine exists for CPU state to be queried. That is why CI and laptops can run dump/verify/fingerprint without real kernels or disk images when the default synthesized config is used.

3. **Template dump pipeline**

   3.1. The monitor is locked and asked to export CPU configuration for all virtual CPUs. The helper then selects the *first* exported snapshot as representative. That choice matches the common case (homogeneous vCPUs) and keeps output stable and small.

   3.2. The raw snapshot is architecture-specific. On x86 it pairs a normalized CPUID table with a sorted map of model-specific register indices to 64-bit values. On AArch64 it is the guest register vector exposed through KVM for the vCPU.

   3.3. Conversion to the JSON template schema is mechanical: each logical unit in the snapshot becomes a modifier entry. Modifiers always pair a value with a bitmask (“filter”): comparisons and verification apply `(value & mask)` on both sides, so templates can assert only the bits they care about.

   3.4. **x86 shaping.** Each CPUID leaf in the normalized table becomes one leaf object carrying leaf number, subleaf, KVM leaf flags, and up to four register modifiers (the four 32-bit result registers). Leaf iteration order follows the underlying map; after flattening for algorithms, round-tripping sorts by leaf, subleaf, then register name for deterministic JSON.

   3.5. **x86 MSR filtering.** Before serialization, MSRs are filtered to drop ranges that are time-varying (timestamp and deadline counters), machine-check and PMU-related ranges that the product stack does not meaningfully expose, PEBS and related sampling support that Firecracker does not enable, and AMD performance-counter MSRs in analogous ranges. The goal is to keep dumps stable across runs and comparable across software versions without chasing KVM default churn for features the guest cannot use. On AMD hosts, one additional architectural-capability MSR is removed from dumps because the product hides it via CPUID policy even though KVM may emulate it on Intel-class systems—emitting it would add platform noise. Remaining MSRs are sorted by index.

   3.6. **AArch64 shaping.** Each exported register is classified by width. Widths of 32, 64, or 128 bits become modifiers with values widened into a fixed unsigned representation; wider KVM encodings are skipped with a warning because the template format does not represent them. Timer-related system registers and the program counter are excluded: timers drift with time, and the PC is determined by the loaded kernel image rather than a portable CPU model. The remaining modifiers sort by register id.

4. **Template strip pipeline**

   4.1. Strip accepts *two or more* template documents and emits one output per input. The goal is to delete modifiers—or narrow bit masks—where every input agrees, leaving only per-file deltas. Typical uses include maintaining several CPU model variants that share a large common baseline.

   4.2. **Normalization.** Each document is converted into one hash map per modifier family. On x86 there are two families: CPUID register bits (keys include leaf, subleaf, KVM flags, and which of the four result registers) and MSRs (keys are numeric indices). On AArch64 there is a single family keyed by the numeric register id. Values are always register value filters (value plus mask).

   4.3. **Core algorithm (architecture-agnostic).** The implementation seeds a working “reference” map from the *first* input. For each key present in that first map, every other map must contain the same key; if any map lacks it, the key is discarded from the reference entirely—partial overlap does not count as common. For surviving keys, the algorithm accumulates a per-bit difference mask by comparing every other map’s *masked* value to the first map’s *masked* value at that key, combining those XORs with bitwise OR so any bit that differs from the first map in any input is marked.

   4.4. **Subtraction phase.** For each key still in the reference, if the difference mask is zero, the masked values matched the first map across all inputs, so the key is removed from *every* input map. If the mask is nonzero, each input keeps the key but replaces its filter with the bitwise AND of its previous filter and the difference mask—only bits that actually varied (relative to the first file, and among files that had the key) remain targeted.

   4.5. **Ordering.** Maps convert back to the nested vector form with deterministic sorting: x86 CPUID leaves sorted by leaf and subleaf, registers within a leaf sorted by name; MSRs and AArch64 registers sorted by id.

   4.6. **Interaction with verify and fingerprint compare.** The same strip routine is reused when fingerprint compare needs a human-sized diff for guest CPU configuration: two full templates that differ are both normalized and stripped as a pair so the report shows minimized differing bits rather than two complete documents.

5. **Template verify pipeline**

   5.1. Verify builds a monitor from configuration plus optional explicit template file, then resolves “the template under test” from machine resources—the same object dump uses when merging config and CLI overrides. In parallel it runs the dump pipeline to obtain the *effective* guest CPU configuration after everything the VMM applied.

   5.2. The check is **template-driven**: for every modifier in the template, there must be a matching key in the exported configuration map, and the template’s masked expectation must equal the configuration’s masked value under the template’s mask. Extra leaves, MSRs, or registers present in the export but not listed in the template are **not** failures; templates may deliberately specify only a subset.

   5.3. **Per-architecture maps.** On x86, CPUID and MSR families are verified separately with the same flattened keys as strip. On AArch64, a single register map is verified.

   5.4. **Mismatch diagnostics.** When a key is missing, the error names the key in the same human-readable form strip uses (leaf/subleaf/flags/register on x86, numeric id on AArch64). When values disagree, the error includes a three-line bit-oriented display: binary for the template side, binary for the configuration side, and a row of carets under bit positions that differ—MSB to LSB—so off-by-one mask mistakes are obvious.

6. **Fingerprint dump pipeline**

   6.1. Fingerprint dump reuses the full template dump to populate the guest CPU section, so nested JSON matches template dump output for the same machine state.

   6.2. Host metadata is collected alongside: the helper’s package version (standing in for tool lineage), the running kernel release from the standard system call that fills a `utsname` structure, a CPU revision string from sysfs (x86 reads the per-logical-cpu microcode version node under the CPU device tree; AArch64 reads the CPU identification revision register exposed in sysfs), and DMI-derived firmware strings for BIOS version and BIOS release from the virtual DMI sysfs tree. Trailing newlines are stripped from sysfs reads so string equality is stable.

   6.3. **Failure modes.** If sysfs nodes are unreadable in a given environment, dump may fail with the underlying I/O error; kernel version retrieval fails if the system call errors. This is intentional: partial fingerprints would silently omit host context.

7. **Fingerprint compare: semantics and filters**

   7.1. Compare loads two JSON documents into a strongly typed structure. The caller supplies one or more *field selectors*; default behavior selects every top-level field.

   7.2. **Scalar fields** (tool version, kernel string, microcode/revision string, both firmware strings) use strict string equality. Any inequality for a selected field produces one JSON object recording the field label, previous value, and current value. Multiple differences concatenate into the error text.

   7.3. **Guest CPU configuration field.** The embedded template is compared as structured data first. If the two deserialized snapshots are deeply equal, no diff object is emitted for that field. If they differ, the pair is passed through the strip pipeline (exactly two inputs, so strip always succeeds) and the diff object’s “previous” and “current” values are the *stripped* templates, minimizing noise to differing bits and keys.

   7.4. **Exit status.** If every selected field matches, compare succeeds with no standard output. If any selected field differs, compare fails and prints the concatenated pretty-printed diff objects.

   7.5. **Filter ergonomics.** Selecting only host fields ignores guest CPU differences; selecting only guest CPU ignores host drift. This supports workflows like “same firmware and kernel, did the guest CPU view change?” versus “did our BIOS update without touching CPU?”

8. **x86 versus AArch64: structural differences**

   8.1. **Modifier families.** x86 templates expose CPUID and MSR modifier lists; AArch64 exposes a single register modifier list keyed by KVM’s 64-bit register ids. Strip and verify run one or two map passes accordingly; fingerprint compare’s guest section always feeds the same strip implementation compiled for the host architecture.

   8.2. **Key richness.** x86 CPUID keys are four-dimensional (leaf, subleaf, flags, register slot), reflecting how KVM carries per-leaf metadata. AArch64 keys are a single opaque id with a uniform 128-bit value width in the template format.

   8.3. **Noise sources addressed.** x86 dumps apply a large MSR exclusion list and a vendor-specific extra exclusion; AArch64 drops timer and PC registers and skips unsupported widths. Both aim at comparable, policy-aligned snapshots rather than raw hardware trivia.

9. **Cross-cutting: value filters and bit display**

   9.1. Throughout dump, strip, verify, and fingerprint guest sections, numeric types implement a small display trait used only for mismatch lines: bits render from most significant to least, with spaces where the two masked values agree and carets where they diverge. The number of bits in the display matches the type’s width (32 for x86 CPUID registers, 64 for MSRs, 128 for AArch64 register modifiers).

10. **End-to-end views**

10.1. **Template operations overview**

```
  +------------------+     optional JSON machine config
  | CLI: template    |     optional explicit template JSON
  |   dump|verify    |------------------------------------+
  +--------+---------+                                    |
           |                                               v
           |                          +--------------------+--------------------+
           |                          | Parse resources, merge template,      |
           |                          | build monitor (empty seccomp),        |
           |                          | anonymous instance metadata             |
           |                          +--------------------+--------------------+
           |                                               |
           |                         +---------------------+---------------------+
           |                         |                     |                     |
           v                         v                     v                     v
  +------------------+    +-------------------+   +------------------+   +------------------+
  | template strip   |    | export vCPU0      |   | load template    |   | (verify only)    |
  | (no monitor;     |    | CPU config ->     |   | from resources   |   | compare template |
  |  reads N JSON    |    | template JSON     |   | + dumped config  |   | vs dump maps     |
  |  files only)     |    +-------------------+   +--------+---------+   +---------+--------+
  +------------------+             |                        |                        |
                                   v                        v                        v
                          write pretty JSON          pretty JSON              stderr on mismatch
```

10.2. **Fingerprint operations overview**

```
  +----------------------+
  | CLI: fingerprint     |
  |   dump | compare     |
  +----------+-----------+
             |
     +-------+--------+
     |                |
     v                v
+------------+   +------------------+
| dump path  |   | compare path     |
| monitor +  |   | read two JSON    |
| host probe |   | per-field filter |
+------+-----+   +---------+--------+
       |                   |
       v                   v
+------------+   +---------------------------+
| uname +    |   | scalars: string compare   |
| sysfs DMI  |   | guest: optional strip of  |
| + guest    |   |   pair if JSON differs    |
|   dump     |   +---------------------------+
+------------+
```

10.3. **Verify data flow**

```
  machine JSON ----------> resources + merged template intent
        |                          |
        |                          v
        +----------------> boot-time monitor build
                                    |
                                    v
                          export CPU configuration
                                    |
                                    v
                          template-shaped "actual"
                                    |
                                    v
               template JSON ------> per-key masked equality
               (expected)              (template filters applied
                                        to both sides)
```

10.4. **Strip conceptual model (three inputs, one key)**

```
  File A   File B   File C          Reference taken from A.
  key K    key K    key K           XOR/OR pass marks bits where B or C
  value a  value a  value a'        differs from A’s masked value at K.

  If all three masked values equal: difference mask = 0 -> key removed everywhere.

  If C differs in some bits: mask nonzero -> each file keeps K but
  new_mask = old_mask & difference_mask, so shared agreeing bits drop out
  of the maintained template noise.
```

11. **Build-time and testing notes**

   11.1. The build script for the mock kernel reruns when the tiny C source changes on x86; on AArch64 it writes the header blob when invoked. Unsupported host architectures fail fast at build time.

   11.2. Automated tests exercise the strip core on synthetic maps, verify error formatting, architecture-specific conversions with crafted inputs, sysfs helpers where the environment allows, and CLI smoke tests that run dump, strip, verify, and fingerprint subcommands against temporary files.
