# Snapshot Format, Version Negotiation, and Persistence Integration

## Purpose & Boundaries

This subsystem is responsible for turning in-memory virtual machine state into a portable byte sequence that can be written to durable storage and later read back, and for validating those bytes well enough that accidental corruption or incompatible evolution does not silently produce a running system built on garbage. It sits at the boundary between higher-level state objects that know what must be preserved and the raw input and output streams provided by the rest of the process. Its core job is to define a stable on-disk layout, to apply a consistent binary encoding policy for structured data, and to attach enough metadata and integrity checks that consumers can decide whether a blob is meant for this build and this machine class before they commit to interpreting the payload.

What it does not do is define what “the virtual machine” means in domain terms. The version string carried in every file is supplied by callers; this layer does not own product versioning or release numbering. It also does not implement placement policy for files on disk, file locking, crash-consistent multi-file transactions, or network transfer. Those belong to orchestration and storage layers above or beside it. Similarly, it does not decide how individual devices or subsystems serialize their own nested structures beyond providing generic encode and decode helpers with shared limits and endianness rules.

If this component failed entirely, snapshots could not be produced or consumed in a uniform way. The hypervisor would lose a single, shared recipe for packaging state, and every caller would need to reinvent headers, integrity checks, and compatibility rules, almost certainly inconsistently. Recovery from host maintenance, migration workflows, and debugging by replaying saved state would all become fragile or impossible without coordinated replacement work across the tree.

## Interfaces & Contracts

Externally, the subsystem exposes a small set of conceptual operations: create a snapshot writer bound to a semantic version that the caller asserts describes the payload schema; write a self-describing blob including an architecture-specific magic value, the version, the encoded state, and optionally a trailing checksum; read a format version back from an existing stream without fully decoding the payload; load and validate a snapshot with checksum verification; load with an additional compatibility check against the reader’s own supported version; and encode or decode arbitrary structured values using the same binary rules as the snapshot body, so that nested pieces of state stay consistent with the container.

Consumers must supply byte streams that honor read and write semantics, and they must know the total size of an existing snapshot when validating integrity, because the integrity mechanism is expressed as a suffix whose position depends on the length of the preceding material. Callers also choose the semantic version written into the header; the subsystem treats that triple as opaque except for the compatibility predicate described later.

The subsystem consumes generic structured data capabilities: anything that can be flattened to the shared encoding and reconstructed on read may be used as the state object. It depends on a standard semantic-versioning notion for comparing triples. For integrity, it depends on a sixty-four-bit cyclic redundancy implementation that matches the reader and writer wrappers described below. Architecture identity is encoded as distinct magic values for different CPU families so that a blob produced for one architecture cannot be mistaken for another without explicit handling.

Inbound invariants include that the caller’s stated version on save reflects the actual schema of the encoded object, and that concurrent writes to the same output are avoided. On load, the caller must not truncate the stream length used for integrity checking. Outbound guarantees are weaker than full semantic safety: successful decode proves the magic matches the running architecture, the checksum matches the bytes if that path is used, and optional version policy passes. It does not prove that the higher-level state is internally consistent; that remains the responsibility of the types and restore logic built on top.

## Data Flow

Data enters as structured state in memory together with a semantic version chosen by the caller. On the save path, the writer first emits a fixed-width magic integer, then encodes the version triple using the same binary rules as the rest of the file, then encodes the state object. When the full save with integrity is used, all of those bytes pass through a streaming checksum accumulator before the computed sixty-four-bit value is appended as the final encoded field. The on-wire layout is therefore a header-like prefix, a body whose layout is entirely determined by the schema of the state object, and a small trailer for integrity.

The following diagram summarizes the save path from application state to bytes on the wire.

```
  +------------------+     +------------------+
  | In-memory state  |     | Caller-supplied  |
  | object + schema  |     | semantic version |
  +--------+---------+     +--------+---------+
           |                        |
           v                        v
  +--------------------------------------------------+
  | Shared binary encoder (fixed-width integers,     |
  | little-endian, bounded allocation on decode)     |
  +------------------------+-------------------------+
                           |
                           v
  +--------------------------------------------------+
  | Optional checksum wrapper (write-through hash)   |
  +------------------------+-------------------------+
                           |
                           v
  +--------------------------------------------------+
  | Output byte sink (memory buffer, file, socket)   |
  +--------------------------------------------------+
```

On the load path with integrity, the total byte length is used to partition the stream. Everything except the final sixty-four-bit checksum is read into a buffer while a running checksum is updated. The stored checksum value is then read and compared to the computed value. Only after equality is established does the implementation reinterpret the buffered prefix through an inner decode sequence that assumes integrity was already verified: magic and version are read first, magic is compared to the expected architecture value, then the state object is decoded from the remainder of the buffer. This ordering matters because the checksum covers exactly the bytes that correspond to magic, version, and payload, but not the stored checksum itself.

The diagram below shows the logical stages for loading with integrity.

```
  +------------------+     Total length of snapshot
  | Incoming bytes   |------------------------------+
  +------------------+                              |
           |                                        v
           v                          +------------------------------+
  +------------------+                | Read N-8 bytes into buffer   |
  | Checksum stream  |--------------->| while updating rolling hash |
  +------------------+                +--------------+---------------+
           |                                         |
           v                                         v
  +------------------+                +------------------------------+
  | Read stored      |                | Compare computed vs stored   |
  | checksum field   |                | hash; abort on mismatch    |
  +------------------+                +--------------+---------------+
                                                      |
                                                      v
                                    +----------------------------------+
                                    | Decode header: magic + version   |
                                    +------------------+---------------+
                                                       |
                                                       v
                                    +----------------------------------+
                                    | Decode payload object            |
                                    +----------------------------------+
```

When callers only need the format version, they perform a partial decode of the header without traversing the full payload, which is cheaper and does not require the full object type to be known up front.

## Control Flow

Execution is always driven by an explicit caller invoking a save or load operation on a byte-oriented channel. There is no background thread or subscription model inside this layer. The critical path for saving begins with binding a snapshot handle to a semantic version, then optionally wrapping the output in a checksum-aware writer, emitting header and body, and finally emitting the checksum when required. The critical path for loading with integrity begins with length validation, buffered read of the checksummed region, checksum comparison, then header validation and payload decode.

Branching appears at several well-defined decision points. If the caller-provided total length cannot accommodate even the checksum field, processing stops before reading large buffers, which avoids pointless allocation on obviously truncated inputs. If the checksum does not match, the implementation returns an error that carries the computed value for diagnosis without attempting to interpret the payload. If the magic does not match the expected architecture constant, loading stops with an invalid identity error even if the remainder might decode under a different policy. When a version-aware load is requested, the implementation compares the on-disk semantic triple to the caller’s supported triple under a specific compatibility rule: the major numbers must match exactly, and the minor number in the file must not exceed the minor number of the loader’s supported version. Patch level differences within an accepted major and minor band are tolerated, including patches larger than the loader’s own patch component. If that test fails, the error surfaces the version found in the file.

The optional persistence-oriented abstraction in this subsystem is purely contractual: it describes components that can export a pure state value and later be reconstructed from that value together with external constructor inputs. Control flow involving that pattern lives in higher layers; here it is only a shared vocabulary for how subsystems participate in capturing and restoring state without dictating transport or file layout.

## State & Lifecycle

At runtime, the snapshot helper holds very little durable state of its own. A handle created for saving remembers only the semantic version it is authorized to stamp into new files. It does not cache open files or maintain internal buffers beyond what the encoding and checksum layers need for transient work. Decoding operations are stateless aside from stream position on the caller-provided reader.

Lifecycle phases are straightforward. During initialization of a save operation, the version binding is fixed for the lifetime of that handle. During steady-state encoding, repeated writes append to the stream without implicit reset. On the load side, each call consumes the relevant portion of the reader; callers must present a fresh positioned stream if they intend to read multiple snapshots from one file, which is not a typical pattern for this format.

Recovery is not a first-class feature inside this layer. If an interrupted write leaves a partial file, the integrity trailer will be missing or inconsistent, and a subsequent load will fail closed. Higher layers may implement atomic rename patterns or write-to-temporary-then-move semantics; those policies are outside this boundary.

## Failure Modes

I/O errors propagate with operating-system error codes preserved where possible, because the root cause may be permission, media, or network-backed storage issues. Serialization errors are wrapped into a generic string-carrying error to avoid leaking dependency details upward while still giving operators a breadcrumb in logs. Checksum mismatch is treated as a hard failure: the implementation does not attempt partial recovery or best-effort decode. Magic mismatch similarly fails immediately, preventing accidental cross-architecture misuse from proceeding to typed decoding.

Version mismatch is explicit and carries the version read from the file so callers can branch on policy, emit user-facing guidance, or attempt a different code path. The design intentionally rejects newer minor versions on older loaders, which avoids silent omission of fields that older code cannot interpret. Conversely, older minor versions on newer loaders are accepted within the same major line, which assumes forward compatibility at the schema level for omitted data and defaulting rules implemented above this layer.

Silent corruption is mitigated but not impossible. If a payload is internally inconsistent yet still decodes, this layer will not detect logical contradictions. If a caller disables integrity on a code path that still uses the shared encoder, bit flips might survive until higher-level validation. The checksum path closes the gap for on-disk tampering and many storage faults for the guarded layout. Assumptions that can lead to subtle issues include mismatched total length arguments from the caller, which could cause checksum verification to operate on the wrong span, and reliance on patch-level flexibility where upstream schema evolution might still require semantic checks beyond numeric version comparison.

## Operational Characteristics

Resource use is dominated by the size of the snapshot payload plus a modest constant overhead for headers and checksums. During integrity-protected loads, the implementation allocates a contiguous buffer large enough to hold the entire checksummed prefix so it can be validated before decoding. Very large snapshots therefore imply a corresponding peak memory reservation for that copy, independent of streaming decode of the payload after validation. The binary decoder applies an upper bound on how much memory it will allocate while interpreting structured data, which limits certain denial-of-service patterns via maliciously crafted inputs at the cost of rejecting legitimately huge single fields unless limits are revisited alongside product needs.

Scaling bottlenecks are unlikely inside this layer in isolation; it is strictly single-threaded per call and CPU-bound on encoding and hashing. Throughput scales with clock speed and memory bandwidth writing the buffer. Observability is minimal at this layer: errors bubble up without mandatory logging here, so operators depend on surrounding services to record failures, sizes, and durations. No metrics are embedded by default.

## Design Rationale

The layout separates identity, versioning, payload, and integrity deliberately. The magic number answers “is this even for us and this CPU family?” before any schema-specific work happens. The semantic version answers “is this snapshot within the compatibility window for this build?” without conflating that question with hypervisor package versions. Keeping the version caller-defined allows the rest of the project to move the schema number independently of the library that implements the container format.

Checksums append rather than prepend so that writers can stream header and body through a single pass while accumulating a hash, then finalize. On read, the two-phase process—hash the bytes, then decode from memory—trades an extra copy for a clear failure boundary before typed interpretation. The alternative of streaming decode while checking integrity would reduce memory but tightly couples parsing order with integrity, which is harder to reason about when deserialization itself consumes bytes unpredictably.

Version negotiation is intentionally conservative on minor numbers because minor bumps are the project’s practical channel for additive schema changes that older readers might not understand. Major changes break the world explicitly. Patch is not part of the inequality test because patch is treated as non-semantic for compatibility decisions in this policy, relying on schema designers not to introduce incompatible changes under the same major and minor labels.

The small persistence abstraction complements this container by encouraging each subsystem to expose a pure state projection and a reconstruction entry point. The snapshot format is then just the envelope; the persistence contract is the agreement about how domain objects participate in migration without tying them to a particular file name or storage backend. Together, the envelope and the contract let higher layers compose a full virtual machine snapshot from many pieces while keeping this subsystem focused on bytes, versions, and integrity.

## Diagram: Conceptual Topology

The next figure places the snapshot container relative to domain state and storage. Edges represent data movement, not compile-time dependencies.

```
  +-------------------+       +------------------------+
  | Device models,    |       | Snapshot container     |
  | memory tables,    | save  | (magic, version,       |
  | vCPUs, etc.       +------>| encode, checksum)      |
  +-------------------+       +-----------+------------+
                                          |
                                          v
                              +-------------------------+
                              | Durable medium or       |
                              | transport buffer        |
                              +-------------------------+
                                          |
                                          | load
                                          v
                              +-------------------------+
                              | Restore orchestration   |
                              | using persisted state   |
                              | and constructor inputs  |
                              +-------------------------+
```

Before the diagram, recall that orchestration decides when to persist each participating component’s state and how to thread constructor arguments on restore. After it, note that the snapshot container never sees those constructor details; it only sees the encoded aggregate the orchestration chooses to write.

## Diagram: Version Check Decision

Compatibility is evaluated after integrity and magic checks succeed.

```
                    +------------------+
                    | File version (F) |
                    | Loader max (L)   |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Majors equal?    |
                    +--+-----------+---+
                       | no         | yes
                       v            v
                 +---------+  +------------------+
                 | Reject  |  | F.minor <= L.minor? |
                 +---------+  +----+---------+-----+
                                   | no       | yes
                                   v          v
                             +---------+  +---------+
                             | Reject  |  | Accept  |
                             +---------+  +---------+
```

Before this diagram, remember that rejection returns the file’s version to aid messaging. After it, note that patch is not illustrated on the graph because it does not gate acceptance once major and minor rules pass.

---

This document focuses on architectural roles, data shapes, and policy. For exact on-disk bit layouts and type identities, refer to the implementation and its tests, which exercise magic tampering, checksum failure, truncated files, and version edge cases deliberately.
