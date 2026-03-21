# Firecracker Python integration test system — architecture

## 1. Purpose and quality contract

1.1. The Python integration layer exists to exercise Firecracker end-to-end on real Linux/KVM hosts: it drives the VMM through the same HTTP API and host affordances that operators use, rather than linking against internal Rust APIs (those are covered separately by native tests in the Rust tree).

1.2. The suite encodes three overlapping contracts:

1.2.1. **Correctness** — guest-visible behavior (boot, virtio devices, networking, snapshots, CPU templates, metadata services, signals, logging) matches expectations across supported guest kernels and configuration dimensions.

1.2.2. **Security posture** — seccomp filters, jailer containment, dependency and vulnerability policy, and configuration that reduces attack surface behave as intended; some checks compare “before vs after” a change rather than against a static golden file.

1.2.3. **Performance and operability** — latency, throughput, boot time, memory overhead, and rate limiting stay within tracked bounds; long-running statistical comparisons can run outside the default pull-request gate.

1.3. Tests are designed to be **hermetic at the microVM boundary**: each test receives fresh processes, fresh network namespaces from a pool, and (by default) freshly built or explicitly supplied binaries. Failures trigger best-effort capture of host and guest-visible diagnostics into per-test artifact directories when structured reporting is enabled.

---

## 2. Pytest layering: how the runner is structured

2.1. **Configuration vs. code hooks.** The runner separates *declarative* defaults (verbosity, traceback style, default marker filter, JSON report location, cache location, default per-test time limit, logging format) from *imperative* session logic implemented in Python. Tests inherit a single global hook module that registers command-line options, augments reports, and defines the fixture graph. Subtrees can add narrower hook modules when a whole directory shares special setup without polluting unrelated tests.

2.2. **Session scope vs. function scope.** Three conceptual layers emerge:

2.2.1. **Process lifetime (session).** A unique temporary root is created under a well-known parent so concurrent CI jobs on shared hosts never collide. Session-scoped fixtures compile small native probes once, build network-namespace factories, and amortize expensive one-shot work across hundreds of tests.

2.2.2. **Test item lifetime (function).** Each test receives its own metrics scope, its own results directory keyed by node identity (including parametrization), and its own microVM factory handle. Teardown runs after every test even when setup failed partway, so leaked processes are bounded by single-test granularity.

2.2.3. **Autouse cross-cutting.** Some fixtures run for every test without being named: they attach host identity and docstring text to reports, and they establish the session directory. This keeps tests short while guaranteeing consistent observability.

2.3. **Phases and reporting.** The runner’s protocol has three phases per test item—setup, call, teardown. A custom hook wraps phase reporting so fixtures can consult whether the *call* phase failed when deciding to preserve diagnostics. Another hook streams per-phase duration and outcome into a metrics sink with rich dimensions (exact node id, coarse test name with parameters stripped, host kernel, CPU model, instance metadata). Distributed execution suppresses duplicate metric publication from worker processes so only the controller emits global aggregates.

2.4. **Parametrization as a design axis.** Fixtures are parametrized to express product dimensions explicitly: guest kernel glob matches, CPU template sets (none, static vendor templates, JSON-defined custom templates), snapshot kinds (full vs differential variants), and I/O engine modes. Indirect parametrization allows tests to override defaults like vCPU count or memory size without forking the fixture graph. Combining fixtures yields either **minimal** VMs (fixed kernel line, single root filesystem) or **broad** matrices (all kernels × template axis × boot vs restore constructors).

2.5. **Markers and gating.** Two marker families carve the suite: tests excluded from default continuous integration, and tests whose failure is non-blocking for merges. Everything unmarked is merge-critical. The default invocation applies the marker filter so local runs match CI unless the operator widens the selection.

2.6. **Time bounds.** A default per-test timeout prevents hung VMMs or stuck I/O from blocking pipelines indefinitely; specialized tests override with explicit extended limits for formal verification or exhaustive scenarios.

2.7. **Layering diagram (conceptual).**

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Declarative defaults (verbosity, timeouts, marker filter)   │
  └────────────────────────────┬────────────────────────────────┘
                               │
  ┌────────────────────────────v────────────────────────────────┐
  │  Session hooks: unique temp root, option registration,       │
  │  phase reports for fixtures, metrics on controller only       │
  └────────────────────────────┬────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         v                     v                     v
  ┌──────────────┐    ┌─────────────────┐    ┌──────────────┐
  │ Session      │    │ Function-scoped │    │ Autouse      │
  │ fixtures:    │    │ fixtures:       │    │ record props,│
  │ netns pool,  │    │ factory,        │    │ metrics      │
  │ helper bins  │    │ results_dir,    │    │ forwarding   │
  └──────────────┘    │ parametrized    │    └──────────────┘
                      │ kernels, etc.   │
                      └────────┬────────┘
                               │
                               v
                      ┌────────────────┐
                      │ Test body      │
                      │ (call phase)   │
                      └────────────────┘
```

---

## 3. Fixture architecture in the test tree

3.1. **Factory pattern for microVMs.** Tests do not construct processes by hand. A factory binds which Firecracker and jailer binaries to run, how network isolation is obtained from the namespace pool, and optional default CPU template injection from the command line. The factory tracks all VMs it created so teardown can terminate them deterministically in a defined order (monitors and SSH before signals, external backends before the VMM when both exist).

3.2. **Binary selection.** By default the factory uses release binaries produced by the Rust build for the host architecture. An override allows swapping in prebuilt pairs for regression testing across releases without recompiling the tree under test.

3.3. **Network namespace reuse.** Creating and destroying network namespaces for every test would be slow and can leave transient kernel state. A session-scoped pool hands out namespaces, tracks whether a namespace is still in use, and returns idle namespaces to the pool. Each parallel worker maintains its own pool keyed by worker identity to avoid cross-worker races.

3.4. **Guest kernel matrix.** Supported guest kernels are discovered from the artifact store using pattern and regex filters that include special cases (for example, legacy firmware paths that remain supported until policy removes them). Parametrization expands tests across all matched kernels so compatibility is continuously enforced. Narrow fixtures filter to a single line (for example one LTS branch) when the test does not need the full matrix.

3.5. **Root filesystem modes.** A read-only squashfs image is the default for broad compatibility tests; a writable ext4 image is used where tests must mutate the guest filesystem (for example, to inject helpers or collect logs) or where kernel debug variants are required. Debug kernel fixtures can point at an alternate artifact subtree without changing test code.

3.6. **CPU template coverage.** Static templates, repository-defined custom templates, and the absence of a template are treated as orthogonal axes. Template names are recorded as properties for reporting. Optional command-line injection allows repeating a suite against a single out-of-tree template without editing tests.

3.7. **Boot vs restore construction.** Many tests need equivalent coverage for VMs that were cold-booted versus VMs resumed from a snapshot: a shared constructor path builds a running VM, captures a snapshot, tears down the first process, and restores into a second process. A parametrized fixture selects which constructor function applies, so the same test body runs against both paths. This catches bugs in snapshot metadata, device reinitialization, and migration of guest-visible state.

3.8. **Dimension combinations.** “Minimal” fixtures fix kernel and rootfs to reduce Cartesian explosion for tests that do not need multi-kernel coverage. “Broad” fixtures intentionally multiply kernels, templates, and boot/restore paths to stress compatibility. Performance-oriented directories sometimes add fixtures that iterate shipped release binaries as parameters so compatibility and throughput are checked per artifact version.

3.9. **Failure artifact capture.** If the test body fails, the framework attempts to flush in-process metrics from each VM, copy host ring-buffer diagnostics, preserve SSH keys, copy select jailer-visible files from the chroot root, and persist console captures. This is best-effort: a crashed or wedged VMM might omit some artifacts.

3.10. **Fixture dependency sketch.**

```
                    guest_kernel ──┐
                                   ├──► build ──► spawn ──► configure ──► start
                    rootfs ────────┤         ▲                    │
                                   │         │                    │
                    cpu_template ──┘         └── snapshot path ───┘
                                              (optional restore)
```

---

## 4. MicroVM abstraction: lifecycle and control plane

4.1. **Process and jail.** Starting a microVM spawns the jailer (or equivalent isolation wrapper) so the VMM runs with constrained mount namespace, dropped privileges, and seccomp profile as production would. The abstraction tracks socket paths, chroot locations, and per-VM identifiers.

4.2. **HTTP over Unix domain sockets.** The control plane speaks REST-shaped JSON over a Unix socket. The client keeps a connection pool sized below the server’s advertised connection limit to avoid races where the server must reject new connections while tearing down old ones.

4.3. **Resource configuration.** VM objects expose composable configuration steps: boot source, machine sizing, drives, network interfaces, entropy, ballooning, logging, metrics, metadata service versions, CPU templates, huge-page settings, and snapshot-related flags. High-level helpers apply opinionated “basic configs” for common cases.

4.4. **Readiness layering.** After launch, readiness is not a single boolean. Depending on configuration, the framework may wait for the API socket, log lines indicating the API server, generic boot completion, or guest SSH when networking was configured—because SSH implies userspace and network stack readiness beyond “the VMM thread ran.” Interactive console modes swap daemonization for terminal attachment so serial traffic lands in a capturable stream.

4.5. **Guest interaction.** Beyond the API, tests reach the guest through SSH over the virtual NIC, virtio-vsock channels for host–guest tests, serial consoles for early-boot issues, and orchestrated terminal-multiplexer sessions for interactive debugging workflows.

4.6. **Snapshots.** Snapshot types distinguish full memory images from differential images that may require rebasing on a parent. Some modes require dirty-page tracking during execution; others use alternative memory tracking strategies. The abstraction carries disk and NIC metadata so a restored VM can reconstruct matching backend files.

4.7. **Host-side monitoring.** Optional monitors attach to Firecracker’s metrics endpoint and to host memory accounting where tests validate resource behavior under load.

4.8. **Lifecycle state machine (simplified).**

```
   configure API resources
            │
            v
      InstanceStart ──► optional SSH / serial gate
            │
            v
    running ──► pause ──► snapshot create
            │                    │
            │                    v
            │              tear down VM #1
            │                    │
            │                    v
            │              restore VM #2
            v                    │
         shutdown ◄──────────────┘
            │
            v
    teardown: close clients, kill backends, cgroup cleanup
```

---

## 5. Artifacts: what gets resolved before a test runs

5.1. **Guest kernels and root disks.** Kernels and filesystem images live under a machine-specific artifact directory with a documented fallback when shared storage is absent. Discovery uses glob and filter predicates so optional variants (for example ACPI-less builds) stay opt-in. Parametrized kernel fixtures record a normalized property string for reports.

5.2. **Firecracker release binaries.** Released builds can be globbed, parsed for semantic versions, and exposed as parameters capped to a supported range relative to the workspace version. The current workspace build can be published alongside releases by mirroring the same naming scheme so performance and compatibility tests treat “tip” like any other version.

5.3. **Session-built host utilities.** Small C programs are compiled once per session into the session temp directory: virtio-vsock ping-pong, MMIO config space mutation, instruction-set probes, synthetic timing binaries, and MSR readers. These exist because some behaviors are awkward to express purely in Python or require specific compiler flags.

5.4. **Rust workspace outputs.** Tests shell into the Rust build for seccomp demonstration binaries, snapshot manipulation tools, and example crates. The integration layer locates artifacts under the configured target output layout.

5.5. **Per-test results directories.** When JSON reporting is enabled, each test receives a unique directory derived from the report file location and the test’s node name (including parameters). Anything written there is intended for upload with the machine-readable report. Doctest items do not receive a directory.

5.6. **Reference payloads for static checks.** JSON documents model metadata service schemas, CPU template fingerprints per host flavor, custom template definitions, and tabular register lists. These are fixtures in the sense of *test inputs*, not pytest fixtures—they ground assertions without embedding large blobs in Python source.

---

## 6. Host assumptions and environmental contracts

6.1. **Privilege.** The suite expects superuser capability on the host: network namespaces, device access, cgroup manipulation, and performance tuning in specialized suites all assume elevated rights. The session aborts early if the effective user is not root.

6.2. **Hardware virtualization.** A working KVM device (or architecture-appropriate equivalent) is required. Without it, the design cannot run meaningful guests.

6.3. **Python runtime.** The tree targets a minimum language version; older interpreters fail fast at import time with an explicit error rather than obscure failures mid-session.

6.4. **Concurrency and shared hosts.** Default CI may share bare-metal hosts across jobs. Tests must not assume exclusive access to host-global performance knobs unless they live in suites scheduled on single-tenant machines. Parallel pytest workers introduce CPU contention; some timing-sensitive assertions are disabled when worker fan-out is detected.

6.5. **Network and cloud metadata.** Optional probes populate instance identifiers when the environment responds to cloud metadata protocols; reports include these fields when present so failures can be correlated with a specific machine image or instance.

---

## 7. Failure modes: what breaks and how it surfaces

7.1. **Timeouts.** The default per-test ceiling catches hung polls, deadlocked SSH, or VMMs that never reach readiness. Tests that legitimately need longer runs must declare extended limits explicitly.

7.2. **API and transport errors.** Control plane failures deserialize JSON error bodies where available and raise with explicit messages so tests fail loudly rather than continuing in a half-configured state.

7.3. **Guest-side failures.** SSH connection failures, command non-zero exits, and missing console patterns trigger test failures with the same diagnostic bundle as VMM errors where possible.

7.4. **Teardown races.** Kill ordering prefers stopping external backends and closing SSH sessions before sending fatal signals to the VMM, reducing socket and file descriptor races. Namespace and cgroup cleanup use bounded retries but cannot always fix a wedged kernel.

7.5. **Partial diagnostics.** After a failed call phase, artifact capture is best-effort: metrics flush may throw, the host log copy may be huge, and console buffers may be incomplete if the guest never reached userspace.

7.6. **Distributed execution quirks.** Metrics from workers are suppressed on the controller path; properties intended once per suite may be duplicated per test when suite-level recording is incompatible with worker processes.

7.7. **Failure triage flow (conceptual).**

```
  test failed
      │
      ├─► structured report + dimensions (which kernel, template, host)
      ├─► per-test results directory (if reporting enabled)
      ├─► flushed guest metrics (if API responsive)
      └─► host log excerpt + chroot file copies + console buffer
```

---

## 8. Functional tests vs. performance tests: different contracts

8.1. **Functional tests — behavioral truth.** These exercises end-to-end workflows: API sequences, virtio block and network stacks, vsock, balloon, entropy, RTC, serial, signals, logging, metadata services, CPU feature bits, userfaultfd-driven memory, snapshots and editing tools, and cross-kernel restore compatibility. Assertions are typically discrete: expected return codes, file contents, guest-visible state, or deterministic log substrings. The emphasis is *what the system does*, not how fast it does it under load.

8.2. **Performance tests — measurement and regression detection.** These measure boot time, network throughput, block I/O, vsock latency, memory overhead, rate limiters, huge-page effects, jailer overhead, snapshot restore latency, and process startup time. Many emit embedded metrics time series rather than hard-coded thresholds so results can be analyzed statistically in dedicated pipelines or compared across binary pairs.

8.3. **Threshold philosophy.** Functional tests generally use strict predicates. Performance tests often avoid brittle absolute SLOs in favor of recorded series, repeated samples, or external A/B orchestration that applies statistical tests to paired runs. Dimensions must uniquely identify parametrized cases so aggregation does not merge incompatible series.

8.4. **Workload isolation.** Performance tests are more sensitive to noisy neighbors and CPU frequency scaling; they may pin threads, warm caches, or repeat measurements. Functional tests may run with broader parallelism as long as correctness invariants hold.

8.5. **Comparison table.**

```
  Dimension          │ Functional              │ Performance
  ───────────────────┼────────────────────────┼──────────────────────────
  Primary question   │ Is behavior correct?    │ How much / how fast?
  Typical assertion  │ Equality, substring,    │ Metrics, distributions,
                     │ exit code                 │ statistical tests
  Flake sensitivity  │ Lower (discrete state)  │ Higher (timing, load)
  CI default         │ Often merge-blocking    │ May be scheduled or
                     │                         │ analyzed offline
```

---

## 9. Cross-cutting observability

9.1. **Host fingerprinting.** At session start, the framework captures a structured snapshot of the machine: CPU vendor and model strings, microcode, host kernel version tuple, libc, Rust toolchain, optional cloud instance metadata, and source control identity when available. These properties are duplicated into per-test reports to make flaky or environment-specific failures diagnosable after the fact.

9.2. **Structured reporting.** When JSON reporting is enabled, each test can write artifacts into a directory keyed by test identity; those directories are intended for upload alongside the machine-readable test report.

9.3. **Phase-aware metrics.** For each test item, metrics are emitted at the granularity of setup, call, and teardown, with dimensions that allow aggregation by exact node identifier, by test name without parameters, by host kernel, and globally. Duration and binary pass/fail counters are recorded per phase.

---

## 10. Other test families in the same tree

10.1. **Security.** Validates seccomp profiles (including custom profiles), jailer confinement, dependency auditing, and vulnerability baselines. Some tests are comparative: they only fail when a change introduces *new* risk relative to a baseline revision.

10.2. **Repository hygiene.** Static checks enforce formatting, OpenAPI drift, license headers, Markdown policy, Python style, and git commit message rules. These are fast, deterministic, and mostly host-independent.

10.3. **Build and analysis hooks.** A separate cluster of tests shells into the Rust toolchain for lints, unit tests, coverage thresholds, redundant seccomp rule detection, debugger scripts, and dependency graphs. They validate engineering constraints rather than runtime guest behavior.

10.4. **Formal methods.** Where enabled in heavy CI, a dedicated test invokes a Rust verifier across harnesses with long timeouts and captures the full log for archival.

---

## 11. Host-native helpers (compiled once per session)

11.1. **Compiled micro-utilities.** Small C programs are built once per session to probe behaviors that are awkward to express purely in Python: virtio-vsock ping-pong, MMIO config space mutation, instruction set features, synthetic jailer timing, and MSR reads. These binaries are linked or compiled with flags that force specific code generation when testing CPU features.

11.2. **Kernel-building tools.** Performance and correctness tests may shell out to compile ancillary eBPF or tracing tools when validating host-side semantics.

---

## 12. Reference data (inputs, not executable fixtures)

12.1. **Metadata service payloads** — Versioned JSON documents model the guest-visible metadata schema, including deliberately invalid fragments for negative testing.

12.2. **CPU template fingerprints** — For each supported host CPU flavor, canonical CPUID-derived fingerprints capture what the CPU template machinery should synthesize or mask; tests diff live behavior against these expectations.

12.3. **Custom template documents** — JSON definitions describe vendor-specific guest CPU models for Firecracker’s custom template feature, covering both x86 and AArch64 feature bundles.

12.4. **Model-specific register enumerations** — Tabular exports list model-specific registers exposed to guests for particular host/guest kernel combinations, enabling parity checks after changes to CPU modeling.

---

## 13. A/B and comparative testing philosophy

13.1. **Git A/B.** Some security and policy tests execute the same probe on two revisions: typically the merge base or main branch versus the candidate change. Equality of structured outputs means “no new regressions introduced by this diff”; divergence flags a genuine policy change worth human review.

13.2. **Binary A/B for performance.** Performance tests can emit metrics without asserting absolute SLOs. An external orchestrator runs the same pytest selection twice against two Firecracker builds, aligns metric series by dimensions, and applies non-parametric statistical tests to highlight regressions or improvements.

13.3. **Dimension discipline for metrics.** Embedded metrics carry dimensions that must uniquely identify parameterized cases; otherwise statistical aggregation merges incompatible series. Tests include parameters in dimension sets and use a stable performance-test key to map A and B runs.

13.4. **Local replay of analyses.** Emitting metrics to a local sink allows capturing reports for offline comparison between arbitrary environments, not only between compiled binaries.

---

## 14. Parallelism, isolation, and scheduling constraints

14.1. **Worker safety.** Distributed execution is supported, but not all tests are safe to interleave: those that rebuild heavy artifacts, mutate global host performance settings, or rely on exclusive hardware characteristics may require serial execution or dedicated hosts.

14.2. **Noisy neighbors.** Default CI may share bare-metal hosts across jobs; tests must not assume exclusive access to host-global knobs unless they live in suites that request single-tenant machines.

14.3. **Locking shared mutable operations.** Certain clone and artifact operations use file locks so parallel workers do not race when materializing shared binaries or reference trees.

---

## 15. End-to-end data flow (conceptual)

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
   │  templates,   │     │  Rust builds)  │      │               │
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

## 16. Relationship to other test layers

16.1. **Rust integration tests in the VMM crate** validate programmatic APIs and process lifecycle without HTTP; they complement this suite by covering surfaces tests here cannot see.

16.2. **Unit tests throughout the workspace** provide fast feedback; failures there are cheaper than reproducing issues through full KVM paths.

16.3. **Reusable Python framework.** The library that implements microVM construction, API clients, artifact resolution, snapshot serialization, and A/B helpers is documented in a companion architecture note scoped to that layer; this document stays at the breadth of the full Python integration tree including taxonomy and pytest orchestration.

---

## 17. Operational boundaries

17.1. Commands for containers, developer scripts, and bulk artifact download live outside this document by design; this note focuses on architecture and contracts rather than how to invoke a particular pipeline step.
