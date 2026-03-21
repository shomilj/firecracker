# Architecture: Offline CPU Template and Fingerprint Helper

## Purpose & Boundaries

This system is a standalone command-line utility that operates entirely offline with respect to running guest workloads. Its responsibility is to help authors and operators work with custom CPU templates for a lightweight virtual machine monitor: producing human-readable snapshots of effective guest CPU configuration, deduplicating repeated modifier entries across multiple template documents, checking that a template’s declared intent matches what the stack actually applies, and capturing or comparing environment fingerprints that bind host characteristics to guest CPU state. The tool is intentionally narrow. It does not start long-lived services, manage disk images beyond the minimal scaffolding needed to construct an in-process micro virtual machine, enforce cloud deployment policy, or replace the hypervisor’s own validation. It assumes the surrounding monitor implementation is the source of truth for how templates are interpreted.

If this component were unavailable or broken, template authoring workflows would lose a repeatable way to diff models, strip boilerplate, or prove parity between template files and observed guest state. Continuous integration that relies on deterministic dumps or fingerprint comparisons would fail to signal regressions in CPU normalization. The runtime hypervisor would continue to boot guests, but human and automated processes that depend on reproducible artifacts would be degraded rather than the core boot path itself.

What stays explicitly outside this utility includes live migration orchestration, performance tuning of workloads, and any network-facing control plane. The helper consumes the same configuration and template JSON shapes as the production stack but only to drive an isolated construction path suitable for inspection.

---

## Interfaces & Contracts

The outward surface is a hierarchical command structure with two top-level groups. One group addresses template-oriented operations: emitting a serialized guest CPU configuration, removing entries that are identical across several inputs, and verifying that a loaded template matches the configuration derived from a freshly built micro virtual machine. The other group addresses fingerprints: emitting a JSON document that merges host metadata with the guest CPU snapshot, and comparing two such documents with selectable field filters.

Inputs include optional paths to a Firecracker-style instance configuration, an optional path to a custom CPU template JSON file, and paths for outputs or comparison endpoints. When no instance configuration is supplied, the utility synthesizes a minimal valid configuration by pairing an empty root filesystem with a tiny kernel image. On one architecture the kernel image is produced at build time by compiling a minimal stub; on another architecture a preformatted header blob is written to satisfy boot expectations without executing a host compiler. Callers must supply well-formed JSON where they provide files; malformed documents surface as deserialization failures. The tool promises to write pretty-printed JSON for dumps and to exit with a non-zero status and a message on stderr when any operation fails.

Internally, the helper depends on the project’s virtual machine monitor crate for resource parsing, micro virtual machine construction, CPU configuration dumping, and the template data model. It uses the standard filesystem, kernel version discovery through the operating system interface, and selected sysfs nodes for firmware and microcode identifiers. A small set of cross-cutting utilities defines how architecture-specific modifier lists flatten into associative structures for comparison and stripping, and how bitwise differences are rendered when verification fails.

Callers should treat outputs as diagnostic artifacts, not as secrets. Fingerprints embed host software and hardware version strings read from the local machine.

---

## Data Flow

Understanding this system begins with what crosses its boundary. Configuration text enters as UTF-8 JSON describing boot source, drives, and machine parameters. Template JSON enters when the user wants modifiers applied before the micro virtual machine is built. Optional inputs are combined: absent user configuration, synthetic boot artifacts are materialized as temporary files whose paths are embedded into generated configuration text.

The next transformation constructs in-memory resource descriptors and then builds a micro virtual machine ready for boot without requiring a full guest operating system workflow. From that live object, the monitor exposes a dump of CPU state—CPUID leaves and model-specific registers on one architecture, system register snapshots on another. The dump path converts that low-level configuration into the same high-level custom template shape used for authoring, applying architecture-specific filtering to drop volatile counters, unsupported performance facilities, and other entries known to add noise or vary with time.

The strip path ingests multiple template documents, maps each into parallel associative views of modifiers, and computes per-bit differences across inputs so that shared stable bits can be removed from every file while preserving bits that genuinely differ. The result is written alongside originals according to a configurable filename suffix, including the option to overwrite in place when an empty suffix is chosen.

The verify path loads the template referenced by machine resources, dumps the effective guest configuration through the same conversion used for authoring, and compares the template’s declared masks and values against the dump using the template’s own filter bits as the equality domain. Mismatches produce textual diffs with bitwise annotations.

Fingerprint dump extends the CPU snapshot with strings gathered from the running kernel’s version, sysfs-exposed firmware identifiers, and an architecture-specific sysfs location for microcode or processor revision data. Fingerprint compare deserializes two JSON documents, applies user-selected field filters, and for the guest CPU portion may run the strip logic pairwise to highlight only divergent modifier bits when the high-level structures differ.

The following diagram summarizes end-to-end movement of data for the primary workflows.

```
┌─────────────────┐     optional JSON      ┌──────────────────────────┐
│ User-supplied   │ ──────────────────────► │ Parsed machine resources │
│ config file     │                        │ and optional custom      │
└─────────────────┘                        │ CPU template             │
        │                                  └────────────┬─────────────┘
        │ optional absence                             │
        ▼                                              ▼
┌─────────────────┐                        ┌──────────────────────────┐
│ Synthetic kernel│ ──embed paths────────► │ Micro virtual machine    │
│ and empty rootfs│                        │ construction (in-process)│
└─────────────────┘                        └────────────┬─────────────┘
                                                      │
                     ┌────────────────────────────────┼────────────────────────────┐
                     │                                │                            │
                     ▼                                ▼                            ▼
            ┌────────────────┐              ┌─────────────────┐          ┌──────────────────┐
            │ Raw CPU config │              │ Template JSON   │          │ Host metadata via  │
            │ dump from VMM  │              │ in/out for strip│          │ kernel API and     │
            └────────┬───────┘              └────────┬────────┘          │ sysfs reads        │
                     │                               │                   └─────────┬────────┘
                     ▼                               ▼                             │
            ┌────────────────┐              ┌─────────────────┐                 │
            │ Convert + filter │              │ Bitwise strip   │                 │
            │ to template form│              │ across N files  │                 │
            └────────┬───────┘              └────────┬────────┘                 │
                     │                               │                           │
                     └───────────────┬───────────────┴───────────────────────────┘
                                     ▼
                          ┌─────────────────────┐
                          │ JSON on disk or      │
                          │ stderr diff messages │
                          └─────────────────────┘
```

Before the diagram, the key observation is that all heavy lifting flows through a single construction step that unifies optional external configuration with a safe default. After the diagram, the important consequence is that every downstream artifact—plain template dumps, verification, and fingerprints—derives from the same conversion and filtering philosophy, which keeps semantics aligned across commands.

---

## Control Flow

Execution is strictly request-driven from the command line. Parsing happens once at startup; there is no interactive loop or daemon behavior. The critical path for template dumping reads inputs, builds the micro virtual machine, locks the monitor object long enough to obtain the CPU dump, converts and filters, serializes JSON, and writes the output file. Strip skips micro virtual machine construction entirely: it only reads template files, transforms them, and writes outputs. Verify combines construction with a second retrieval of the resolved template from machine resources, then runs parallel comparisons for each architecture’s modifier families.

Fingerprint dumping follows the template dump path and then enriches the result with host queries. Fingerprint comparison is fully offline: it only touches the filesystem to read two JSON files and performs field-wise logic in memory.

Branching is minimal but meaningful. Absent versus present user configuration chooses between synthetic boot assets and user-provided paths. Architecture-specific code behind compile-time configuration selects register versus CPUID and model-specific register handling. Verification branches on whether each template key exists in the dumped configuration and whether filtered values match. Strip refuses to run with fewer than two inputs because the algorithm is defined as removing intersections across a multiset of documents.

A compact view of the command routing illustrates how operations partition without implying implementation detail.

```
                    ┌──────────────────────┐
                    │  CLI invocation      │
                    └──────────┬───────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
    ┌──────────────────┐              ┌──────────────────┐
    │ Template branch  │              │ Fingerprint      │
    │                  │              │ branch           │
    └────────┬─────────┘              └────────┬─────────┘
             │                                  │
     ┌───────┼───────┐                   ┌──────┴──────┐
     │       │       │                   │             │
     ▼       ▼       ▼                   ▼             ▼
  dump    strip  verify              dump        compare
```

Prose before: the tree shows that template and fingerprint concerns share infrastructure where guest CPU state is involved but diverge where host metadata or multi-file deduplication appears. Prose after: this separation keeps the critical path for offline diffing fast while still allowing heavier construction when fidelity to the live stack matters.

---

## State & Lifecycle

The process is short-lived. State is held in heap-allocated structures for the duration of a single invocation. The micro virtual machine object and its event infrastructure exist to satisfy construction APIs; they are not repurposed as a long-running service. Temporary kernel and rootfs files created for the default configuration live for the scope of the function that prepares them and disappear when their handles drop.

There is no persistent cache between runs. Each execution re-reads inputs from disk and re-queries the kernel for version information during fingerprint generation. Steady-state operation is therefore identical to startup: there is no warm pool or background reconciliation. Shutdown is implicit process exit. Recovery from partial writes is not specialized: failed writes surface as I/O errors, and users rerun the command.

The nearest thing to a lifecycle diagram is the implicit state of a single run, moving from argument validation through optional file reads, optional synthetic asset creation, construction, transformation, and output.

```
  START
    │
    ▼
  Parse CLI
    │
    ▼
  Load inputs ──error──► EXIT(failure)
    │
    ▼
  [Optional] Create temp boot files
    │
    ▼
  Build microVM ──error──► EXIT(failure)
    │
    ▼
  Execute sub-operation (dump / strip / verify / fingerprint)
    │
    ▼
  Write outputs / print diffs ──error──► EXIT(failure)
    │
    ▼
  EXIT(success)
```

Narrative before: this linear lifecycle underscores that there is no reconciliation loop or retry policy inside the tool. Narrative after: operators who need retries or orchestration should wrap the binary externally.

---

## Failure Modes

Failures are intended to be loud and localized. File read and write problems propagate as errors with standard I/O context. JSON parsing errors indicate corrupt inputs. Template resolution failures occur when machine resources do not specify a template in contexts that require one. Construction failures bubble up from the monitor when configuration is inconsistent with supported features or when the builder cannot initialize devices.

Verification distinguishes missing keys from value mismatches. A missing key means the template declares a modifier entry that the dumped configuration does not expose under the same associative keying scheme, which should be rare if both sides use the same conversion. A value mismatch means the filtered template bits differ from filtered values observed in the guest snapshot; the tool prints bitwise diff lines to localize offending bits.

Fingerprint comparison aggregates differences per selected field. If nothing differs under the chosen filters, the command succeeds silently. Any difference produces a consolidated error string embedding pretty-printed JSON fragments for inspection.

Silent corruption is not a stated goal: the tool does not attempt to continue after partial failures. The main assumption that, if violated, could yield misleading confidence is comparing fingerprints generated on different architectures or with incompatible helper versions: the document includes a packaged version string for the helper itself to make such drift visible when users compare across time.

---

## Operational Characteristics

Resource usage follows a burst pattern. Each invocation loads linked libraries, parses JSON, and may construct a micro virtual machine in-process. Memory spikes are bounded by that single object graph plus serialized JSON buffers. CPU time is dominated by construction and, on fingerprint dump, a handful of sysfs reads and a system call to query the kernel version.

Scaling bottlenecks are not throughput-related: the utility is single-threaded and intended for developer machines and CI agents running occasional checks. Parallelism, if needed, must come from launching multiple processes with disjoint inputs.

Observability is minimal by design. User-visible feedback is stderr messages on failure and the absence of output on success for comparisons. There is no structured logging channel in the default feature set; an optional tracing feature may exist when compiled with additional instrumentation, but the baseline expectation is console-only diagnostics.

Build-time behavior differs by host architecture: one platform invokes a system compiler to produce a tiny binary kernel image consumed as bytes; another synthesizes a header directly. Build failures therefore depend on toolchain availability on the first platform.

---

## Design Rationale

The central design choice is to reuse the production monitor’s configuration and construction pipeline rather than reimplementing CPU semantics in the helper. That coupling trades independence for fidelity: the dumped guest configuration is exactly what the stack would derive internally, modulo explicit filtering to remove volatile or irrelevant registers. Authors gain confidence that template JSON round-trips through the same code paths used at runtime.

Strip encodes a specific maintenance workflow: when several template files inherit a large common baseline, storing the intersection once and deleting duplicates reduces merge noise. The bitwise algorithm preserves differences where templates diverge and encodes those differences in the filter masks, aligning with how verification interprets masks as the meaningful comparison domain.

Fingerprinting acknowledges that guest CPU state alone is insufficient to reason about portability. Kernel, firmware, and microcode revisions can change normalization behavior even when template files stay byte-identical. Bundling those host signals with the guest snapshot creates a single artifact suitable for longitudinal comparison, while field filters let operators focus on the dimensions they care about when triaging drift.

The synthetic boot path exists to lower friction: users who only need a CPU snapshot should not be forced to craft a full instance configuration. The tradeoff is embedding small binary assets and temporary files, which is acceptable for a developer tool and unacceptable for a hardened appliance without review.

Finally, strict failure reporting and bitwise diff output favor diagnosability over silent fixes. The tool assumes a knowledgeable operator who can iterate on templates when verification fails, which matches its position in the authoring toolchain rather than in the hot path of guest serving.

---

## Summary Topology

The following high-level picture situates the helper between human workflows and the embedded monitor without naming internal units.

```
  Operator / CI job
         │
         │  commands + JSON files
         ▼
  ┌──────────────────────────────┐
  │ Offline helper process       │
  │  - parses CLI                │
  │ - loads / synthesizes config │
  │ - builds microVM via VMM API │
  │ - transforms & compares CPU  │
  │   templates                  │
  └──────────────┬───────────────┘
                 │
                 │ uses
                 ▼
  ┌──────────────────────────────┐
  │ Virtual machine monitor      │
  │ library (shared with         │
  │ production hypervisor)       │
  └──────────────────────────────┘
                 │
                 │ reads (fingerprint)
                 ▼
  ┌──────────────────────────────┐
  │ Local kernel interfaces and │
  │ sysfs metadata              │
  └──────────────────────────────┘
```

Text before the figure positions the helper as a thin orchestration layer over shared library logic and host introspection. Text after reinforces that its value is coordination and presentation, not reimplementation of virtualization mechanics.
