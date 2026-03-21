# Non-Functional Integration Testing Architecture

This document describes the umbrella architecture for integration tests that validate performance, security posture, repository and API style, and build health, together with a top-level formal verification entrypoint. It deliberately excludes the separate functional integration subtree, which focuses on correctness of guest-visible behavior and feature parity. The areas covered here are complementary: they stress operational limits, enforce policy, and guard engineering hygiene rather than enumerating feature scenarios. Together they form a layered assurance model in which fast static gates catch broad classes of defects early, dynamic host and microvirtual machine exercises validate runtime properties, and selected heavyweight checks run only where continuous integration can supply sufficient resources.

## Purpose & Boundaries

The responsibility of this testing umbrella is to provide repeatable, automated evidence that the project remains safe to merge and release along several independent dimensions. Performance-oriented cases assert that boot paths, networking, storage, memory accounting, and related subsystems stay within expected bounds under representative workloads. Security-oriented cases validate sandboxing rules, audit third-party and first-party supply chain signals, and compare guest-reported vulnerability posture against host baselines where applicable. Style-oriented cases enforce formatting, linting, license compliance, documentation structure, and machine-readable interface specifications so that human and automated consumers see a consistent contract surface. Build-oriented cases ensure that the primary language test suites and benchmarks compile and execute, that coverage thresholds are met with exclusions that reflect platform reality, that dependency hygiene holds, and that security rule sets stay aligned with what binaries actually require. The standalone formal verification entrypoint exercises exhaustive proof harnesses for memory safety and related properties when the continuous integration environment opts in, because those jobs are too resource-intensive for typical developer machines.

This umbrella is not responsible for proving complete functional correctness of every guest feature, nor for end-to-end product workflows outside the narrow contracts each test encodes. It does not replace manual threat modeling, penetration testing, or compliance audits. It does not guarantee absence of performance regressions under all possible customer workloads, only under the scripted scenarios included here. It does not certify that all security mitigations are optimal—only that certain measurable conditions hold and that specific tools report acceptable results under pinned configurations.

If this layer failed wholesale, the larger system would lose confidence that merges do not silently erode boot time, throughput, isolation, dependency health, or API stability. Regressions could reach production through pull requests that still pass narrower unit checks. Conversely, if only this layer failed while functional tests passed, the project might still appear feature-complete yet be slower, leakier, more permissive at the syscall boundary, or harder to maintain because style and build gates no longer run.

## Interfaces & Contracts

Externally, this suite exposes a contract to continuous integration and local developers through the shared Python-based test harness. Individual cases declare timeouts, platform skips, and environment-gated execution so that runners know which jobs require privileged hosts, large memory, or specific vendor CPUs. The suite consumes interfaces from host build helpers that wrap the primary language toolchain, from utilities that spawn and control microvirtual machines, and from static analysis helpers that inspect binaries and rule files. Security cases depend on compiled demonstration utilities and on a compiler tool that turns JSON-described syscall policies into attachable filters. Performance cases depend on guest images, networking tools inside guests, and host-side load generators. Style cases depend on canonical configuration for formatters and linters anchored at the repository root. Build cases depend on coverage aggregation tooling and on the ability to emit instrumented builds.

Callers—typically automation—must provide a clean working tree or an environment that satisfies documented prerequisites: appropriate toolchains, optional nightly components for certain static checks, network access where external scripts are fetched, and sufficient disk for coverage artifacts. The suite returns guarantees at the level of assertions and process exit codes: thresholds on timings and throughput, non-empty negative tests for forbidden behavior, absence of reported vulnerabilities over baseline comparisons on pull requests, successful completion of formatting and lint commands, and successful compilation and execution of tests and benchmarks under specified targets.

Invariants expected of maintainers include keeping policy files, OpenAPI descriptions, and changelog structure aligned with release practices. The suite provides in return a stable notion of “green” that spans static, dynamic, and proof-based evidence without requiring every machine to run every job.

## Data Flow

Inputs originate from the repository tree, the lockfile that pins transitive dependencies, generated build artifacts under predictable output locations, logs emitted by the virtual machine monitor and guests, and occasionally remote resources such as vulnerability checker scripts or security advisory feeds. Configuration for rate limiting, huge pages, and similar knobs flows from test parameters into API requests and guest environments. Seccomp-related data flows from JSON policy descriptions through a compilation step into attachable filter representations, then into process execution outcomes. Coverage data flows from instrumented test runs into raw profiles, then into merged human-readable reports compared against numeric thresholds.

The following diagram summarizes how evidence moves from sources to decisions without naming implementation artifacts.

```
  [Repository sources]
           |
           v
  +------------------+
  | Static analysis  |----> lint / format / spec validity ----> pass/fail
  | & policy files |
  +------------------+
           |
           v
  +------------------+
  | Build & tests    |----> binaries, profiles, logs ---------> thresholds
  | (instrumented)   |
  +------------------+
           |
           v
  +------------------+
  | Live microVMs    |----> metrics, timing, guest reports ----> SLO-style
  | & host tools     |         comparisons
  +------------------+
           |
           v
  +------------------+
  | Formal proofs    |----> proof logs (CI-only heavy path) ---> pass/fail
  +------------------+
```

Before the diagram, the intent is to show that not all flows are equal: static paths consume text and metadata, build paths emit measurable artifacts, live paths produce time series and logs, and proof paths consolidate many internal verification steps into a single gate. After the diagram, the important point is transformation: raw logs become parsed numbers, advisory databases become sets compared for growth, and coverage becomes a ratio against moving baselines. Outputs ultimately land in human-readable failure messages, archived logs for debugging, and pass-fail signals for merge automation.

## Control Flow

Execution is triggered by developer runs of the test harness or by continuous integration pipelines that invoke the same entrypoints. Within the umbrella, control branches early based on architecture, vendor, and environment variables that signal whether a job is allowed to run locally or must be skipped. The critical path for a full signal in CI typically walks from inexpensive static checks through compilation and unit-level aggregation, then into longer integration cases that allocate microvirtual machines. Some paths are strictly sequential because they assume exclusive access to hardware features or large memory. Others tolerate parallelism at the harness level but still serialize work inside guests.

A representative branching pattern starts with a gate that asks whether the host matches the assumptions of a security audit job; if not, the case exits as skipped rather than failed, preserving signal where it is valid. Another branch decides whether formal verification should run based on the presence of a CI flag, preventing fragile local runs. Performance cases branch on whether to exercise burst versus steady token refill behaviors, reflecting different customer configurations. Build coverage branches include vendor-specific exclusions so that platform-specific modules do not distort aggregates.

The next diagram sketches control flow at the level of decisions rather than individual cases.

```
            Start run
                |
                v
         +--------------+
         | Environment |
         | & platform   |
         | probes       |
         +--------------+
           /    |     \
          /     |      \
    skip /      |       \ continue
        /       |        \
       v        v         v
  [Heavy jobs] [Static   [Dynamic VM
   gated off]   gates]    exercises]
       |            |          |
       +------------+----------+
                    v
               Aggregate
               pass/fail
```

Before this diagram, control flow is worth emphasizing because skipped tests are not failures; they are part of the contract for portability. After the diagram, the merge of branches matters: automation must interpret skips, failures, and timeouts distinctly. Timeouts are especially important on long-running build or proof jobs because they prevent hung workers from stalling pipelines silently.

## State & Lifecycle

State is mostly ephemeral: temporary directories for compiled filters, session roots for build commands, microvirtual machine instances with associated network namespaces and tap devices, and log buffers accumulated until assertions run. Some state is semi-persistent within a CI worker, such as incremental build caches and downloaded external scripts reused across steps. The lifecycle begins with harness collection of tests, followed by fixture setup that may compile policies or boot guests, steady-state measurement windows where traffic generators run, and teardown that releases interfaces and deletes scratch paths.

Startup concerns include ensuring the working directory context matches what host helpers expect, because several style and build cases change directory to the repository root explicitly. Shutdown concerns include killing guests and servers cleanly so subsequent cases do not inherit port conflicts or stray processes. Recovery is not a first-class feature inside individual tests; failed cases rely on process isolation from the harness to reset state before the next case. For formal verification, lifecycle is dominated by long single-shot proof runs that either complete or hit a wall-clock timeout sized to the CI budget.

A small state-machine view clarifies how dynamic performance cases oscillate between setup and measurement.

```
   [Idle] --> configure guest/host
                  |
                  v
            warm-up / sync
                  |
                  v
            measure window -----> evaluate thresholds
                  |                      |
                  |                      v
                  +----------------> [Done]
                  |
                  +---- on failure --> capture logs --> [Done]
```

Before the diagram, steady-state operation is intentionally short: microvirtual machine tests optimize for signal per minute, not for soaking days. After the diagram, cleanup returns the worker to idle so the next case starts from a known baseline, which is essential when timing-sensitive assertions are involved.

## Failure Modes

Failures divide into hard assertion failures, toolchain failures, infrastructure flakes, and misconfiguration. Assertion failures are the desired kind: they mean a regression crossed a threshold or a policy was violated openly. Toolchain failures include missing nightly components or incompatible linker flags on unusual hosts; these often present as non-zero exit codes from build helpers rather than assertion messages. Infrastructure flakes appear as transient network errors when fetching external scripts, guest boot races, or timing noise on busy hosts; performance tests mitigate with retries and tolerances but cannot eliminate variance entirely.

Silent corruption is a risk wherever parsing assumes log formats remain stable. If logging text changes without updating parsers, tests might measure the wrong quantity or skip detection paths, yielding false confidence. Similarly, coverage exclusions that ignore large swaths of platform-specific code can mask gaps if someone mistakenly broadens ignore rules. Security comparisons that use advisory sets must ensure JSON parsing failures do not collapse into empty sets that appear clean.

Assumptions that are easy to violate include pinning external checker versions, relying on specific guest userspace utilities, and assuming CPU vendor constants for jobs that should be portable. When those assumptions break, failures may look like environment errors even when the code under test is fine.

## Operational Characteristics

Resource consumption is uneven. Static style jobs are CPU-bound but short. Build and coverage jobs are both CPU and disk intensive, with large artifact directories. Microvirtual machine jobs consume memory and require KVM-capable hosts; some also pin CPUs or manipulate huge pages, increasing operational sensitivity. Formal verification is memory-heavy and long-running, which is why it is gated to CI with extended timeouts.

Scaling bottlenecks include single-host throughput for nested virtualization, sequential coverage aggregation, and proof parallelism bounded by RAM. Observability is primarily harness logs, captured command output, and artifacts written to per-session result folders. There is no unified distributed tracing across tests; debugging relies on reproducing the command sequence locally with similar toolchain versions.

## Design Rationale

The problem this design solves is how to keep a security-sensitive, performance-sensitive systems project honest across many contributors without collapsing everything into one undifferentiated test suite. Splitting non-functional concerns into performance, security, style, and build gates allows pipelines to schedule fast feedback first and park expensive work behind flags or dedicated agents. It also maps cleanly to reviewer mental models: a networking throughput regression is conceptually different from a dependency advisory, even though both might block a merge.

Tradeoffs include accepting skips on non-CI machines for the heaviest jobs, which can hide proof failures until late in the cycle but protects local developer ergonomics. Another tradeoff is tolerance bands in performance tests, which reduce flakes but allow small regressions to slip through until they accumulate. Security tests favor deterministic sandbox demonstrations over claiming complete exploit resistance, which keeps runtime bounded but requires complementary review practices.

Constraints that shaped the design include the need to support multiple CPU architectures with different optional modules, the requirement to run many checks on Linux hosts with virtualization support, and the reality that some static analysis features only exist on nightly toolchains. The relationship among areas is synergistic: style gates keep public interfaces machine-verifiable; build gates ensure binaries and tests remain compilable across targets; security gates validate that isolation policies match actual syscall usage; performance gates ensure those policies and builds still yield acceptable latency and throughput. The formal verification entrypoint adds a different kind of assurance—mathematical bounded checking of critical components—complementing tests that can only sample behavior.

Taken together, these layers do not promise perfection. They promise that each merge receives multidimensional scrutiny: humans read diffs, machines enforce consistency, dynamic tests stress representative paths, and where enabled, provers push deeper on properties that unit tests cannot exhaustively cover. That combination is the architectural guarantee this umbrella aims to provide.
