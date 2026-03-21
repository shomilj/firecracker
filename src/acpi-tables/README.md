# ACPI table construction for virtual machines

1. **Purpose and placement in the firmware stack**

   1.1. The library exists to synthesize ACPI data structures that a guest operating system discovers at boot. A hypervisor or virtual machine monitor places those structures in guest physical memory; firmware does not supply a BIOS in the same way bare metal does, so the VMM must emit tables that match what the guest kernel’s ACPI interpreter expects.

   1.2. The design splits into two cooperating layers: **binary ACPI System Description Tables (SDTs)** whose layouts are fixed by specification, and an **AML (ACPI Machine Language) bytecode layer** used to describe devices, methods, and resource templates inside the Differentiated System Description Table. Tables are built in host memory, checksummed, then written into the guest address space through an abstract guest memory interface.

   1.3. **Checksum discipline** applies throughout. Each table uses an additive checksum over every byte such that the eight-bit sum of the entire structure (including the checksum byte) is zero. The implementation computes this by taking the negation of the sum of all payload bytes in wrapping eight-bit arithmetic, then adding one—equivalently “what byte makes the total fold to zero.” Empty slices contribute nothing, which matches ACPI’s expectation for trivial tables.

   1.4. **Creator identity** is embedded consistently: a fixed four-byte creator tag and a 32-bit revision number (a date-like constant) are stamped into every SDT header so tools and OS parsers can attribute table provenance.

2. **Cross-cutting core: headers, addressing, and the table trait**

   2.1. **System Descriptor Table header** models the common prefix for ACPI tables: four-byte signature, total length, revision, checksum placeholder, six-byte OEM identifier, eight-byte OEM table identifier, OEM revision, creator id, creator revision. Constructors fill all fields except the checksum; callers or specialized writers set the checksum once the full byte image is known.

   2.2. **Generic Address Structure (GAS)** encodes where registers live: address space id (memory, I/O, PCI config, etc.), bit width and offset within the register, access size, and a 64-bit address. This appears in tables like the Fixed ACPI Description Table for reset registers, sleep control/status in extended layouts, and extended PM blocks. Values use little-endian integer types at the wire level.

   2.3. The **abstract table trait** unifies “something that has a length and can be serialized into guest RAM.” Length is defined per concrete type—either from embedded length fields or from header plus trailing payload sizes. Serialization writes bytes through the guest memory abstraction; overflow when advancing guest addresses surfaces as an explicit error rather than wrapping silently.

   2.4. **Error taxonomy** separates guest memory faults (read/write failures through the memory backend) from logical issues such as invalid guest addresses after checked arithmetic and invalid register sizing where the API enforces constraints.

3. **Root System Description Pointer (RSDP)**

   3.1. The RSDP is the anchor the OS searches for in low memory or EFI configuration tables. The synthesized structure follows ACPI 2.0+: eight-byte signature, first checksum over the original 20-byte “legacy” portion, OEM id, structure revision (2), legacy 32-bit RSDT address field cleared when only the extended path is used, total structure length including the extended tail, **64-bit XSDT physical address**, extended checksum over the entire structure, and reserved padding.

   3.2. **Dual checksums**: the first byte covers only the initial 20 bytes for backward compatibility validators; the extended checksum covers the full variable-length RSDP including the 64-bit pointer and extended fields. Both are computed with the same additive-zero property.

   3.3. Construction fixes the XSDT pointer at build time; the legacy RSDT field stays zero to signal that only the 64-bit route should be followed on modern guests.

4. **Extended System Description Table (XSDT)**

   4.1. The XSDT is an array of **64-bit little-endian pointers** to other SDTs, prefixed by a standard header whose signature is “XSDT”. The implementation concatenates the header with the packed pointer list and checksums header and body together as one contiguous byte range.

   4.2. **Write pattern**: header first at the base guest address, then the pointer array immediately following, with address arithmetic checked so the second write cannot silently truncate.

   4.3. Semantically, the XSDT is the **catalog**: order of entries can matter for some firmware parsers, but logically it is an unordered set of pointers the platform exposes; the hypervisor decides which tables exist and where they reside.

5. **Fixed ACPI Description Table (FADT / FACP)**

   5.1. The FADT is a large packed record bridging **legacy fixed hardware** fields and **extended 64-bit GAS** fields. Revision 6 is targeted; a minor version field is set to match expected sub-revision behavior.

   5.2. **Hardware-reduced ACPI** is the dominant mode for many virtual platforms: legacy PM1/PM2/GPE blocks, SMI command ports, and related fields can be left zero because the guest is not expected to drive chipset-style ACPI hardware. A flag exists to declare hardware-reduced operation so the OS uses software alternatives (interrupt-based events, simplified sleep control via GAS, etc.).

   5.3. **DSDT linkage**: the primary hook from FADT into AML is the **64-bit extended DSDT address** field; a dedicated setter updates that pointer while the legacy 32-bit DSDT field remains at default zero when unused.

   5.4. **Feature flags** (32-bit) and **IA-PC boot architecture flags** (16-bit) can be set independently—e.g., indicating absence of VGA, MSI, or ASPM at the firmware handoff, or toggling global ACPI capabilities. These influence how Windows and Linux interpret available subsystems without requiring AML.

   5.5. **Hypervisor vendor id**: eight bytes reserved in the extended FADT for a vendor string; the VMM can stamp an identifier visible to OS code that reads this field.

   5.6. **Serialization** re-checksums the entire packed struct on write (single contiguous image) and writes it as one slice, ensuring any last-minute field mutation is reflected in the checksum.

6. **Multiple APIC Description Table (MADT / APIC)**

   6.1. The MADT describes interrupt controllers: a header including **local APIC base address** (typically 0xFEE00000 on x86) and flags, followed by a **variable-length list of interrupt controller structures** encoded as raw bytes.

   6.2. **Local APIC entries** are small records: type 0, length 8, processor UID and APIC id (often kept equal per vCPU index), and flags with the enable bit set so the CPU is marked usable.

   6.3. **I/O APIC entries** describe a system I/O APIC: type 1, length 12, id, reserved byte, 32-bit physical base address, and **Global System Interrupt base** (often zero for the first chip).

   6.4. The **interrupt controller blob** can hold additional APIC structures beyond these two (e.g., interrupt source overrides, local x2APIC) if the VMM appends pre-serialized bytes; the library treats that region as opaque payload once the header is consistent.

   6.5. **Checksum** spans the composite header structure plus the entire controller byte vector. Writes split into header then body with checked address math.

7. **Differentiated System Description Table (DSDT)**

   7.1. The DSDT wraps an AML **definition block**: a standard SDT header (signature “DSDT”, revision commonly 2) followed by raw AML bytes. The definition block is produced elsewhere—typically by composing AML fragments—and passed in as a byte vector.

   7.2. **Checksum** covers header plus definition block. Length in the header equals the sum of header size and payload size.

   7.3. The DSDT is where the bulk of **device topology** lives: scopes, devices, power resources, and control methods. The table itself is intentionally thin; complexity lives in AML generation.

8. **AML subsystem: encoding model**

   8.1. **Trait-based emission**: every AML construct implements a common interface that appends bytes to a growable buffer (and can fail with structured errors). A convenience helper allocates a fresh buffer for whole-term serialization.

   8.2. **Name paths** parse strings into ACPI name segments: optional leading root `\`, segments of exactly four characters, separated by `.` for multi-segment paths. Encoding uses root char, dual-name prefix for two segments, multi-name prefix with count for three or more, then four-byte chunks per segment. Violations (empty path, wrong segment length) are errors.

   8.3. **Primitive constants**: single-byte encodings exist for zero, one, and all-bits-one constants. Integers emit typed opcodes: byte, word, dword, qword prefixes followed by little-endian payload. Unsigned machine-width values pick the narrowest AML integer form that fits.

   8.4. **Named terms** combine the name opcode, a path, and a value term (another AML fragment). This is how `_HID`, `_CRS`, `_UID`, etc. are bound to data.

   8.5. **Package** encoding builds a small header with element count, serializes children in order, then wraps the result in **PkgLength**—a variable-length integer encoding whose high bits of the first byte declare how many follow-up bytes participate. The same helper is reused for any construct that needs ACPI’s package-length wrapper (methods, scopes, device bodies, etc.).

   8.6. **PkgLength algorithm** mirrors the spec: lengths up to 63 in one byte; larger values use multi-byte encodings with the top two bits of the lead byte indicating width, lower nibbles contributing to the numeric length, and an inclusive length that can account for the length bytes themselves when required.

9. **Resource templates and IRQ/IO/Memory descriptors**

   9.1. **ResourceTemplate** wraps a set of resource descriptors in a buffer opcode: child descriptors serialize first, an end tag with zero checksum is appended, then the buffer length is prepended as a small AML integer (bytes inserted at the front), and the whole buffer is wrapped with buffer opcode plus PkgLength.

   9.2. **Fixed 32-bit memory** descriptor encodes read/write vs read-only, base, and length in the ACPI short-form layout.

   9.3. **Address space descriptors** exist in 16-, 32-, and 64-bit variants (word, dword, qword). Each carries resource type (memory, I/O, bus number), generic flags (min/max fixed), type-specific flags (cacheability, read/write for memory; “entire range” for I/O), zero granularity, min, max, zero translation offset, and length computed as `max - min + 1`. Constructors reject `min > max`.

   9.4. **Legacy I/O port descriptor** encodes a 16-bit aligned range with alignment and length bytes for fixed ISA-style ports.

   9.5. **Extended IRQ descriptor** packs consumer/producer, edge/level, high/low, shared/exclusive flags, a count, and one or more IRQ numbers—used for line-based interrupt assignment in `_CRS`.

10. **Device, scope, method, and control flow**

   10.1. **Device** and **Scope** terms emit an outer opcode, PkgLength, then path followed by child terms—devices use the extended opcode pair; scopes use the scope opcode. Children can nest arbitrary AML inside.

   10.2. **Methods** include a path, argument count and serialized flag in a single method flags byte, then the method body terms. **Return** emits return opcode plus value.

   10.3. **Fields** describe bit-packed operation regions: access type (byte/word/dword/qword/buffer), update rules (preserve, write ones, write zeros), then a sequence of named bitfields or reserved regions encoded with internal PkgLength for field lengths.

   10.4. **Operation regions** bind a name to a space type (memory, I/O, PCI config, embedded controller, etc.) plus offset and length as AML integers.

   10.5. **Conditionals and loops**: `If` and `While` serialize predicate and body terms with PkgLength wrappers using the same helper as packages.

   10.6. **Comparison and arithmetic**: equality and less-than comparisons; a suite of binary operators (add, subtract, multiply, shifts, bitwise ops, string operations) emit opcode followed by operands and an ACPI “target” operand per AML’s three-operand form.

   10.7. **Arguments and locals** encode Arg0–Arg6 and Local0–Local7 single-byte opcodes; invalid indices are errors.

   10.8. **Store** emits store opcode, value, then target—note AML’s operand order in the wire format.

   10.9. **Mutex, Acquire, Release** support synchronized methods: mutex object with sync level, acquire with 16-bit timeout, release.

   10.10. **Notify** sends a notification value to an object term—used for plug events, device checks, etc.

   10.11. **Method calls** encode a path followed by argument terms without a separate call opcode—consistent with AML’s name-invocation syntax.

   10.12. **Buffers** embed raw bytes with a length prefix inside a buffer opcode and PkgLength.

   10.13. **CreateField** variants (dword vs qword) generate opcodes that alias subfields inside a buffer for dynamic resource editing.

11. **EISA identifiers and strings**

   11.1. **EISA IDs** compress a seven-character PNP-like string into a 32-bit little-endian quantity per the ACPI EISA naming rules (compressed characters and hex nybbles), then emit as a dword constant—used heavily for `_HID`.

   11.2. **Strings** emit the string opcode, UTF-8 payload bytes, and a null terminator.

12. **Integration picture**

```
  Guest physical memory
  +------------------+
  | RSDP             |----.
  +------------------+    |
                          v
  +------------------+    64-bit
  | XSDT             |<---'       points to ----> [ FADT | MADT | ... ]
  +------------------+
        |
        | (FADT.x_dsdt)
        v
  +------------------+
  | DSDT (AML blob)  |  <--- built from AML composables + ResourceTemplate
  +------------------+
```

   12.1. **Data flow**: the VMM chooses OEM strings and revision numbers, constructs individual tables (RSDP → XSDT → dependent tables), fills the FADT’s extended DSDT pointer, places each table at chosen aligned guest addresses, and populates the XSDT pointer array with those addresses.

   12.2. **Guest memory writes** are the final authority: checksums are computed before write for most tables; FADT recomputes on write to catch any late mutation. Address arithmetic uses checked addition when splitting header and payload across contiguous guest regions.

   12.3. **Testing philosophy** (internal to the crate): golden-byte comparisons ensure AML encoders match disassembler expectations for representative constructs—serial ports, memory windows, PCI-style resource sets, mutex patterns, and dynamic resource methods.

13. **Dependencies and wire-format safety**

   13.1. **Zerocopy-style derives** on packed structs guarantee that the in-memory layout used for SDT headers and fixed tables matches the on-the-wire ACPI layout when serialized as bytes—critical for RSDP, FADT, MADT entries, and headers.

   13.2. **Guest memory** abstraction decouples the table logic from how backing storage is implemented (mmap, etc.), so the same builders can run in different VMMs as long as the guest memory trait is satisfied.

14. **Operational constraints for integrators**

   14.1. **Table order and placement** are not dictated by the library; integrators must ensure guest physical addresses are reserved and not overwritten by other firmware data.

   14.2. **AML completeness** is the integrator’s responsibility: the DSDT must expose whatever devices the VMM emulates; missing `_CRS` or `_PRT` entries manifest as broken drivers, not as crate-level errors.

   14.3. **Versioning**: ACPI revision fields in each table should remain consistent with the features described—mismatch between FADT revision and actual AML capabilities can confuse operating systems’ capability gating.
