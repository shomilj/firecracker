# Guest CPU configuration and templates

## Purpose & Boundaries

This subsystem defines how a micro virtual machine’s processors are described to software running inside the guest. Its responsibility is to collect, reshape, and apply policy to the machine-specific details that guests discover through architected discovery mechanisms and through privileged registers. On the sixty-four-bit Intel-compatible path, that means the information historically exposed by the processor identification instruction and by model-specific registers that the hypervisor mirrors into the guest. On the sixty-four-bit Arm path, it means the feature bits and system register images that the virtual CPU exposes after initialization. The subsystem also provides named static profiles and user-supplied custom descriptions so that deployments can align guest-visible capability with expected hardware classes, for example cloud instance families, without rewriting the rest of the virtual machine monitor.

What this subsystem does not do is implement the actual execution of guest instructions, the KVM ioctl surface in full, or general device emulation. It does not choose scheduling or pinning for virtual CPUs. If it failed outright, guests would either not start or would see inconsistent or invalid processor identity and feature bits, which would break operating system feature detection, licensing checks that depend on reported model, security mitigations that key off specific capability bits, and any workload that assumes a particular cache topology or core count layout. The surrounding monitor would still exist, but configuring a runnable, trustworthy guest CPU view would be blocked or would mislead software in subtle ways.

## Interfaces & Contracts

The outward-facing contract is built around two complementary ideas: a declarative description of desired adjustments, and a resolved runtime configuration that can be applied to virtual CPUs. Declarative descriptions come in two flavors. Static profiles are small enumerated choices that expand, only on matching hosts, into a full set of adjustments maintained by the project. Custom descriptions are data-defined lists of adjustments validated when they are parsed, typically from a structured text interchange format. Both flavors can express optional extra checks against kernel virtualization capabilities.

The sixty-four-bit Intel-compatible path exposes a resolved configuration that pairs a normalized tree of identification and feature leaves with a map of model-specific register values keyed by address. Construction starts from what the host kernel advertises as supported for guests, then layers template adjustments. The Arm path exposes a vector of register images sized according to which architectural registers the template touches, after optional initialization-time feature words are applied. Template resolution turns an optional selection plus an optional custom blob into a single effective description; static choices may be rejected when the host vendor, processor family, or model does not match what the profile allows, which is an intentional guardrail so that instance-class emulation is not silently applied on the wrong silicon.

A central behavioral contract is that adjustments are bit selective rather than wholesale replacement. Each modification specifies which bits of a word may change and what values those bits take, while leaving unspecified positions untouched. This preserves host-provided defaults for bits the template does not care about and reduces the risk that a template accidentally strips required features. Another contract is that on the Intel-compatible path, vendor identity in the root identification leaf is refreshed from the host during normalization so that custom templates cannot forge a different vendor string; templates may still change many other leaves and registers where policy allows.

Callers must supply consistent virtual CPU counts and topology parameters where normalization needs them, and templates must only target leaves and registers that the host configuration already contains. The subsystem guarantees that unsupported targets result in explicit errors rather than silent omission on the Intel-compatible apply path, and that parsed custom descriptions reject obviously inconsistent register widths on Arm.

## Data Flow

Data enters from three coarse sources: the host kernel’s view of what can be exposed to guests, the chosen template or custom description, and live readbacks from the first virtual CPU used to seed model-specific state. On the Intel-compatible path, supported identification data is ingested and classified by vendor into one of two internal strategies, because the two major vendors diverge in extended leaves and in how cache and topology should be normalized. That classification drives later steps. Templates supply ordered lists of leaf targets, each with optional hypervisor-specific flags for whether sub-leaves are significant, plus per-register bit masks. Model-specific register changes are merged by address into the values read from the processor.

The following diagram sketches how these inputs converge into the resolved guest view.

```
  Host KVM supported ID tree ----+
                                 |
  First vCPU MSR readback -------+---> [Classify by vendor]
                                 |              |
  Template (static or custom) ---+              v
                                        [Apply bit filters to
                                         leaves and MSRs]
                                                 |
                                                 v
                                        Guest CPU configuration
                                        (ID tree + MSR map)
```

On Arm, the flow is flatter: feature words for initialization may be adjusted first, then architectural registers are read into a working set, then register-sized bit filters are applied in lockstep order. There is no vendor fork inside this subsystem for Arm comparable to the Intel-compatible split, because the discovery model is register-centric rather than tree-centric.

Transformed data exits toward components that program virtual CPUs, serialize state for migration snapshots, and present configuration back through management interfaces. Static profile selections collapse to a neutral sentinel in some API responses when the user supplied a fully custom description, which keeps older clients from misinterpreting snapshots; that behavior is a contract with the persistence layer rather than with the guest itself.

After the diagram, the important point is that nothing in this pipeline invents identification leaves from whole cloth for the Intel-compatible case: it rewrites and normalizes within the envelope the kernel already granted. That is why missing leaves or registers surface as errors during application rather than as automatic insertion.

## Control Flow

Execution is triggered when the monitor builds guest processor state for startup or when it reapplies policy after configuration changes. The critical path begins with resolving which template is effective. Absence of a selection yields an empty effective template that leaves normalization defaults unchanged apart from architecture-specific baseline behavior. A static selection performs host capability checks, then materializes the embedded profile when allowed. A custom selection bypasses host model gates but still goes through validation rules for the structured description.

On the Intel-compatible path, the next critical segment classifies the supported identification blob, then runs a normalization pass per virtual CPU index. Normalization adjusts topology-related fields, hypervisor presence bits, brand-related extended leaves, cache reporting leaves, and power or feature leaves according to vendor-specific tables. Only after that does the template apply phase merge user or static bitmaps into the working identification tree and model-specific register map. Branching is heavy along vendor lines and along whether certain extended leaves exist, because missing data is sometimes an error and sometimes a cue to skip an optional refinement.

Arm control flow initializes virtual CPUs when needed, reads the baseline register file, then applies register filters in template order. The sixty-four-bit Arm branch currently does not gate static profiles on detailed processor model identifiers the way the Intel-compatible branch does; a placeholder concern noted in implementation is that stricter matching could be added later.

A second control-flow diagram highlights the normalization versus template layering on Intel-compatible hosts.

```
                    +------------------+
                    | Per-vCPU index   |
                    +--------+---------+
                             |
                             v
              +------------------------------+
              | Vendor-neutral normalization |
              | (identity, topology, caches) |
              +--------------+---------------+
                             |
              +--------------+--------------+
              |                             |
              v                             v
     +----------------+             +----------------+
     | Intel-specific|             | AMD-specific   |
     | refinements   |             | refinements    |
     +--------+-------+             +--------+-------+
              |                             |
              +-------------+---------------+
                            |
                            v
                   +----------------+
                   | Template merge |
                   +----------------+
```

The branching point at vendor-specific refinements exists because the two vendors use different extended leaves for brand strings, cache topology, and core enumeration. Treating them identically would produce incorrect guest-visible cache geometry or inconsistent brand text.

## State & Lifecycle

Owned state is the resolved guest processor configuration object: on Intel-compatible hosts, the working identification structure plus model-specific register values; on Arm, the vector of register values carrying the identifiers and feature bits templates may alter. Before steady operation, state is initialized by reading host-supported capabilities and, where required, seeding model-specific registers from hardware readback. Static templates may also inject warnings when a particular security-related capability is not enumerated on the host, informing operators without blocking template selection in those cases.

During steady-state operation, this configuration is treated as immutable for a given boot once applied; changes require a new configuration cycle. Shutdown and migration interact with higher layers: only part of the template choice is always persisted historically, so restored machines may need compatible template definitions present at resume time. Recovery from partial application failures is straightforward because failed applies return errors before mutating the committed guest state; the monitor does not leave half-applied bitmaps in a nominal success path.

A simple lifecycle view:

```
  [Parse / resolve template]
            |
            v
  [Build baseline from KVM + hw]
            |
            v
  [Normalize per architecture]
            |
            v
  [Apply template bitmaps]
            |
            v
  [Hand off to vCPU programming]
```

The terminal state is not owned solely by this subsystem; virtual CPU threads and the kernel hold the authoritative registers during execution.

## Failure Modes

Failures fall into validation failures, capability mismatches, and missing substrate errors. Validation failures arise from malformed structured descriptions, bitmap strings that specify impossible bit patterns, or register targets whose width does not match the architectural register size. Capability mismatches include choosing a static profile designed for one processor vendor while running on another, or on the Intel-compatible path selecting a profile tied to specific silicon generations when the host reports a different family and model combination.

Missing substrate errors dominate the Intel-compatible apply path: a template that references an identification leaf the kernel did not expose, or a model-specific register the first virtual CPU could not read, causes an explicit error. That fail-fast behavior avoids silent divergence between what the operator requested and what the guest actually received. Normalization can fail when expected leaves disappear after kernel upgrades; such failures are distinct from template errors and indicate environment drift rather than user input mistakes.

Assumptions that could cause silent corruption if violated are largely delegated elsewhere: if a caller bypassed the supported-identification envelope and injected arbitrary leaves, guests could see impossible combinations. This subsystem assumes it is composed with KVM’s supported identification as the source of truth for leaf presence. On Arm, pairing register filters in the wrong order relative to the read vector would mis-apply values; the implementation ties order to template declaration order matched with the read sequence.

## Operational Characteristics

Work is mostly proportional to the size of the supported identification tree and the number of template entries, not to guest memory size. The expensive parts are normalization passes that walk multiple sub-leaves for topology and cache description, which scale with how many leaves the host exposes. Arm paths are lighter in comparison because they operate on explicitly listed register identifiers rather than scanning large tables.

Scaling bottlenecks are unlikely inside this module itself; the surrounding monitor’s virtual CPU count and kernel ioctl traffic dominate. Observability is primarily indirect: warnings can be emitted when static profiles imply security expectations the hardware does not meet. There is no dedicated metrics layer inside this package; operators infer health from startup success and from higher-level tests.

Fingerprinting, in the systems sense used here, refers to treating the guest-visible processor identity as a stable signature that can be compared against expected baselines over time. After normalization, brand strings, topology encodings, and selected feature bits form a coherent picture of what software running inside the guest believes about the machine. When the host kernel’s virtualization layer changes which sub-leaves it populates or how it flags indexed leaves, that signature can shift even if guest configuration files are unchanged. Automated comparisons between a captured signature and a stored baseline therefore act as regression detectors for KVM and kernel behavior, complementing unit-level checks. This concept aligns with why vendor identity is re-synchronized from the host and why topology leaves are rewritten carefully: the signature is meant to reflect policy and silicon class, not accidental drift from partially inherited host identification strings.

## Design Rationale

The design solves the problem of portable micro virtual machines that must still present believable, policy-aligned processor identities across heterogeneous hosts. Bit-filter templates trade verbosity for safety: they allow precise surgical changes without demanding that operators hand-craft entire identification tables. Separating normalization from template application separates concerns between what the monitor must do for every guest to keep topology and hypervisor bits coherent, and what optional policy layers add for cloud instance parity or experimentation.

Tradeoffs include maintaining parallel vendor-specific normalization logic, which increases conceptual surface area but avoids incorrect cross-vendor generalization. Another tradeoff is strict static profile gating on Intel-compatible hosts, which can frustrate operators who want a profile on unsupported silicon but prevents misleading emulation claims.

Cross-architecture differences are structural. The Intel-compatible path centers on an instruction-oriented discovery tree and model-specific registers; normalization is deep, vendor-branched, and concerned with legacy compatibility fields such as initial interrupt controller identifiers and extended cache descriptors. The Arm path centers on initialization feature arrays and explicit system register identifiers; static catalogues are smaller, and validation focuses on register width and bitmap range because a single register can be one hundred twenty-eight bits wide. Where the Intel-compatible path refuses to apply template bits to absent leaves, the Arm path emphasizes pre-read completeness for targeted registers and bitwise coherence at the declared width. Together, both paths implement the same product idea—declarative, filtered adjustments on top of a grounded baseline—even though the underlying hardware abstractions share almost no literal data formats.

Constraints from KVM and from architecture manuals shaped the design: hypervisor flag bits on identification entries exist because the kernel distinguishes indexed leaves from flat leaves; Arm’s KVM initialization structure uses parallel arrays for feature words, which maps naturally to indexed bitmap adjustments. The empty-template default preserves passthrough semantics for operators who trust the host’s baseline. Static profiles encoded both in source and in external interchange files allow auditing and parity checks between shipped defaults and documentation artifacts without divergent meanings.

The following table summarizes the systems-level contrast without tying the description to any particular identifier or module layout.

| Concern | Intel-compatible direction | Arm direction |
|--------|----------------------------|---------------|
| Primary discovery abstraction | Hierarchical identification leaves and model-specific registers | Feature words for init and architectural register images |
| Vendor handling | Distinct normalization branches for two major vendors | No vendor fork inside this package; platform expectations differ |
| Static profiles | Multiple named profiles keyed to vendor and CPU generation lists | Narrower set aimed at specific hardware class masking |
| Template application errors | Strong checks for leaf and register presence | Emphasis on width-aligned bitmap ranges |
| Fingerprinting role | Topology and brand leaves produce rich, testable signatures | Register-level signatures reflect init and system register policy |

This subsystem will continue to evolve with KVM and CPU errata, but the layered pattern—baseline from the kernel, normalization for correctness, templates for policy—should remain stable because it isolates operational policy from low-level emulation mechanics.
