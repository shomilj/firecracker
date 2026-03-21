# Firecracker tooling architecture

1. **Purpose and scope**

   1.1. The tooling layer is the operational bridge between a developer host and the Firecracker codebase. It standardizes how the virtual machine monitor and its satellite binaries are compiled, how integration tests run under KVM with realistic guest images, how performance is compared across revisions, and how releases are cut and published. Design centers on reproducibility (pinned toolchains and container images), isolation (builds and tests inside a controlled Linux userspace), and safety (privileged operations are explicit, followed by permission repair on the host).

   1.2. Four cooperating planes structure the work: **host orchestration** (prepare directories, ensure the development image, invoke the container runtime with correct bind mounts and capabilities), **build** (Rust workspace compilation, static linking checks, optional release bundle assembly), **verification** (pytest integration tests, optional statistical comparison of performance telemetry), and **automation** (version bumps, changelog and credits updates, Git tagging, draft release publishing). Auxiliary utilities cover interactive debugging, kernel UAPI binding regeneration, and ABI drift detection.

2. **Development container pipeline**

   The project does not rely on editor-specific devcontainer metadata. Instead, a **pinned development image** built from an explicit recipe is the single source of truth for developer and automation environments. Understanding that pipeline is prerequisite to understanding every other tool behavior.

   2.1. **Provenance and versioning**

      2.1.1. The default image reference combines a public registry namespace with a **tag** that moves only when maintainers intentionally upgrade toolchains or system packages. Operators may override the tag via environment for emergency pins or experiments.

      2.1.2. The orchestration layer pulls the image with **bounded retries** so transient registry failures do not abort long CI jobs. Local workflows benefit the same way.

   2.2. **Recipe stages (what the image contains and why)**

      2.2.1. **Base operating system** is a recent long-term-support Ubuntu image, chosen for wide package availability and predictable glibc behavior for host-tool compatibility.

      2.2.2. **One-off compiled dependencies** are built from verified upstream sources where integrity matters (for example, a QEMU contrib binary for block-device testing). The recipe installs minimal build dependencies, compiles, strips, installs the artifact into a system prefix, then **purges** build-only packages to keep layers smaller.

      2.2.3. **Python** is not used from the host Python stack. A locked **Poetry** manifest drives creation of a dedicated virtual environment during image build; only main dependencies are installed, and caches are removed so the layer stays reproducible. The runtime sets the virtual environment on the default path so pytest, formatters, linters, and metrics libraries resolve consistently.

      2.2.4. **Rust** is installed via the standard toolchain installer with a **fixed stable version**, minimal profile expanded with formatting, linting, coverage helpers, and architecture-specific musl targets pre-registered so on-the-fly downloads during builds are avoided. Additional Cargo-installed utilities cover auditing, dependency sorting, policy checking, fuzzing, and formal verification; one toolchain installs a nightly used only for specialized checks, with extra components layered onto that nightly for small auxiliary tools.

      2.2.5. **Crosvm** is built from a **fixed commit** as an external block backend for tests, then the temporary extra Rust toolchain used only for that compile is removed to avoid ambiguity about which compiler applies to the main workspace.

      2.2.6. **Musl static linking support** compiles a static libseccomp from upstream because distribution packages are typically glibc-linked and unsuitable for fully static musl release binaries. Header symlinks align architecture-specific and generic kernel UAPI includes under the musl include tree so cross-compilation finds consistent definitions.

      2.2.7. **Static iperf** for vsock is built separately from a pinned upstream snapshot so network performance tests can use a known artifact.

      2.2.8. **Cross GNU toolchains** for the non-host architecture are installed so static analysis and `cargo clippy` can target both supported 64-bit Linux ABIs from either host, using the GNU target for checks where musl cross-tooling is awkward.

      2.2.9. **Codecov uploader** binaries are fetched per architecture into a standard binary directory for coverage upload steps.

      2.2.10. **Git identity and secret scanning** tooling may be baked in so local style scans match automation expectations.

   2.3. **Process supervision**

      2.3.1. The image entrypoint is a minimal **init** process that reaps zombies. That matters because integration tests spawn Firecracker, jailer, pytest workers, and sometimes nested helpers; without correct reaping, long sessions leak defunct processes.

   2.4. **Local image build versus pull**

      2.4.1. Maintainers can rebuild the image from the recipe with an architecture build argument, tagging it to match the default logical name so subsequent runs use the freshly built layers.

      2.4.2. Most contributors pull the prebuilt image and never rebuild it; the rebuild path exists for air-gapped work, recipe changes under review, or debugging Dockerfile layers.

3. **Invocation fabric: how the host runs the container**

   A single **run helper** centralizes Docker arguments so every command shares the same mount topology, environment forwarding, and privilege model.

   3.1. **Bind mounts and ephemeral storage**

      3.1.1. The full repository tree is mounted read-write into a fixed inner path so sources and generated artifacts stay synchronized with the host working copy.

      3.1.2. Cargo registry and Git-based crate caches live on the host under the build tree and are mounted over the container’s Rust home locations. That way crates download once across many container invocations.

      3.1.3. Device nodes under the host dev filesystem are exposed so KVM character devices and related nodes behave like on the host.

      3.1.4. The host boot directory may be mounted read-only for firmware or microcode introspection in diagnostics.

      3.1.5. A large **tmpfs** with execute permission backs an inner service directory used as `TMPDIR` during tests so ephemeral VM images and sockets stay on fast storage with a clear lifetime.

      3.1.6. **SELinux-friendly volume labels** may be requested so Fedora-style hosts can share trees between containers without permission errors.

   3.2. **Interactive versus batch**

      3.2.1. When standard input and output are terminals, the helper attaches TTYs so shell sessions and pdb-style debugging work naturally.

   3.3. **Proxy awareness**

      3.3.1. Common HTTP and HTTPS proxy variables are copied from the host into the container so corporate proxies do not break pulls and downloads.

   3.4. **Privilege and identity**

      3.4.1. **Unprivileged** interactive shells run the container as the invoking user and group, optionally passing through the KVM device for direct access. The user is warned that jailer-based runs may still need elevated mode.

      3.4.2. **Privileged** sessions run as root inside the container, with raised **nofile** and **memlock** ulimits and **outer seccomp disabled** for the container itself so inner seccomp policies under test are not fighting Docker’s default profile. Integration tests that manipulate cgroups, network namespaces, and KVM therefore see a realistic capability set.

   3.5. **Permission repair as a first-class phase**

      3.5.1. Privileged runs create root-owned artifacts under shared trees. A **repair** step re-enters the same image non-interactively to recursively change ownership of build outputs, test result directories, and architecture-specific resource outputs back to the invoking user. This is not optional polish; without it, subsequent unprivileged builds fail on permission errors.

4. **Devtool phases: command lifecycle**

   The host-facing entry script dispatches subcommands. Each subcommand follows a recognizable **phase pattern**: validate prerequisites, optionally mutate host state, enter the container with an appropriate profile, run an inner driver, then always run repair when privilege was used.

   4.1. **Dispatch and global behavior**

      4.1.1. Unknown commands fail fast with guidance. An unattended flag exists for automation that would otherwise prompt.

   4.2. **Build subcommand**

      4.2.1. **Prerequisites**: ensure Docker works, ensure the development image exists locally, ensure build directories exist.

      4.2.2. **Profile selection**: debug versus release, musl versus GNU libc. GNU builds may exclude the jailer crate where historical compatibility requires.

      4.2.3. **Optional SSH key injection**: public and private keys from the host can be mounted to fixed inner paths so Cargo can fetch private Git dependencies during the build.

      4.2.4. **Optional revision build**: a specific commit may be built by materializing a temporary worktree, running the build there, and copying binaries into a revision-keyed area of the build tree for side-by-side comparison.

      4.2.5. **Inner driver**: a release shell script invokes Cargo with the chosen profile, may split debug information with `objcopy` on release builds, verifies static linkage for release artifacts, and optionally assembles a full release bundle with checksum manifests.

      4.2.6. **Repair**: always runs after the privileged build so the build tree returns to user ownership.

   4.3. **Test subcommand**

      4.3.1. **Prerequisites**: optional KVM capability check, development image, build directories, **guest artifact acquisition** (see below).

      4.3.2. **Artifact acquisition**: resolve the workspace semantic version’s minor line, construct a public object-store URL scoped by host architecture, sync idempotently if missing, then run a preparation pass that fixes executable bits on bundled binaries, generates or reuses an SSH keypair, injects the public key into compressed read-only root images while preserving originals, recompresses, and materializes writable copy-on-write disk images where tests need them. Parallel decompression may expose debug symbols for guest debugging scenarios.

      4.3.3. **Binary build**: by default builds release artifacts before testing. When the environment indicates a pull request against a target branch, an **additional** release build of the merge-base revision may run so tests can compare current behavior against the branch the PR would merge into.

      4.3.4. **Host mitigations**: on certain host kernel and CPU combinations, a routine may reload KVM modules with tuned parameters under a file lock when vulnerability reporting shows it is safe, mitigating known boot-time regressions. Warnings may appear if cgroup features related to dynamic code loading are absent on vulnerable CPUs.

      4.3.5. **Telemetry context**: relevant environment variables for CI vendor, coverage upload, and embedded metrics are captured into a file passed into the container so pytest and wrappers see the same context as the outer agent.

      4.3.6. **Performance mode** (opt-in): on x86 hosts, may disable turbo boost, pin Intel P-states to the highest non-turbo ratio with restoration afterward, force the performance cpufreq governor, and allocate a large hugepage pool to reduce variance for benchmark-style tests.

      4.3.7. **Inner test driver**: privileged container with relaxed outer seccomp and high ulimits; optional CPU and memory cgroup affinity passed through from the host. The inner script sets `TMPDIR` to the large tmpfs, enables **cgroups v2 nesting** by moving processes into an init subgroup and enabling controllers (mirroring a known container runtime pattern), copies staged guest images into that tmpfs so hardlinks behave, then runs pytest with an IPython-based debugger hook class.

      4.3.8. **Failure diagnostics**: when tests fail in CI, disk usage summaries print to catch space-related flakes.

      4.3.9. **Repair and teardown**: ownership repair; performance and hugepage settings restored if altered; captured env file removed.

      4.3.10. **Result archival**: in CI, test result trees may be compressed into a single tarball under the results directory to speed upload and download.

      4.3.11. **A/B mode**: may swap the inner driver for a Python tool that runs paired pytest passes against two binary directories and statistically compares embedded metrics (see section 9 in spirit—statistical comparison remains a dedicated subsystem).

   4.4. **Shell and one-shot execution**

      4.4.1. Interactive shell uses the privilege model described earlier. One-shot mode runs a quoted inner command without a login shell, still through the same mount topology.

   4.5. **Guest resource rebuild subcommand**

      4.5.1. Because building squashfs guests and compiling kernels may need **nested containers**, this subcommand enters the privileged development container and invokes the resource rebuild driver on the inner tree. That inner driver starts its own Docker daemon when needed, compiles small native helpers, runs privileged base-image steps, and writes architecture-named outputs beside the resource definitions.

      4.5.2. **Repair** runs afterward because nested builds also create root-owned files.

   4.6. **Formatting and static checks**

      4.6.1. **Format** runs Rustfmt with repository-specific options, sorts Cargo dependencies, runs Python black and isort over the tests tree and tooling Python, and runs markdown formatting across tracked markdown.

      4.6.2. **Checkstyle** optionally runs git secret scanning locally, then executes style integration tests and doctest collection **without** rebuilding or requiring KVM, using parallel pytest distribution.

      4.6.3. **Checkbuild** runs cross-target `cargo clippy` with warnings denied for each supported architecture, using GNU targets for the check where musl cross compilation is problematic.

   4.7. **Sandbox and debug helpers**

      4.7.1. **Sandbox** builds release binaries, ensures guest artifacts, then launches a tmux session with IPython importing a small module that constructs microVMs through the integration framework.

      4.7.2. **Native sandbox** skips the container and prepares a local Python venv on supported host distros for cases where nested container networking is undesirable.

      4.7.3. **Test debug** runs the inner pytest driver under tmux with pdb for focused iteration.

   4.8. **Miscellaneous**

      4.8.1. **Install** copies selected musl-linked binaries into a host prefix.

      4.8.2. **Distclean** removes build and test result trees and may delete the pulled development image.

      4.8.3. **Checkenv** validates KVM access, kernel version floors, and several production-host assumptions (KSM, swap, EPT, vulnerability sysfs) with warnings.

5. **CI integration**

   5.1. **Primary automation** assumes a CI agent that exports vendor-specific environment markers. Disk usage dumps trigger on failing tests when those markers are present, which matches hosted CI agents that capture logs from ephemeral workers.

   5.2. **Pull request comparison builds** use the vendor’s notion of a base branch for merge requests. When that variable is set, the test subcommand’s build phase produces both head and base artifacts so regressions can be isolated to the PR diff.

   5.3. **GitHub Actions hook**: pushes to mainline or release-line branches can evaluate the diff between the previous and new commit; when Rust sources, lockfiles, Cargo config, or seccomp JSON policies change, a webhook schedules an external **performance A/B pipeline** with revision identifiers for the before and after commits. That ties policy and code changes to post-merge performance surveillance without running A/B on every unrelated documentation edit.

   5.4. **Contributor workflow expectations**: the repository’s contribution guide and pull request template call out **checkbuild** and **checkstyle** invocations through the same entry script so local and automation gates align.

   5.5. **Coverage**: coverage upload tokens and CI job identifiers are forwarded via the captured environment file so inner tools can attach coverage to the correct pipeline.

6. **Inner release driver behavior (build plane detail)**

   6.1. Resolves semantic version from the main crate package id, maps Cargo profiles to output directory names, selects libc flavor, and may exclude jailer on GNU builds.

   6.2. Workspace-wide binary builds include examples. Release builds split DWARF into companion files and strip shipped binaries while retaining debug links.

   6.3. Static linkage verification inspects the main VMM binary’s ELF type to catch dynamic linking regressions.

   6.4. Optional release bundle assembly validates version strings across binaries and OpenAPI, copies seccomp JSON filters and legal notices, and emits sorted checksum manifests.

7. **Integration test execution environment (inner driver detail)**

   7.1. Large tmpfs for ephemeral images.

   7.2. cgroups v2 nesting for jailer and resource limit tests.

   7.3. Staging guest images into the tmpfs for consistent hardlink behavior.

8. **Host-specific stabilization**

   8.1. Linux 6.1 KVM module tweaks and cgroup feature warnings as above.

   8.2. Performance mode and hugepages for benchmarks.

9. **Statistical performance testing**

   9.1. Paired pytest runs, EMF JSON parsing with dimension keys, resampling tests per metric, noise suppression across scenarios, ignore lists for high-variance environments, offline analysis mode.

10. **Release automation chain**

    10.1. Version bump across manifests and OpenAPI, lockfile refresh.

    10.2. Release prepare with branch naming validation, credits regeneration, changelog retitling, signed-off commits.

    10.3. Tagging and draft GitHub release with per-architecture tarballs and checksum sidecars.

11. **Kernel UAPI binding generation**

    11.1. Bindgen wrapper with consistent attributes, mixed sources from vendor kernel trees and system headers.

    11.2. Post-processing for Rust integer types and readability.

12. **ABI verification**

    12.1. DWARF struct layout comparison between binaries with pahole, focusing on generated binding structs.

13. **Experimental distro root filesystems**

    13.1. OCI image pulls, ext4 sizing, systemd-nspawn setup scripts, smoke SSH tests across distros.

14. **Shared shell primitives**

    14.1. Central logging, fatal errors, confirmations, KVM checks, version validation—shared by orchestration scripts.

15. **End-to-end conceptual flow**

```
  Host operator or CI agent
           |
           |  ensure Docker + image + directories
           v
  +-------------------+
  |  Container run    |
  |  (user or root)   |
  +---------+---------+
            |
     bind mounts: repo, build tree, cargo caches, dev, boot, tmpfs workspace
            |
            v
  +-------------------+-------------------+
  | Inner actions                         |
  |  cargo build / clippy / fmt / tests   |
  |  nested docker for guest rebuild      |
  +-------------------+-------------------+
            |
            v
  Host post-process: chown repair, optional CI tarball of test results
```

16. **Operational invariants**

    16.1. Reproducibility via pinned toolchains and version-scoped guest artifacts.

    16.2. Separation: shell orchestration, Rust build, Python test layers stay distinct.

    16.3. Fail-closed security: privilege confined to the container context; repair avoids leaving root-owned trees on the host.

    16.4. Observability via structured metrics lines consumed by A/B tooling and dashboards.

17. **Devtool phase diagram (test path)**

```
  +-------------+
  | prerequisites|
  +------+------+
         |
         v
  +-------------+     +------------------+
  | fetch guest | --> | prepare images   |
  | artifacts   |     | keys + ext4 etc. |
  +-------------+     +---------+--------+
                                      |
                                      v
                            +-------------------+
                            | build release     |
                            | (+ merge-base if |
                            |  PR context)      |
                            +---------+---------+
                                      |
                                      v
                            +-------------------+
                            | host mitigations  |
                            | + optional perf   |
                            +---------+---------+
                                      |
                                      v
                            +-------------------+
                            | privileged test   |
                            | container + pytest|
                            +---------+---------+
                                      |
                                      v
                            +-------------------+
                            | fix_perms +       |
                            | restore perf +    |
                            | optional CI tar     |
                            +-------------------+
```
