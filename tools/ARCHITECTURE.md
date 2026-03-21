# Developer Tooling, Continuous Integration Automation, and Supporting Static Resources

This document describes the architecture of the repository’s tooling layer: the scripts and container assets that developers and automation use to build, test, release, and regenerate artifacts, together with the separate tree of static policy and configuration resources that those workflows consume or produce. The tooling layer is deliberately thin orchestration over heavyweight upstream tools such as the Rust toolchain, the Python test stack, container runtimes, and object tooling. The static resources tree holds durable inputs that shape security posture, guest environments, and reproducible CI payloads. The two areas are analyzed together because production builds and integration tests do not exist in isolation from those static inputs: the same policies and guest materials that ship with the repository also anchor what automated tests assume about syscall surfaces, kernel behavior, and guest filesystem layout.

## Purpose & Boundaries

The responsibility of this combined system is to provide a stable, repeatable bridge between a developer’s machine or a CI agent and the project’s actual build and verification work. It establishes the canonical development container image, wires host directories for persistent caches, invokes compilation with the intended libc and optimization profile, stages integration tests with the right privileges and cgroup semantics, and automates release packaging and publication steps. It also owns auxiliary workflows such as regenerating foreign-function bindings from operating-system headers, preparing downloadable test fixtures, comparing performance-oriented test runs across revisions, and offering interactive debugging surfaces for the hypervisor under test.

What this system explicitly does not do is implement the hypervisor itself. It does not define guest workloads beyond the narrow provisioning needed for tests, and it does not replace infrastructure-level concerns such as secrets management for real deployments, fleet-wide monitoring, or production rollout orchestration beyond what is needed for publishing release artifacts. If the tooling layer failed wholesale, developers could still attempt local compilation without the prescribed container, but they would lose the pinned toolchain, consistent dependency versions, and the scripted guardrails that keep contributions mergeable. Continuous integration would fragment: tests would require ad hoc host setup, artifact hashes would drift, and security-sensitive policies would be harder to validate uniformly. If the static resources tree were missing or inconsistent, builds might still succeed, but syscall filtering could diverge from what maintainers expect, guest kernels used in tests would not match the configurations assumed by the test harness, and root filesystem images would not exhibit the networking and initialization properties that integration tests rely upon.

## Interfaces & Contracts

The tooling layer exposes its capabilities primarily through a single command-oriented entry surface that maps high-level verbs such as building, testing, opening a shell in the development environment, formatting sources, checking style, installing binaries, downloading or building CI fixtures, and running release automation. Those verbs accept a small set of cross-cutting flags for unattended operation and forward additional arguments to the underlying tools where appropriate. A parallel family of shell scripts performs focused tasks: one drives the Rust build with profile and libc selection and optional release artifact shaping; another launches the Python-based integration suite inside the container with environment and ulimit settings suited to nested virtualization; others handle artifact download and post-processing, semantic version bumps, tagging, and packaging for distribution hosts.

The development container image is itself a contract. It advertises a pinned operating-system base, a pinned Rust toolchain with multiple compilation targets, Python tooling installed into an isolated virtual environment, ancillary native tools needed by tests or helpers, and optional cross-compilation support so architecture-specific checks can run from a single image flavor. Host-to-container contracts include bind-mounted source and build trees, persistent package registry caches on the host to avoid refetching dependencies on every invocation, pass-through of common proxy environment variables, and selective privilege escalation when tests must manipulate control groups, network namespaces, or the hardware virtualization interface.

The static resources tree exposes contracts of a different kind. Architecture-specific syscall policy documents describe allowed and denied system calls for major components of the stack; these are consumed as authoritative inputs to compilation or validation steps that ensure the running binary’s behavior stays within the declared surface. Guest kernel configuration fragments are layered to produce kernels used in CI: a shared baseline augmented by feature-specific fragments for tracing, debugging, or particular hardware assumptions. A separate overlay describes files that become part of guest root filesystem images, including service unit configuration for automated serial console login, a small custom networking bootstrap aligned with how tests assign addresses, and optional sysctl hardening. Small native programs are compiled as part of artifact generation and embedded into those images to support memory and fault-injection style tests. Together, these artifacts define what “the official guest” means for automated testing.

Callers of the tooling are expected to provide a working container runtime, sufficient disk space for caches and images, and on hosts that run integration tests, access to the virtualization device with appropriate permissions. For artifact download workflows, a object-storage client is required. The tooling in turn guarantees deterministic paths for outputs under the build tree, consistent ownership repair after privileged container runs, and clear failure messages when prerequisites are absent.

## Data Flow

Source code enters the system from the repository checkout bind-mounted into the container. The Rust build reads source and lockfiles from the workspace and writes object code and binaries into a target directory that is also bind-mounted so incremental builds persist. The Python test suite reads test definitions and shared harness libraries, launches the built hypervisor binaries, drives virtual machines using prepared kernel and disk images, and writes reports and logs under a test results area. Environment metadata from the automation system may be captured into a file for propagation into the container so telemetry and coverage integrations behave consistently.

The static resources participate at multiple points in this flow. Syscall policy data flows from the resources tree into the build, where it is transformed or embedded so that runtime behavior matches the declared filter. Guest kernel configuration flows from layered fragments into a kernel source checkout, through the standard kernel build, and out to versioned kernel binaries plus accompanying configuration dumps for debugging. Root filesystem generation flows from an overlay merged into a staging tree, through package installation and service configuration inside an isolated chroot or nested container, into compressed filesystem images and optional writable disk image formats. A post-processing step may inject host-generated authentication material into those images so tests can log in without embedding long-lived secrets in the repository.

Downstream, CI artifact download moves immutable blobs from object storage into a local cache keyed by major version, after which a setup step adjusts permissions, injects keys, regenerates compressed images, and materializes writable volumes where the suite expects them. That closes the loop between remotely published fixtures and the exact layout the Python tests probe.

The following diagram summarizes major data movement without naming individual components.

```
┌─────────────────┐     bind mount      ┌──────────────────────────┐
│  Host checkout  │ ──────────────────► │  Development container   │
│  (sources)      │                     │  (toolchain, Python venv) │
└────────┬────────┘                     └───────────┬──────────────┘
         │                                            │
         │  read/write                                │  emits
         ▼                                            ▼
┌─────────────────┐       ┌───────────────┐    ┌─────────────────┐
│  Build tree     │◄──────│  Rust compile │    │  Binaries and   │
│  (caches, bins) │       │  and linkage  │    │  debug splits   │
└────────┬────────┘       └───────────────┘    └────────┬────────┘
         │                                              │
         │  consumed by                                 │
         ▼                                              ▼
┌─────────────────┐       ┌───────────────┐    ┌─────────────────┐
│  Static policy  │──────►│  Policy sync  │    │  Integration    │
│  and guest      │       │  with build   │    │  tests and      │
│  configuration  │       └───────────────┘    │  metrics        │
└─────────────────┘                            └────────┬────────┘
         │                                              │
         │  packaged for CI                             ▼
         └──────────────────────────────────────►  Archived logs
                                                    and reports
```

Before the diagram, the important idea is that sources and caches live on the host for persistence while compilation and tests execute in the container for hermeticity. After the diagram, note that static policy and guest configuration are not merely documentation: they are inputs that flow into binaries, kernels, and disk images that tests then exercise, so drift in the static tree shows up as concrete behavioral changes in CI.

## Control Flow

Execution is triggered interactively when a developer invokes the orchestration entrypoint, and automatically when continuous integration agents run the same commands with unattended flags. The typical critical path for verification begins by ensuring the container image is present locally, pulling it if needed with retries. The build path then ensures writable cache directories exist, runs the compilation driver inside a privileged container to avoid permission mismatches on outputs, and normalizes file ownership back to the invoking user. The test path optionally rebuilds in release mode, fetches version-aligned remote artifacts if they are not already cached, applies host kernel mitigations when specific kernel versions are detected, and may pin CPU and memory nodes for repeatability. It then launches a privileged container with relaxed seccomp constraints around the test harness, because nested virtualization and jail setup require capabilities that default container profiles disallow.

Branching occurs along several axes. Debug versus release compilation selects optimization and strip behavior. Choice of libc flavor selects which target triple to build and link against. Performance-oriented test modes branch into additional host tuning such as disabling dynamic frequency scaling and allocating large hugepage pools, with cleanup restored afterward. Optional comparative testing swaps the simple sequential runner for a harness that executes the same tests against binaries built from different revisions and performs statistical comparisons on structured metrics logs. Release flows branch separately: they validate versioning discipline against API description files, enforce branch naming expectations, create signed-off artifacts, and push tags or publish packages through provider-specific helpers.

Artifact regeneration flows are heavier. They may spawn nested daemons to build root filesystems inside ephemeral downstream containers, compile small helper programs for the target image, assemble initramfs-style payloads for minimal boot testing, and compile Linux from upstream sources using layered configuration fragments. These flows are not on every developer’s critical path but are central to maintainers refreshing what CI consumes.

A compact control-flow sketch follows.

```
                    ┌─────────────┐
                    │  Invocation │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Build   │ │   Test   │ │  Release │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Cache   │ │  Fetch   │ │  Version │
        │  prep    │ │  fixtures│ │  checks  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Compile │ │  Pytest  │ │  Publish │
        │  Rust    │ │  driving │ │  artifacts│
        └──────────┘ │  VMs     │ └──────────┘
                     └──────────┘
```

Prior to this figure, control begins with a single dispatch point that routes to purpose-specific pipelines. Following it, the build pipeline emphasizes deterministic compilation outputs, the test pipeline emphasizes privileged device access and fixture readiness, and the release pipeline emphasizes policy checks on version alignment and artifact completeness.

## State & Lifecycle

Persistent state is intentionally split. Long-lived caches for package registries and incremental compilation outputs live under the build tree on the host so repeated invocations amortize network and compile costs. Ephemeral state inside containers is discarded on exit; a memory-backed workspace may be used for high-churn temporary files during tests to avoid exhausting slower disks. The development image may be removed by a deep clean command, treating the container as disposable while preserving or wiping caches depending on operator choice.

Test runs own transient state in a dedicated results directory that may be archived wholesale in automation for upload. Performance-oriented modes temporarily alter host kernel parameters such as hugepage reservations and CPU frequency governor settings; these are captured and restored to avoid leaving machines in a specialized state for unrelated jobs.

The static resources evolve on a slower lifecycle than binaries. Syscall policies are updated when new legitimate system calls appear in supported code paths or when stricter filtering closes an exposure. Guest kernel fragments change when CI moves to newer baseline kernels or when debugging and tracing requirements shift. Overlay content changes when guest initialization or networking assumptions for tests change. Artifact regeneration writes versioned outputs next to architecture-stamped directories so multiple host architectures can coexist without collision.

At startup of a containerized command, state is initialized by creating missing directories, validating device accessibility where relevant, and synthesizing small files such as environment lists for telemetry. Shutdown focuses on ownership repair, optional tar archiving of results, and removal of temporary lists so secrets or job identifiers do not linger on disk.

## Failure Modes

The most common failures are environmental. Missing container runtime or inability to reach the image registry stops work early with explicit errors. Insufficient permissions to the virtualization device surface as warnings in interactive shells and hard failures in tests that require hardware acceleration. Download failures for remote fixtures may be transient; the tooling favors retry for pulls and sync operations but ultimately aborts if versioned artifacts cannot be retrieved. Disk exhaustion manifests during root filesystem construction or large guest builds; the test runner attempts to print filesystem usage diagnostics when runs fail in automation to aid triage.

Logical failures include version mismatches between embedded API descriptions and intended release numbers, or between compiled binaries and tagged versions, which release automation treats as fatal to prevent silently shipping inconsistent artifacts. Syscall policy drift can cause runtime failures if policies are tightened without corresponding code changes, or security gaps if policies lag new code paths.

Assumptions that can cause subtle harm include running privileged containers on shared machines without understanding residual state changes from performance modes, or editing static guest configurations without rebuilding the associated images, leading to tests that pass locally against stale disks but misrepresent CI behavior.

## Operational Characteristics

Resource consumption is dominated by compilation and kernel builds. Incremental Rust builds reuse artifacts aggressively when caches are warm. Full artifact regeneration can consume substantial CPU, memory, and disk because it compiles large upstream projects and constructs compressed filesystems. Integration tests consume CPU and RAM proportional to concurrent virtual machines and may require large hugepage pools for specific suites.

Scaling bottlenecks appear at network ingress for container and artifact downloads, at storage throughput when many tests perform disk-heavy operations, and at host kernel contention when multiple hardware-backed guests run together. Observability is primarily log-oriented: decorated console messages from shell helpers, verbose build logs from the Rust toolchain, structured test output from the Python runner, and optional metrics emission during performance-oriented tests. Continuous integration often wraps result directories into a single archive for efficient transport.

## Design Rationale

The design separates orchestration from implementation so the hypervisor codebase stays focused on correctness and performance while tooling encodes repository policy: which toolchain versions are supported, how tests must be invoked to be meaningful, and which static artifacts constitute the contract with downstream users. Containerizing the environment trades slight complexity for reproducibility across Linux hosts and CI fleets. Persisting caches on the host trades a bit of mutability for dramatically faster iteration than a fully ephemeral model.

Pinning syscall policies as explicit data rather than embedding ad hoc lists only in source honors security reviewability and diff clarity. Layered kernel fragments avoid a single monolithic configuration that is hard to reason about or selectively update. Building guest images via scripted overlays plus chrooted package installation balances fidelity to real distributions with the need for deterministic, automatable construction.

Performance test modes deliberately manipulate host state because microbenchmark stability often depends on frequency scaling and memory placement; the tooling encapsulates those manipulations and attempts to restore prior state to limit collateral impact. The comparative testing harness exists because performance regressions are statistical in nature: comparing structured metrics across revisions with explicit ignore lists reduces false positives from known noisy cases while still highlighting unexpected drift.

Taken together, the tooling layer and the static resources tree form a closed loop: policies and guest definitions shape what is built, built artifacts and images feed what is tested, and test outcomes inform when policies or guest materials must evolve. That loop is the architectural backbone that keeps day-to-day development aligned with what automation enforces and what releases ship.
