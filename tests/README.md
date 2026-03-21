# Firecracker Python integration test system — architecture

## 1. Purpose and quality contract

1.1. The Python integration layer exists to exercise Firecracker end-to-end on real Linux/KVM hosts: it drives the VMM through the same HTTP API and host affordances that operators use, rather than linking against internal Rust APIs (those are covered separately by native tests in the Rust tree).

1.2. The suite encodes three overlapping contracts:

1.2.1. **Correctness** — guest-visible behavior (boot, virtio devices, networking, snapshots, CPU templates, metadata services, signals, logging) matches expectations across supported guest kernels and configuration dimensions.

1.2.2. **Security posture** — seccomp filters, jailer containment, dependency and vulnerability policy, and configuration that reduces attack surface behave as intended; some checks compare “before vs after” a change rather than against a static golden file.

1.2.3. **Performance and operability** — latency, throughput, boot time, memory overhead, and rate limiting stay within tracked bounds; long-running statistical comparisons can run outside the default pull-request gate.

1.3. Tests are designed to be **hermetic at the microVM boundary**: each test receives fresh processes, fresh network namespaces from a pool, and (by default) freshly built or explicitly supplied binaries. Failures trigger best-effort capture of host and guest-visible diagnostics into per-test artifact directories when structured reporting is enabled.

---

## 2. Orchestration model (pytest-centric)

2.1. **Discovery and phases** — The runner treats each test function as an independent unit with explicit setup, execution, and teardown phases. Hooks attach global environment metadata to every result, record per-phase duration and outcome, and forward coarse metrics to a metrics sink (with suppression on worker processes when distributed execution is used).

2.2. **Gating semantics** — Two marker dimensions carve the suite:

2.2.1. Tests excluded from default continuous integration (scheduled or manual pipelines only).

2.2.2. Tests that run in optional pipelines: failures surface issues but do not block merges by default.

2.2.3. Everything unmarked is part of the merge-critical path.

2.3. **Session isolation** — Each pytest session allocates a unique temporary root under a well-known base path so concurrent jobs on shared bare-metal hosts do not collide. Session-scoped fixtures compile small native helpers once per session and reuse them across tests.

2.4. **Privilege and hardware assumptions** — The suite expects superuser capability on the host (namespaces, device access, performance tuning in specialized suites) and a working KVM device. Without these, large parts of the design cannot function.

2.5. **Time bounds** — A default per-test timeout prevents hung VMMs or stuck I/O from blocking pipelines indefinitely; individual long-running checks (formal verification invocations, exhaustive proofs) override with explicit extended limits.

---

## 3. Cross-cutting observability

3.1. **Host fingerprinting** — At session start, the framework captures a structured snapshot of the machine: CPU vendor and model strings, microcode, host kernel version tuple, libc, Rust toolchain, optional cloud instance metadata, and source control identity when available. These properties are duplicated into per-test reports to make flaky or environment-specific failures diagnosable after the fact.

3.2. **Structured reporting** — When JSON reporting is enabled, each test can write artifacts into a directory keyed by test identity; those directories are intended for upload alongside the machine-readable test report.

3.3. **Phase-aware metrics** — For each test item, metrics are emitted at the granularity of setup, call, and teardown, with dimensions that allow aggregation by exact node identifier, by test name without parameters, by host kernel, and globally. Duration and binary pass/fail counters are recorded per phase.

---

## 4. Fixture architecture and parametrization strategy

4.1. **Factory pattern for microVMs** — Tests do not construct processes by hand. A factory binds three concerns: which Firecracker and jailer binaries to run, how network isolation is obtained, and optional default CPU template injection. The factory tracks all VMs it created so teardown can terminate them deterministically.

4.2. **Binary selection** — By default the factory uses release binaries produced by the Rust build for the host architecture. An override path allows swapping in prebuilt pairs for regression testing across releases without recompiling the tree under test.

4.3. **Network namespace reuse** — Creating and destroying network namespaces for every test would be slow and can leave transient kernel state. A session-scoped pool hands out namespaces, tracks whether a namespace is still in use, and returns idle namespaces to the pool. Each parallel worker maintains its own pool to avoid cross-worker races.

4.4. **Guest kernel matrix** — Supported guest kernels are discovered from the artifact store using pattern and regex filters that include special cases (for example, legacy firmware paths that remain supported until policy removes them). Parametrization expands tests across all matched kernels so compatibility is continuously enforced.

4.5. **Root filesystem modes** — A read-only squashfs image is the default for broad compatibility tests; a writable ext4 image is used where tests must mutate the guest filesystem (for example, to inject helpers or collect logs) or where kernel debug variants are required.

4.6. **CPU template coverage** — Static templates, repository-defined custom templates, and the absence of a template are treated as orthogonal axes. Template names are recorded as properties for reporting. Optional command-line injection allows repeating a suite against a single out-of-tree template without editing tests.

4.7. **Boot vs restore construction** — Many tests need equivalent coverage for VMs that were cold-booted versus VMs resumed from a snapshot: a shared constructor path builds a running VM, captures a snapshot, tears down the first process, and restores into a second process. This catches bugs in snapshot metadata, device reinitialization, and migration of guest-visible state.

4.8. **Dimension combinations** — “Minimal” fixtures fix kernel and rootfs to reduce Cartesian explosion for tests that do not need multi-kernel coverage. “Broad” fixtures intentionally multiply kernels, templates, and boot/restore paths to stress compatibility.

4.9. **Failure artifact capture** — If the test body fails, the framework attempts to flush in-process metrics from each VM, copy host dmesg, preserve SSH keys, copy select jailer-visible files, and persist console captures. This is best-effort: a crashed or wedged VMM might omit some artifacts.

---

## 5. MicroVM abstraction: lifecycle and control plane

5.1. **Process and jail** — Starting a microVM spawns the jailer (or equivalent isolation wrapper) so the VMM runs with constrained mount namespace, dropped privileges, and seccomp profile as production would. The abstraction tracks socket paths, chroot locations, and per-VM identifiers.

5.2. **HTTP over Unix domain sockets** — The control plane speaks REST-shaped JSON over a Unix socket. The client keeps a connection pool sized below the server’s advertised connection limit to avoid races where the server must reject new connections while tearing down old ones.

5.3. **Resource configuration** — VM objects expose composable configuration steps: boot source, machine sizing, drives, network interfaces, entropy, ballooning, logging, metrics, MMDS versions, CPU templates, huge-page settings, and snapshot-related flags. High-level helpers apply opinionated “basic configs” for common cases.

5.4. **Guest interaction** — Beyond the API, tests reach the guest through SSH over the virtual NIC, virtio-vsock channels for host–guest tests, serial consoles for early-boot issues, and orchestrated `tmux`/`screen` sessions for interactive debugging workflows.

5.5. **Snapshots** — Snapshot types distinguish full memory images from differential images that may require rebasing on a parent. Some modes require dirty-page tracking during execution; others use alternative memory tracking strategies. The abstraction carries disk and NIC metadata so a restored VM can reconstruct matching backend files.

5.6. **Host-side monitoring** — Optional monitors attach to Firecracker’s metrics endpoint and to host memory accounting where tests validate resource behavior under load.

---

## 6. Host-native helpers and build integration

6.1. **Compiled micro-utilities** — Small C programs are built once per session to probe behaviors that are awkward to express purely in Python: virtio-vsock ping-pong, MMIO config space mutation, instruction set features, synthetic jailer timing, and MSR reads. These binaries are linked or compiled with flags that force specific code generation when testing CPU features.

6.2. **Rust workspace integration** — Helpers invoke `cargo` to build examples (for seccomp demonstrations), snapshot manipulation tools, and to locate target binaries under the configured musl release layout.

6.3. **Kernel-building tools** — Performance and correctness tests may shell out to compile ancillary eBPF or tracing tools when validating host-side semantics.

---

## 7. Reference data (fixtures, not code)

7.1. **Metadata service payloads** — Versioned JSON documents model the guest-visible metadata schema, including deliberately invalid fragments for negative testing.

7.2. **CPU template fingerprints** — For each supported host CPU flavor, canonical CPUID-derived fingerprints capture what the CPU template machinery should synthesize or mask; tests diff live behavior against these expectations.

7.3. **Custom template documents** — JSON definitions describe vendor-specific guest CPU models for Firecracker’s custom template feature, covering both x86 and AArch64 feature bundles.

7.4. **Model-specific register enumerations** — Tabular exports list model-specific registers exposed to guests for particular host/guest kernel combinations, enabling parity checks after changes to CPU modeling.

---

## 8. Integration test taxonomy (behavioral layers)

8.1. **Functional** — Exercises API workflows, virtio block and network stacks, vsock, balloon, entropy, RTC, serial, signals, logging, MMDS, CPU feature bits, UFFD-driven memory, snapshots and editors, and cross-kernel restore compatibility. These tests prioritize behavioral correctness over raw speed.

8.2. **Performance** — Measures boot time, network throughput, block I/O, vsock latency, memory overhead, rate limiters, huge-page effects, jailer overhead, snapshot restore latency, and process startup time. Many emit embedded metrics time series rather than hard-coded thresholds so results can be analyzed statistically in dedicated pipelines.

8.3. **Security** — Validates seccomp profiles (including custom profiles), jailer confinement, dependency auditing, and vulnerability baselines. Some tests are comparative: they only fail when a change introduces *new* risk relative to a baseline revision.

8.4. **Repository hygiene** — Static checks enforce formatting, OpenAPI drift, license headers, Markdown policy, Python style, and git commit message rules. These are fast, deterministic, and mostly host-independent.

8.5. **Build and analysis hooks** — A separate cluster of tests shells into `cargo` for Clippy, unit tests, coverage thresholds, redundant seccomp rule detection, GDB scripts, and dependency graphs. They validate engineering constraints rather than runtime guest behavior.

8.6. **Formal methods** — Where enabled in heavy CI, a dedicated test invokes the Kani Rust verifier across harnesses with long timeouts and captures the full log for archival.

---

## 9. A/B and comparative testing philosophy

9.1. **Git A/B** — Some security and policy tests execute the same probe on two checkouts: typically the merge base or main branch versus the candidate change. Equality of structured outputs means “no new regressions introduced by this diff”; divergence flags a genuine policy change worth human review.

9.2. **Binary A/B for performance** — Performance tests can emit metrics without asserting absolute SLOs. An external orchestrator runs the same pytest selection twice against two Firecracker builds, aligns metric series by dimensions, and applies non-parametric statistical tests to highlight regressions or improvements.

9.3. **Dimension discipline for metrics** — Embedded metrics carry dimensions that must uniquely identify parameterized cases; otherwise statistical aggregation merges incompatible series. Tests include parameters in dimension sets and use a stable performance-test key to map A and B runs.

9.4. **Local replay of analyses** — Emitting metrics to a local sink allows capturing reports for offline comparison between arbitrary environments, not only between compiled binaries.

---

## 10. Parallelism, isolation, and scheduling constraints

10.1. **Worker safety** — Distributed execution is supported, but not all tests are safe to interleave: those that rebuild heavy artifacts, mutate global host performance settings, or rely on exclusive hardware characteristics may require serial execution or dedicated hosts.

10.2. **Noisy neighbors** — Default CI may share bare-metal hosts across jobs; tests must not assume exclusive access to host-global knobs unless they live in suites that request single-tenant machines.

---

## 11. End-to-end data flow (conceptual)

```
                    ┌─────────────────────────────────────┐
                    │  Test runner session + global props │
                    └──────────────┬──────────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           v                       v                       v
   ┌───────────────┐     ┌────────────────┐      ┌───────────────┐
   │ Fixture graph │     │ Host helpers   │      │ Metrics +     │
   │ (kernels,     │     │ (C utilities,  │      │ JSON reports  │
   │  templates,   │     │  cargo tools)  │      │               │
   │  snapshots)   │     └────────┬───────┘      └───────────────┘
   └───────┬───────┘              │
           │                      │
           v                      v
   ┌───────────────────────────────────────────┐
   │ MicroVM factory → jailer + Firecracker    │
   │ HTTP API configuration + guest probes     │
   └───────────────────────┬───────────────────┘
                           │
                           v
                  ┌────────────────┐
                  │ Guest workload │
                  │ (SSH, vsock,   │
                  │  virtio I/O)   │
                  └────────────────┘
```

---

## 12. Relationship to other test layers

12.1. **Rust integration tests in the VMM crate** validate programmatic APIs and process lifecycle without HTTP; they complement this suite by covering surfaces tests here cannot see.

12.2. **Unit tests throughout the workspace** provide fast feedback; failures there are cheaper than reproducing issues through full KVM paths.

---

## 13. Further reading

13.1. The reusable Python framework that implements microVM construction, API clients, artifacts, and A/B helpers is documented in a dedicated companion note under that tree for deeper implementation detail.

13.2. Operational commands (container entrypoints, developer scripts, artifact downloads) live at the repository tooling layer and intentionally stay out of this architecture note.
