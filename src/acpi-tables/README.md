# ACPI table construction for virtual machines

1. **Role in the firmware and boot path**

   1.1. A virtual machine monitor must supply ACPI System Description Tables in guest physical memory because there is no traditional platform firmware handing off the same data. Guest kernels locate the root pointer, walk the description table graph, and hand the Differentiated System Description Table’s definition block to the ACPI Machine Language interpreter. This library builds those byte images on the host and writes them through an abstract guest memory interface.

   1.2. Work splits into two layers: fixed-layout binary tables whose layouts are specification-defined, and an ACPI Machine Language encoding layer used to describe devices, methods, and resource templates inside the definition block. Tables are assembled in host buffers, checksummed, then copied to chosen guest addresses.

2. **How tables link to one another**

   2.1. **Discovery anchor.** The root structure is the first object the operating system searches for (typically in low memory or via firmware configuration on UEFI systems). It carries an eight-byte signature, structure revision, OEM identifier bytes, total length of the extended layout, and critically a 64-bit physical address of the Extended System Description Table. The legacy 32-bit table pointer in the same structure is cleared so modern guests follow only the 64-bit route; mixing both routes incorrectly can cause ambiguous or wrong table sets.

   2.2. **Central directory.** The Extended System Description Table is a standard table header followed by a dense array of 64-bit little-endian pointers. Each pointer is the guest physical address of another system description table. There is no inherent ordering requirement in the specification, but integrators should treat the list as the authoritative manifest: every table the platform exposes (fixed ACPI description, multiple APIC description, differentiated definition, and any additional tables) must appear here exactly once with a valid address. Missing entries mean the guest never sees that table; duplicate or dangling addresses manifest as parser failures or subtle runtime bugs.

   2.3. **Bridge into bytecode.** The Fixed ACPI Description Table is both a large fixed-feature record and the primary static link from static tables into the definition block. The extended 64-bit field holds the guest physical address of the Differentiated System Description Table. Until that field points at a placed definition block, the ACPI namespace under the root scope is incomplete regardless of how rich other tables are. The legacy 32-bit field in the same table remains unused when only the extended pointer is populated.

   2.4. **Interrupt topology without AML.** The Multiple APIC Description Table is listed alongside other tables in the central directory; it does not reference the definition block. It carries a local APIC base address and a byte stream of interrupt controller structures. The guest’s interrupt routing logic consults this table in parallel with AML; inconsistency between declared APIC entries and what devices claim in resource templates leads to broken IRQ assignment.

   2.5. **Definition block as leaf.** The Differentiated System Description Table is header plus raw definition bytes. It is not itself a pointer hub: complexity is entirely in the ACPI Machine Language it contains. Device scopes, methods, and named objects form a namespace tree that must agree with what the hypervisor actually emulates.

   2.6. **End-to-end graph.** Conceptually: the anchor points at the directory; the directory lists the fixed ACPI description table, the multiple APIC description table, the differentiated table, and any extras; the fixed ACPI description table points at the differentiated table; the differentiated table holds the bytecode graph. Integrators must assign guest physical addresses so this graph is consistent at the instant the guest reads it—no pointer may be stale, overlap another structure’s storage, or refer to memory the guest will use for unrelated purposes.

```
  Guest physical memory
  +------------------+
  | Anchor           |----.
  +------------------+    |
                          | 64-bit directory address
                          v
  +------------------+       points to each of
  | Directory        |------------------------> [ fixed ACPI | interrupt | definition | ... ]
  +------------------+
        |
        | extended definition-block pointer inside fixed ACPI
        v
  +------------------+
  | Definition block |
  +------------------+
```

3. **Checksum model**

   3.1. **Additive zero property.** For every system description table image, the eight-bit sum over all bytes including the checksum byte must be zero in unsigned wrapping arithmetic. Equivalently, the checksum byte is chosen so that the cumulative sum folds to zero. Empty contributions to a split checksum (for example an empty slice concatenated into the sum) add nothing, which matches expectations for degenerate cases.

   3.2. **Implementation shape.** The checksum helper accepts a list of byte slices, flattens them in order, sums every byte with wrapping addition, then applies a fixed transformation (subtract from 255, then add one with wrap) to yield the single checksum byte. The unit tests pin concrete examples: small payloads, empty slices, and multi-slice inputs all produce the byte that makes the full table sum to zero.

   3.3. **Per-table behavior.** Standard table headers are built with the checksum field cleared, then the checksum is filled after the total length and payload are known. For tables whose image is a header concatenated with a variable tail, both parts participate in one checksum spanning header and tail in order.

   3.4. **Anchor with two scopes.** The root pointer structure is special: a first checksum covers only the original twenty-byte legacy prefix so validators that understand only the small layout still verify correctly. A second checksum covers the entire variable-length structure including the 64-bit directory address, extended length, reserved tail, and both checksum bytes themselves must be consistent with the additive-zero rule over their respective ranges. Construction fills both after all fields are set.

   3.5. **Directory table.** The checksum spans the full contiguous image: the generic header immediately followed by the packed pointer array. No checksum is stored separately for the pointer region.

   3.6. **Composite interrupt table.** The checksum spans the interrupt table’s composite header (generic header plus local APIC base and flags) and the opaque interrupt-controller byte vector as one logical range.

   3.7. **Definition block table.** Checksum covers generic header plus definition bytes; header length equals header size plus payload size.

   3.8. **Large fixed-layout table.** One table serializes as a single packed struct. Right before writing to guest memory, the checksum in the embedded header is recomputed over the entire struct image so any late mutation of fields is reflected. This matters when integrators adjust fields after initial construction.

4. **Shared header and generic register addressing**

   4.1. **Common header fields.** Every system description table begins with signature, total length, revision, checksum placeholder, six-byte OEM identifier, eight-byte OEM table identifier, OEM revision, four-byte creator identifier, and four-byte creator revision. Constructors populate all but checksum; a fixed creator tag and a date-like creator revision constant stamp provenance consistently across emitted tables.

   4.2. **Generic address structure.** Register locations are encoded with address space identifier (memory, I/O, PCI configuration, etc.), bit width and offset within the register, access size, and a 64-bit little-endian address. This appears in extended fixed ACPI fields for reset, sleep control and status, and extended power-management blocks. Values must match what the virtual hardware actually decodes.

5. **Abstract “write a table” contract**

   5.1. Each concrete table type exposes its total byte length and can write itself into guest memory at a base address. Length comes either from the header’s length field or from header size plus trailing data size as appropriate.

   5.2. **Split writes.** When the wire image is not one contiguous in-memory struct, writers emit the header first, then advance the guest address by the header size using checked arithmetic. Overflow or invalid addresses surface as explicit errors rather than silent wrap. The tail is written in a second operation.

6. **Error separation**

   6.1. Failures from the guest memory backend (read/write errors) are distinct from logical address errors (overflow when computing the next write address, invalid register sizing where the API enforces constraints). Integrators should not conflate “could not reach guest RAM” with “table logic inconsistent.”

7. **Fixed ACPI description table**

   7.1. Targets revision six with a minor version field set to match expected sub-revision behavior. The structure is a large packed record: legacy 32-bit block addresses, extended generic address structures for the same roles, feature flags, IA-PC boot architecture flags, hardware-reduced flag, hypervisor vendor identifier bytes, and more.

   7.2. **Hardware-reduced mode.** When the platform declares hardware-reduced ACPI, legacy PM1, PM2, GPE, SMI command, and related fixed blocks are ignored by the guest; fields from SCI interrupt through century, and extended counterparts for PM and GPE blocks, need not describe real chipset hardware. The operating system relies on software-mediated events and simplified sleep control via generic address structures instead.

   7.3. **Setter surface.** Integrators can set the extended definition-block pointer, global feature flags, IA-PC boot flags, and an eight-byte hypervisor vendor string. The extended definition pointer is the critical link for AML.

8. **Multiple APIC description table**

   8.1. After the generic header, the table records a 32-bit local APIC MMIO base (commonly the standard x86 local APIC base) and flags, then appends a vector of raw interrupt controller structures.

   8.2. **Built-in local and I/O APIC entries.** Local entries mark a logical processor with processor identifier, APIC identifier, and an enable flag in the flags word. I/O APIC entries carry an identifier, physical base address, and global system interrupt base for the first register window.

   8.3. **Opaque extension.** Additional controller types may be appended as pre-serialized bytes; the library treats that region as an opaque blob once the composite header is consistent. Integrators are responsible for valid type-length encodings per the ACPI specification.

9. **Differentiated system description table**

   9.1. Wraps a definition block: generic header with the appropriate signature and revision, followed by verbatim ACPI Machine Language bytes. Length in the header is header size plus definition size.

   9.2. Device topology, power resources, control methods, and named data live here. The table wrapper is thin; almost all complexity is in how those bytes are produced.

10. **ACPI Machine Language encoding architecture**

    10.1. **Composable emission.** Every construct implements a common “append wire bytes to a growable buffer” operation. A helper allocates a fresh buffer for a standalone term. Errors are structured (empty name, bad segment length, invalid address range for resource descriptors) rather than panics.

    10.2. **Named terms.** A name binding emits the name opcode, an encoded path, and a value term. This is how standard objects attach hardware identifiers, resource templates, and other data to the namespace.

11. **Name paths**

    11.1. Paths parse from strings: an optional leading root marker, then segments of exactly four characters separated by dots for multi-segment paths. Encoding emits the root character when present; two segments use a dual-name prefix; three or more use a multi-name prefix followed by a segment count and four-byte chunks per segment.

    11.2. Violations such as empty paths or wrong segment length are errors at construction time.

12. **Primitive constants and integers**

    12.1. Single-byte encodings exist for zero, one, and all-bits-one. Wider integers use typed prefixes (byte, word, dword, qword) followed by little-endian payload. Unsigned machine-width values choose the narrowest integer encoding that fits.

13. **Packages and length encoding**

    13.1. **Packages.** A package serializes a leading byte with element count, then each child term in order, then wraps the payload in a package opcode and a variable-length package length prefix.

    13.2. **Package length algorithm.** The high two bits of the first length byte encode how many additional length bytes follow. Values up to 63 fit in one byte using the low six bits. Larger values use multi-byte encodings: reserved bit constraints apply on the lead byte when more than one byte is required; lower nibbles and follow-on bytes concatenate to form the length. The encoding is inclusive: for constructs that require it, the length includes the length bytes themselves, which the helper accounts for when choosing width.

    13.3. **Shared helper.** The same length machinery wraps packages, methods, scopes, devices, resource buffers, and other length-prefixed terms. A separate mode excludes self from the inclusive length where field internals require it.

14. **Resource templates**

    14.1. **Pipeline.** Children (individual resource descriptors) serialize first into a temporary buffer. An end tag and a zero checksum byte append. The buffer length is encoded as an integer term, byte-inserted at the front of that temporary buffer (length includes descriptors, end tag, and checksum byte). The whole inner buffer is then wrapped with a buffer opcode, a package length, and the inner bytes.

    14.2. **Ordering sensitivity.** Because length is inserted at the front after the body is known, the encoder never needs to precompute total size before emitting descriptors; integrators should still think in terms of final resource descriptor order, since that order is what the guest interprets.

15. **Resource descriptors in detail**

    15.1. **Fixed 32-bit memory range.** Short-form descriptor with read/write versus read-only flag, base, and length in the ACPI short layout.

    15.2. **Word, dword, and qword address space descriptors.** Each selects address space kind (memory, I/O, bus number), applies generic flags marking minimum and maximum as fixed, sets type-specific flags (cacheability and read/write for memory; “entire range” style behavior for I/O; neutral flags for bus numbers), emits zero granularity, minimum, maximum, zero translation offset, and length computed as maximum minus minimum plus one. Constructors reject minimum greater than maximum.

    15.3. **Legacy I/O port range.** 16-bit decode with minimum, maximum, alignment, and length bytes for ISA-style fixed ports.

    15.4. **Extended IRQ.** Carries consumer versus producer, edge versus level, active high versus low, shared versus exclusive, a count byte, and one or more IRQ numbers in the concrete encoder (the provided single-IRQ helper emits a count of one). Used heavily in current resource settings for line-based interrupts.

16. **Device, scope, and method encodings**

    16.1. **Device.** Emits an extended opcode pair, then package length, then an encoded path followed by child terms. Nesting is arbitrary: children can include scopes, methods, named objects, and further devices.

    16.2. **Scope.** Emits a scope opcode, package length, path, and children—similar nesting model without the device-specific extended prefix.

    16.3. **Methods.** Emit method opcode, package length, path, a flags byte combining argument count (up to seven) and serialized flag, then the method body. **Return** emits return opcode followed by the value term.

    16.4. **Invocation.** Calling a method encodes the callee path followed by argument terms in order; there is no separate “call” opcode—invocation is syntactically “name then arguments.”

17. **Operation regions and fields**

    17.1. **Operation region.** Binds a name to a space type (system memory, I/O, PCI config, embedded controller, SMBus, CMOS, PCI BAR target, IPMI, GPIO, serial bus, etc.) plus offset and length as integer terms.

    17.2. **Fields.** Describe bit layouts over an operation region: access type (any, byte, word, dword, qword, buffer), update rule (preserve, write ones, write zeros), then a sequence of named bitfields or reserved regions. Named and reserved entries use the package-length helper for field widths in the non-inclusive mode.

18. **Control flow and data movement**

    18.1. **Conditionals and loops.** If and while serialize predicate and body terms under package length the same way as packages.

    18.2. **Comparisons.** Equality and less-than comparisons emit opcode then operands in the order expected by the wire format.

    18.3. **Store.** Emits store opcode, then **value**, then **target**—operand order matches ACPI Machine Language’s store semantics, not intuitive “destination first” ordering.

    18.4. **Arguments and locals.** Argument slots zero through six and local slots zero through seven encode as single-byte opcodes; out-of-range indices are errors.

    18.5. **Mutex, acquire, release.** Mutex declares a path and sync level. Acquire extends with a 16-bit timeout. Release targets the mutex path.

    18.6. **Notify.** Emits notify opcode, object term, then notification value term.

19. **Arithmetic and binary operations**

    19.1. A family of binary operators follow the pattern: opcode, first operand, second operand, target—covering add, subtract, multiply, shifts, bitwise operations, string concatenation variants, modulo, indexing, conversion to string, and related operations.

20. **Buffers and dynamic fields**

    20.1. **Buffer.** Serializes an integer length prefix, raw payload bytes, then buffer opcode and package length around the combined inner buffer.

    20.2. **Create field.** Variants exist for creating dword-wide and qword-wide field aliases into a buffer given base buffer term, offset term, and target path—used when methods patch resource buffers at runtime.

21. **EISA identifiers and strings**

    21.1. **EISA compression.** Seven-character PNP-style strings compress into a 32-bit quantity per ACPI EISA rules (compressed letters and hex nybbles), then emit as a dword integer constant—typical for hardware identifier bindings.

    21.2. **Strings.** String opcode, UTF-8 payload bytes, null terminator.

22. **Representative golden patterns**

    22.1. Internal tests compare fully serialized byte arrays against reference blobs for realistic constructs: serial port devices with hardware identifier and current resource settings combining extended IRQ and legacy I/O descriptors; board-level scopes with memory and IRQ sets; PCI-style resource templates; mutex acquire and release sequences; dynamic resource methods. These guard regressions in the length encoder and resource template pipeline.

23. **Wire-format safety**

    23.1. Packed structs with derive-driven layout guarantees ensure in-memory images of headers and fixed tables match on-the-wire ACPI layout when copied byte-for-byte to the guest.

    23.2. The guest memory abstraction isolates table construction from how backing storage is implemented, so the same builders can run in different monitors as long as the memory trait implementation is correct.

24. **Integrator constraints**

    24.1. **Guest physical layout.** The library does not reserve memory or enforce alignment beyond what writers need for split header and body. Integrators must place each table in a non-overlapping region that remains stable for the guest’s lifetime and is not reused for RAM, DMA buffers, or firmware blobs unless intentionally shared.

    24.2. **Pointer consistency.** All pointers in the anchor, directory, and fixed ACPI table must be filled with the final addresses used during the write phase. Partial initialization (for example directory built before tables are copied) is safe only if the integrator never exposes the guest to the intermediate state.

    24.3. **ACPI revision coherence.** Table revision fields, fixed ACPI revision, minor version, and the capabilities implied by feature flags should match the AML and static tables actually implemented. Mismatch confuses operating systems that gate features on revision and flags.

    24.4. **Namespace completeness.** Missing current resource settings, missing routing tables for PCI, or APIC entries that do not reflect vCPU and IRQ wiring produce driver failures that appear as “hardware not found” rather than as errors from this library. The monitor must emit AML and static tables that jointly describe what it emulates.

    24.5. **Hardware-reduced honesty.** If the fixed table declares hardware-reduced operation, legacy PM and GPE blocks should not be relied upon by AML or by the guest; conversely, if the monitor emulates legacy ACPI ports, flags and fields must be set consistently so the guest’s ACPI core does not skip required setup paths.

    24.6. **Error handling at the boundary.** Guest memory write failures propagate to the caller; integrators should treat them as fatal to ACPI setup for that boot attempt, since partial writes leave checksums or pointers invalid.
