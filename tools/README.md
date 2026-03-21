# Firecracker tooling architecture

1. **Purpose and scope**

   1.1. The tooling layer is the operational bridge between a developer’s host machine and the Firecracker codebase: it standardizes how the VMM and its satellite binaries are compiled, how integration tests run under KVM with realistic guest images, how performance is compared across revisions, and how releases are cut and published. Everything is designed around reproducibility (pinned toolchains and container images), isolation (builds and tests run inside a controlled Linux userspace), and safety (privileged operations are explicit, followed by permission repair on the host).

   1.2. Architecturally, the layer splits into four cooperating planes: a **host orchestration** plane (shell logic that prepares directories, pulls or builds the development image, and invokes the container runtime with the right bind mounts and capabilities), a **build** plane (Rust workspace compilation, static linking checks, optional release artifact assembly), a **verification** plane (pytest-based integration tests, optional statistical A/B comparison of performance telemetry), and an **automation** plane (version bumps, changelog and credits updates, Git tagging, GitHub release drafting). Auxiliary utilities cover interactive debugging (REPL-driven microVM bring-up), kernel UAPI binding regeneration, and ABI drift detection.

2. **Host orchestration and the development container model**

   2.1. The primary workflow assumes a container runtime on the host. A **pinned image** from a public registry (version controlled via an environment override) is pulled with retries; the repository tree and a dedicated **build tree** on the host are bind-mounted into the container so that sources and artifacts remain visible after each run. Cargo’s registry and git-based crate caches live under the build tree so repeated invocations do not re-download dependencies.

   2.2. The container entry uses a minimal init (`tini`) to reap subprocesses correctly—important for long test runs and nested tooling. Inside the image, Python dependencies are installed into an isolated virtual environment (declared via Poetry) so test runners, formatters, linters, and metrics libraries share one coherent stack. Rust is installed via `rustup` with both musl targets pre-registered; additional components (formatting, clippy, coverage and verification tools, cross-compilation helpers) are bundled so local developers and CI share the same capabilities.

   2.3. **Invocation modes** distinguish non-privileged and privileged runs. Building the Rust workspace generally does not require host-level virtualization privileges beyond what the container already provides; running integration tests that exercise the jailer, cgroups, namespaces, and KVM typically requires a **privileged** container with relaxed seccomp, generous `memlock` and file descriptor ulimits, and sometimes CPU and memory affinity (`cpuset`) for deterministic performance. Proxy-related environment variables are forwarded from the host so corporate networks work transparently.

   2.4. **Permission model**: privileged runs create root-owned files under the shared build tree. A dedicated repair step re-invokes the container to recursively `chown` build and test output directories back to the invoking user, preventing accidental lock-in of the workspace after a failed or interrupted run.

   2.5. **Optional revision builds**: the orchestrator can build a specific git revision by materializing a temporary worktree, running the build there, and copying artifacts back into a revision-keyed area of the build tree—useful for comparing binaries side by side without disturbing the main checkout.

3. **Development container image internals**

   3.1. The image is built from a recent Ubuntu LTS base. Layers separate **one-off compiled tools** (for example, a QEMU `vhost-user-blk` helper built from verified upstream tarballs with OpenPGP verification) from **long-lived runtimes** (Rust toolchain, Python venv). Where possible, build-only packages are purged after compilation to shrink the final layer.

   3.2. **Rust ecosystem**: a specific stable toolchain version is installed; both musl targets are added by default. Supplementary binaries installed via Cargo include auditing, dependency sorting, license/policy checking, fuzzing helpers, and the Kani verifier with its nightly dependency—reflecting the repo’s formal methods and security posture. Crosvm is built from a fixed commit as an external `vhost-user-blk` backend for tests that need it, then the temporary toolchain used only for that build is removed.

   3.3. **Musl static linking support**: because release binaries link against musl, a statically built `libseccomp` is compiled from upstream sources with `musl-gcc`—distribution packages are often unsuitable for fully static musl links. Symlinks align kernel UAPI headers under the musl include path so cross-compilation behaves predictably.

   3.4. **Python stack**: the Poetry manifest locks testing (pytest with xdist, rerun, repeat, timeout, JSON report), formatting (`black`, `isort`, `mdformat` with plugins), linting (`pylint`, `gitlint`), HTTP and Unix socket clients, retry and file-lock utilities, scientific stacks for A/B analysis (`scipy`), embedded metrics emission, YAML, semver, and process title setting—matching what integration tests import at runtime.

   3.5. **Host utilities**: the image includes debugging and tracing tools (`gdb`, `strace`, `trace-cmd`), networking utilities used in tests (`iperf3`, `socat`, `iptables`, …), squashfs tooling, and a static `iperf3` variant built for vsock. A Codecov uploader binary is fetched per architecture. Cross GNU toolchains are installed so `cargo clippy` can target the non-host architecture for “check only” cross builds.

   3.6. **Git hygiene**: a checked-in git config can be added for consistent identity inside automation. `git-secrets` is built from source and installed to prevent accidental credential commits during style scans on developer machines.

4. **Build pipeline behavior**

   4.1. The in-container build driver resolves the semantic version from the workspace (via `cargo pkgid` on the main crate), maps Cargo profile names to output directory names (debug vs release), and selects a **libc flavor**: `musl` for static, fully featured release artifacts, or `gnu` for glibc-linked builds where cross-compilation and tooling compatibility matter. When glibc is selected, the jailer crate is excluded from the build graph by design (historical compatibility constraint).

   4.2. The driver performs a workspace-wide binary build including examples. On release profiles it **splits debug information**: `objcopy` extracts DWARF into companion `.debug` files, strips the shipped binary, and links the split debug info—yielding smaller production binaries while retaining symbols for crash analysis.

   4.3. **Static linking verification** uses `file` on the main VMM binary: release builds must report static or static-PIE linkage depending on architecture—catching accidental dynamic linking regressions early.

   4.4. **Release bundle assembly** (optional flag) only proceeds for musl. It validates each shipped binary’s `--version` output and the OpenAPI specification’s embedded version against the intended tag, copies seccomp JSON filters, legal notices, CPU template JSON fixtures, and computes a sorted `SHA256SUMS` manifest over the bundle—establishing a tamper-evident inventory.

5. **Integration test execution environment**

   5.1. The test harness sets `TMPDIR` to a large tmpfs inside the container (`/srv`) so ephemeral VM images and sockets stay on fast, writable storage with a defined cleanup boundary.

   5.2. **cgroups v2 nesting**: on modern hosts, the harness moves container processes into an `init` subgroup and enables controllers in `subtree_control`, mirroring a known containerd pattern so integration tests can create child cgroups without `EBUSY`—essential for jailer and resource tests.

   5.3. **Artifact staging**: guest disk images used in CI are copied from the host build tree into `/srv` so hardlinks and paths expected by tests resolve consistently.

   5.4. **Pytest invocation** defaults to an IPython-based debugger class for interactive failure inspection; on CI failure, disk usage summaries print to aid diagnosis of space-related flakes.

6. **CI artifact acquisition and local preparation**

   6.1. For tests that need prebuilt kernels and root filesystems, the orchestrator resolves the current minor Firecracker version, constructs a public S3 URL scoped by architecture, and syncs a versioned artifact tree idempotently. The AWS CLI must be present on the host for download.

   6.2. A follow-up preparation pass fixes executable bits on bundled helper binaries, ensures an SSH keypair exists, injects the public key into squashfs-based images (preserving originals), recompresses with zstd, and materializes writable ext4 copies where tests need copy-on-write behavior. Compressed debuginfo archives are gunzipped in parallel—making symbols available for guest-side debugging scenarios.

7. **Host-specific test stabilization**

   7.1. On certain kernel and CPU combinations, a mitigation routine reloads KVM modules with `nx_huge_pages=never` under a file lock when vulnerability sysfs reports indicate it is safe—addressing known boot-time regressions. Other paths warn if cgroup `favordynmods` is missing when the CPU is vulnerable to iTLB multihit—performance tests may degrade.

   7.2. **Performance mode** (opt-in) on x86_64 disables turbo boost, pins Intel P-states to the highest non-turbo ratio (with restoration afterward), forces the `performance` governor, and allocates a large hugepage pool—reducing variance for benchmarks. The harness records kernel version, a snippet of `cpuinfo`, and firmware package versions for traceability.

   7.3. **PR testing behavior**: in CI, when building from a pull request, the orchestrator may also build the merge-base revision’s binaries so tests can compare behavior against the target branch—supporting regression gates.

8. **Interactive sandbox (REPL-driven microVMs)**

   8.1. A small Python entrypoint (invoked under `tmux` with IPython) constructs one or two microVMs using the integration framework’s factory: default artifact paths resolve to the latest downloaded guest kernel and Ubuntu ext4 unless overridden. It enables serial console logging, resizes the writable root disk, applies baseline network and CPU/memory configuration, optionally merges a JSON CPU template into the API, starts the guest, and dumps metrics.

   8.2. A **second** microVM may boot a “debug” kernel sibling and run a short `trace-cmd` workload inside the guest—after enabling IP forwarding for DNS-dependent tracing—illustrating kernel MSR tracing workflows next to a “production” kernel guest.

9. **Statistical A/B performance testing**

   9.1. A standalone Python tool orchestrates **paired runs** of a single pytest selection against two directories of prebuilt binaries (baseline vs candidate). Each run writes JSON pytest reports; stdout is scanned for Embedded Metric Format (EMF) JSON lines.

   9.2. **EMF parsing** extracts CloudWatch dimensions as a key for each metric bundle. List-valued metrics (time series samples) are merged when dimensions repeat across lines. Metrics containing certain substrings (internal Firecracker metrics, CPU utilization) are ignored for A/B purposes.

   9.3. **Regression testing** uses a resampling-based statistical test per metric and dimension tuple, emitting p-values and mean deltas back through the metrics pipeline for observability. A **noise-suppression layer** aggregates relative changes across all scenarios for the same metric: isolated spikes that cancel in aggregate are treated as noise—grounded in empirical observation that genuine performance shifts appear across scenarios, not singly.

   9.4. **Ignore lists** encode known high-variance combinations (specific instance types, engines, or metrics) to avoid false positives on non-deterministic environments.

   9.5. Subcommands allow **end-to-end runs** (build artifacts supplied) or **offline analysis** of two existing JSON reports—supporting manual experimentation and CI triage.

10. **Release automation chain**

    10.1. **Version bump** replaces the previous semantic version across the OpenAPI spec and all crate manifests, then runs `cargo check` in every lockfile-bearing workspace subtree to refresh lock metadata—keeping the dependency graph consistent.

    10.2. **Release prepare** validates branch naming conventions against the target release line, runs the bump, regenerates contributor credits from git history (sorted, deduplicated, excluding bots), retitles the unreleased changelog section to the new version, and commits with a signed-off trailer. It prints a PR link comparing the correct upstream base (`main` vs patch release branch).

    10.3. **Release notes extraction** reads the changelog and emits plain text suitable for an annotated tag message—stripping markdown structure while preserving bullet semantics.

    10.4. **Tagging** creates an annotated tag from the current branch tip after interactive confirmation, then atomically pushes the tag and the release branch to `upstream`.

    10.5. **GitHub draft release** builds per-architecture tarballs from on-disk release directories, skips signatures and redundant template JSON already published, marks binaries executable inside the archive, attaches checksum sidecars and a test-results archive, and creates a **draft** GitHub release with the changelog body—awaiting human review before publication.

11. **Kernel UAPI binding generation**

    11.1. A shell driver wraps `bindgen` with consistent Rust crate attributes (lint allows, derive defaults) and feeds system headers for virtio, tun/tap, socket ioctl, prctl, and arch-specific MSRs. Some generations pull from a shallow-cloned Amazon Linux 5.10.y tree to match production kernel UAPI; others use live `/usr/include` headers for generic interfaces.

    11.2. **Post-processing** replaces Linux `__u*` / `__s*` integer aliases with Rust fixed-width types, selectively rewrites constant initializers to hexadecimal for readability, and applies small textual transforms (for example, signedness fixes for particular prctl constants).

    11.3. **Patches** can be applied to the working tree after generation—typically to correct mismatches between bindgen’s view of a type and kernel reality (such as unsigned character arrays for interface names).

12. **ABI verification for generated bindings**

    12.1. A Python utility compares **DWARF layout** of bindgen-related structs between two Firecracker binaries using `pahole`. It filters struct names to those whose paths suggest generated bindings, extracts size and alignment, and flags size regressions as hard failures while alignment drift may warn—new structs appearing only in the newer binary are acceptable (expansion of surface area).

13. **Experimental root filesystems from popular base images**

    13.1. A standalone workflow (intended for environments with `ctr` and systemd tools) pulls OCI images from a public catalog, mounts them read-write, sizes an ext4 image from measured content plus headroom, embeds SSH keys and a small overlay tree from the main repository, wires a custom network unit for microVM DHCP-less addressing, and uses `systemd-nspawn` to run distribution-specific setup scripts inside the image—installing SSH daemons and adjusting init systems per distro (systemd vs OpenRC).

    13.2. A companion Python driver loops over generated ext4 images, boots each with the standard integration harness + latest guest kernel, and runs a trivial SSH command to print `/etc/issue`—smoke-testing portability across Alpine, multiple Ubuntu series, Amazon Linux, etc.

14. **Shared shell primitives**

    14.1. A sourced library centralizes **logging** (timestamped, colorized when interactive), **fatal error handling** (`die` / `ok_or_die`), **user confirmation** for destructive operations, **KVM presence checks**, **Swagger version extraction**, **version string validation**, and **release branch name checks**—so every orchestration script behaves consistently and fails loudly on precondition violations.

15. **End-to-end data flow (conceptual)**

```
  Host
    |
    |  pull / ensure image
    v
  Container runtime
    |
    +-- bind mount: repo + build tree + cargo caches
    |
    +-- mode: unprivileged (build/fmt) vs privileged (test/sandbox)
    v
  In-container actions
    |
    +--> cargo build / clippy / fmt / kani (as invoked)
    |
    +--> pytest (+ optional hugepage / cpufreq tuning on host)
    |
    +--> artifacts: binaries, test_results, optional tarballs
    v
  Host post-process
        |
        +-- chown build outputs
        +-- (CI) archive test_results for upload
```

16. **Operational invariants**

    16.1. **Reproducibility**: toolchain and image versions are pinned; S3 artifact paths include the minor Firecracker version—tests target a coherent guest userspace/kernel pairing.

    16.2. **Separation of concerns**: shell orchestration handles Docker and host permissions; Rust build scripts handle compilation; Python handles test orchestration and metrics—minimizing overlap and keeping each layer testable independently.

    16.3. **Fail-closed security posture**: integration tests assume elevated privileges only inside the dedicated container context; the host repair step avoids leaving root-owned trees that might trip future unprivileged operations.

    16.4. **Observability**: EMF structured logs tie performance tests to CloudWatch-style dimensions, enabling both human A/B scripts and automated regression analysis to consume the same telemetry shape.
