# ACPI Table Generation for Virtual Machines

## Purpose & Boundaries

This subsystem is responsible for turning a virtual platform’s configuration into the binary ACPI artifacts that guest operating systems expect at boot. Its core job is to materialize standard system description tables, compute the integrity fields those tables require, and serialize them into guest physical memory at addresses chosen by the broader virtual machine stack. The work spans both fixed-layout tables defined by the ACPI specification and the AML bytecode that describes devices, resources, and control methods in the differentiated system description table.

What this layer does not do is decide where tables live in guest memory, how many virtual processors or devices exist, or how the hypervisor emulates hardware. Those policies belong to the machine model and device model layers above. This component also does not implement the guest firmware handoff that locates the root pointer; it assumes the caller will place the root structure where the boot path requires and will wire any firmware or bootloader expectations accordingly.

If this component failed or produced invalid tables, guests would typically fail ACPI initialization: power management, interrupt routing discovery, and device enumeration through the ACPI namespace would be unreliable or absent. On many operating systems, boot would halt early or fall back to limited hardware descriptions, breaking paravirtualized or ACPI-dependent devices even when the underlying emulation was correct.

## Interfaces & Contracts

Externally, the subsystem exposes typed representations of individual ACPI tables and a small set of shared behaviors: each table can report its serialized size and can write its bytes into guest memory through an abstract memory interface. Callers supply OEM identity fields, revision numbers, and—where tables point at one another—the physical addresses that must appear inside pointer fields. The AML side exposes composable building blocks that append well-formed ACPI Machine Language encodings into byte buffers, suitable for embedding as the payload of the differentiated system description table or for other definition blocks.

The subsystem consumes a guest memory abstraction so that serialization does not assume a particular host mapping strategy. It depends on layout-oriented facts (such as guest physical addresses for linked tables) from the caller. A shared checksum routine ensures every table’s single-byte additive checksum is consistent with ACPI rules when headers and payloads are split across buffers.

Callers must maintain several invariants. Table lengths must remain within representable fields, and guest addresses used in pointer fields must refer to where the caller will actually place the corresponding table. For tables whose headers and bodies are written as separate slices, address arithmetic must not overflow the guest address space. The AML builders assume valid name paths and resource encodings; malformed names or inconsistent resource templates produce encoding errors rather than silent output.

In return, the subsystem guarantees that standard header fields it controls—signatures, length fields, revision markers where fixed by construction, and creator metadata embedded in headers—are written consistently. Checksums are computed over the exact byte ranges ACPI expects. Tables that split header and trailing payload write those parts in order at contiguous increasing guest addresses.

## Data Flow

Configuration intent enters as parameters: OEM strings, table revisions, lists of guest physical addresses for tables that the extended system description table will reference, interrupt controller descriptions for the multiple APIC description table, AML blobs for the differentiated system description table, and optional fields for the fixed ACPI description table such as feature flags or hypervisor identification. The flow is almost entirely one-way: inputs are expanded into packed byte representations, checksums are applied, and the result is copied into guest memory.

The root system description pointer is constructed with a pointer to the extended system description table. That table, in turn, holds an array of sixty-four-bit little-endian addresses pointing to other system description tables the platform exposes. The fixed ACPI description table carries both legacy thirty-two-bit fields (often left unused in reduced-hardware virtual platforms) and extended sixty-four-bit generic address structures; critically, it carries the guest physical address of the differentiated system description table so the operating system can load the AML definition block. The multiple APIC description table combines a fixed header that names the local advanced programmable interrupt controller’s base address with a variable-length sequence of interrupt controller structures (for example local application processors and I/O units). The differentiated system description table wraps an AML definition block with a standard system description header.

AML data is produced separately: hierarchical constructs, named objects, packages, resource templates, and methods serialize into bytecode. That bytecode becomes the definition block portion of the differentiated system description table. Resource descriptors for memory, I/O, interrupts, and PCI-style address spaces are encoded into buffers that operating systems interpret when evaluating named resource methods.

```
  Platform intent (OEM ids, addresses, CPU/APIC layout, AML)
                           |
                           v
              +---------------------------+
              |  Per-table construction |
              |  (headers + payloads)   |
              +---------------------------+
                           |
         +-----------------+------------------+
         |                 |                  |
         v                 v                  v
    [Root pointer]   [Table directory]   [FADT + MADT + DSDT ...]
         |                 |                  |
         +--------+--------+--------+---------+
                  |
                  v
           Guest physical memory
           (caller-chosen layout)
```

Before the diagram above, the inputs are purely declarative: they describe what the guest should believe about the machine. The middle stage turns those declarations into concrete byte layouts with correct lengths and linkage. After the diagram, the output is opaque to the guest except through ACPI parsing; the hypervisor must ensure the written regions are reachable and not overwritten unexpectedly during boot.

## Control Flow

Execution is driven by the virtual machine bring-up sequence when the integration layer decides ACPI is required. The critical path begins after guest memory is available and platform addresses for each table are known. The extended system description table is typically populated with the addresses of the fixed ACPI description table, the multiple APIC description table, and any additional tables the integration layer includes. The root system description pointer is then filled with the address of the extended system description table and its checksums are computed.

Branching appears in two places: optional configuration of the fixed ACPI description table (which flags and address fields are meaningful depends on whether the platform is hardware-reduced, whether certain PC/AT expectations apply, and whether the hypervisor wants to advertise vendor identity), and the shape of AML (different devices and buses require different resource templates and scopes). The AML subsystem itself follows a compositional pattern: each encoding appends to a buffer when explicitly converted, and larger structures nest smaller ones.

There is no long-running runtime loop inside this subsystem. Once tables are written, control returns to the caller; any further changes require deliberate regeneration and rewriting, which is uncommon except for hotplug or dynamic reconfiguration scenarios handled elsewhere.

## State & Lifecycle

Persistent state owned by this subsystem is minimal during generation: intermediate vectors of bytes for table bodies, header templates with checksum fields cleared until finalization, and AML builder state held only while encoding. There is no server-style session state. Initialization is implicit when constructors run; cleanup is automatic when the owning objects go out of scope in the host process.

Lifecycle phases align with virtual machine creation. During startup, the integration layer allocates guest memory regions for ACPI, builds tables in a defined order so addresses used in pointers are stable, writes each table, and publishes the root pointer through firmware or platform-specific mechanisms. Steady-state operation usually does not touch these pages. Shutdown or reset may discard guest memory entirely; the host-side builders carry no obligations across resets unless the caller reuses them for a new guest instance.

Recovery is straightforward: if writing to guest memory fails, the error propagates upward and the caller can abort bring-up. There is no partial transactional protocol inside this layer; idempotence depends on the caller not marking the guest as bootable until all writes succeed.

## Failure Modes

Guest memory write failures surface as errors from the memory abstraction; they are not masked. Invalid guest addresses—such as overflows when advancing past a header to write a trailing payload—are rejected rather than wrapped silently.

Checksum or length mistakes would cause firmware or the operating system to reject tables; this implementation reduces that risk by centralizing checksum computation and tying declared lengths to serialized sizes. If a caller supplies inconsistent addresses in pointer fields, the guest may load AML from the wrong location or follow dangling references; that is a contract violation outside this layer’s ability to detect.

AML construction can fail when names are not four-character segments as required by ACPI naming rules, when address ranges for resource descriptors are inverted, or when method arity constraints are violated by the builder API. Those failures are explicit rather than producing malformed bytecode silently.

Assumptions that could cause subtle corruption if violated include: the guest physical address width is sufficient for encoded pointers, the memory backing the ACPI region remains stable for the lifetime of ACPI parsing at boot, and no other component overwrites ACPI pages after they are written. Silent corruption is most likely from integration mistakes—such as reusing a buffer while computing checksums—rather than from internal logic, because checksum recomputation happens immediately before writing for tables that mutate checksum fields in place.

## Operational Characteristics

Resource use is dominated by temporary buffers proportional to AML size and the number of interrupt controller structures. For typical virtual machines, this is small relative to guest RAM. The scaling bottleneck is not CPU time but careful ordering of construction: very large AML blocks increase encoding time linearly, and extremely wide tables stress address bookkeeping in the integration layer more than this library.

There is no intrinsic logging, metrics, or tracing inside this subsystem; observability depends on the caller’s boot tracing. From an operations perspective, the important signals are guest failures to parse ACPI, which appear as early boot warnings or kernel messages about invalid table checksums or signatures.

## Design Rationale

ACPI exists so operating systems can discover platform topology and manage power without hardcoding every machine. For virtual machines, ACPI is synthesized rather than read from firmware ROM, which requires a faithful yet flexible encoding path. Splitting responsibilities keeps this layer focused on specification-accurate serialization while letting the machine model decide policy.

Using a unified header pattern and shared checksum logic reduces duplication and the chance that one table’s checksum rules diverge from another’s. Representing the extended system description table as a list of sixty-four-bit pointers matches modern ACPI expectations and avoids the thirty-two-bit limitations of the legacy root table directory for large guest physical address spaces.

The fixed ACPI description table retains full-field layout so hypervisor integrations can opt into legacy hardware features or hardware-reduced profiles without forking the serializer. Exposing extended address fields for the differentiated system description table reflects how sixty-four-bit guests locate AML in high memory.

The multiple APIC description table models interrupt controllers as a typed header plus an extensible byte sequence because ACPI allows a heterogeneous list of variable-length records; pushing record construction to callers keeps the core flexible for different CPU counts and I/O unit placements.

The AML subsystem is large because ACPI’s bytecode is verbose and hierarchical; providing structured builders for packages, scopes, devices, methods, fields, operation regions, and resource templates shifts complexity away from every integration point and centralizes encoding rules like package length and resource template framing. Automated checks can compare generated bytes to specification-derived golden patterns, which anchors correctness against the wire format without relying on informal hand-editing.

The tradeoff is explicit: flexibility and spec fidelity over minimalism. A slimmer encoder would force callers to hand-roll more bytecode, increasing the risk of interoperability bugs with guest parsers.

---

## Topology: From Root Pointer to Guest Tables

The following diagram shows how the root pointer anchors the extended table directory, which fans out to the major tables discussed in this document. Arrows represent logical references encoded as guest physical addresses inside ACPI structures.

```
                    +------------------+
                    | Root pointer     |
                    | (ACPI 2.0+)      |
                    +--------+---------+
                             |
                             | contains address of
                             v
                    +------------------+
                    | Extended system  |
                    | description      |
                    | table (directory)|
                    +--------+---------+
                             |
           +-----------------+------------------+
           |                 |                  |
           v                 v                  v
    +-------------+  +-------------+   +------------------+
    | Fixed ACPI  |  | Multiple    |   | Other SDTs     |
    | description |  | APIC descr. |   | (platform-     |
    | table       |  | table       |   |  specific)     |
    +------+------+  +-------------+   +------------------+
           |
           | points to (extended field)
           v
    +----------------------+
    | Differentiated       |
    | system description   |
    | table (AML payload)  |
    +----------------------+
```

Before the diagram, recall that the root pointer is the discovery handle: firmware or the loader exposes it, and the operating system’s ACPI subsystem starts traversal there. The extended system description table is the hub that lists every other system description table participating in this virtual platform. After the diagram, the important takeaway is that the differentiated system description table is reachable both through the directory (as a listed table) and through the fixed ACPI description table (as the primary AML-bearing definition block), which matches how guests resolve the namespace graph.

## Data Flow: Checksums and Split Writes

ACPI mandates an eight-bit checksum over each system description table such that the sum of all bytes modulo 256 equals zero. Some tables compute checksum over a contiguous serialization, while others combine the header bytes with trailing payload bytes logically contiguous in guest memory.

```
   [Header bytes]     [Payload bytes]
         |                    |
         +---------+----------+
                   |
                   v
            Additive fold (8-bit)
                   |
                   v
            Complement to zero sum
                   |
                   v
            Write header + payload
            at consecutive GPAs
```

Before the diagram, note that the checksum algorithm is simple by design so firmware and kernels can verify tables without heavy cryptography. After the diagram, the operational implication is that any last-minute patch to a table after checksum calculation requires recomputation; writers finalize checksums immediately before guest writes to keep the invariant intact.

## State: Minimal Build-Time Machine

The effective state machine for table generation is shallow: prepare fields, finalize integrity fields, emit bytes. There is no concurrent state shared across guests.

```
        +-----------+
        | Construct |
        +-----+-----+
              |
              v
        +-----------+
        | Finalize  |
        | checksums |
        +-----+-----+
              |
              v
        +-----------+
        | Write to  |
        | guest RAM |
        +-----+-----+
              |
              v
        +-----------+
        | Complete  |
        +-----------+
```

Before the diagram, construction may involve multiple steps for composite tables such as the multiple APIC description table, where the header depends on the size of the trailing records. After the diagram, completion means the guest-visible image is consistent; there is no asynchronous persistence step inside this layer.

## AML as a Layered Bytecode Factory

AML for guests is not authored as a monolithic binary in this design; it is assembled from conceptual layers. At the lowest layer, literal values and name paths encode as typed term bytes. Structured objects combine those literals into named declarations. Device and scope constructs nest declarations into an object tree that mirrors how operating systems walk the ACPI namespace. Resource templates bundle low-level descriptors—fixed memory blocks, address space ranges, I/O ports, extended interrupts—into buffers assigned to standard names that drivers evaluate. Control methods bundle procedural bytecode for dynamic behavior, including branches, comparisons, arithmetic, and access to locals and arguments.

Package-length encoding recurs throughout AML because many opcodes wrap variable-length term lists; the builder computes these lengths so nested structures remain self-consistent. Resource templates wrap descriptor streams in a buffer object with a terminator and checksum placeholder appropriate for resource serialization.

This separation—tables for discovery and linkage, AML for programmable description—mirrors ACPI’s own layering. Virtual platforms benefit because the same AML machinery can serve multiple machine models: only the resource and device templates change, while the table scaffolding (root pointer, directory, interrupt description, fixed ACPI description table linkage) follows a repeatable pattern.

## Closing Notes on Guest Correctness

Successful ACPI for guests is as much about interoperability as about filling fields. Operating systems are strict about signatures, lengths, and checksums, and they interpret AML with expectations inherited from hardware ecosystems. A virtual platform therefore aims for conservative, specification-aligned encodings rather than clever minimalism. The building blocks described here exist to make that conservatism systematic: one checksum implementation, one header style, one AML composition path, and explicit failure when inputs do not map cleanly to ACPI’s wire formats.

When integrating this subsystem, treat the byte image as part of the guest ABI: changing OEM strings, table revisions, or AML templates can alter how guests enumerate devices or route interrupts even if the underlying emulation is unchanged. Stability and predictability—anchored by the structural relationships among the root pointer, extended directory, fixed ACPI description table, multiple APIC description table, and differentiated system description table—are the operational guarantees this architecture is optimized to support.
