# Firecracker — Architecture Documentation Index

This document is the entry point for the systems-oriented architecture notes distributed through the repository. Each linked document follows the same conventions: conceptual description of responsibilities, interfaces, data and control flow, lifecycle, failure behavior, operational characteristics, and design rationale, without relying on source-level identifiers. Read the directions file at the repository root before contributing new material or spawning delegated analysis tasks, so that style, depth, and decomposition rules stay consistent across the tree.

## System summary

Firecracker is a minimal virtual machine monitor oriented toward secure, multi-tenant serverless and container workloads. A small host-facing process exposes a configuration and control surface over a local channel, while the bulk of logic lives in a library that drives hardware-assisted virtualization, guest memory, paravirtual devices, and snapshot-based migration semantics. An optional companion process can harden the host boundary through namespaces, filesystem confinement, and privilege reduction before the monitor runs. Around this core, the repository ships offline tools for security policy compilation, CPU capability templating, and snapshot manipulation, plus a substantial automated test system that exercises functional behavior, performance, security posture, and build hygiene. The architecture is deliberately layered: declarative configuration becomes validated internal state, then virtual hardware and execution backends, with observability and rate control applied at the edges where guest traffic meets the host.

## Major topology

The diagram below describes the principal runtime and tooling relationships at a high level. The control plane is the narrow path through which an operator or orchestrator configures and drives the microVM. The isolation wrapper is optional but recommended for production-style deployments. The monitor library is the heart of execution: it owns virtual hardware state, device models, and guest CPU configuration. Paravirtual devices and guest-facing services sit inside that layer, while the guest kernel and user space run within the virtual machine boundary. Offline utilities and the test system sit outside the hot path but define the safety and compatibility guarantees the project maintains over time.

```
                    +-----------------------------+
                    |   Operator / orchestrator   |
                    +-------------+---------------+
                                  |
                                  v
                    +-----------------------------+
                    |   Control plane (API)       |
                    +-------------+---------------+
                                  |
          +-----------------------+-----------------------+
          |                                               |
          v                                               v
+------------------+                            +------------------+
| Isolation wrapper|                            |  Monitor library |
| (optional)       |                            |  (core VMM)      |
+--------+---------+                            +--------+---------+
         |                                               |
         | host boundary                                 | guest boundary
         v                                               v
+------------------+                            +------------------+
| Host OS / KVM    |                            | Guest kernel +   |
|                  |                            | user space       |
+------------------+                            +------------------+

        Offline tools (policy, snapshots, CPU templates) -----> build / CI / ops
        Test harness (Python + Rust) -------------------------> validation & gates
```

The control plane accepts declarative intent and translates it into concrete resource allocation and lifecycle decisions. The isolation wrapper reduces the attack surface of the host process by constraining what the monitor inherits from the parent environment and where it can read and write on the host filesystem. The monitor library cooperates with the host kernel virtualization interface to run virtual processors, map guest memory, inject interrupts, and service paravirtual queues. Paravirtual devices move bulk data between guest memory and host backends such as block files, network taps, entropy sources, and sockets. Guest-facing services that resemble in-network HTTP are implemented with a deliberately small stack that terminates traffic on a virtual interface path rather than acting as a general-purpose server on the host. Offline tools compile policy and reshape artifacts for migration scenarios; they are not part of the request path but are essential for repeatable deployments and safe upgrades. The test system combines Python integration tests with Rust-level benchmarks and crate-local tests to cover end-to-end behavior and micro-level performance invariants.

## Documentation map

The following links point to every architecture document in the repository, grouped by concern. Paths are relative to the repository root.

### Host binaries and isolation

- [Jailer and host isolation](src/jailer/ARCHITECTURE.md)
- [Firecracker binary and control-plane shell](src/firecracker/ARCHITECTURE.md)

### Virtual machine monitor — spine and cross-cutting

- [Monitor assembly, API adaptation, and persistence spine](src/vmm/ARCHITECTURE.md)
- [Declarative configuration and validation](src/vmm/src/vmm_config/ARCHITECTURE.md)
- [Logging and metrics](src/vmm/src/logger/ARCHITECTURE.md)
- [Snapshot envelope, versioning, and integrity](src/vmm/src/snapshot/ARCHITECTURE.md)
- [Virtual machine state and KVM-facing lifecycle](src/vmm/src/vstate/ARCHITECTURE.md)
- [Device manager and MMIO routing](src/vmm/src/device_manager/ARCHITECTURE.md)
- [Shared utilities and ACPI integration](src/vmm/src/utils/ARCHITECTURE.md)
- [Rate limiting](src/vmm/src/rate_limiter/ARCHITECTURE.md)
- [Asynchronous host I/O engine](src/vmm/src/io_uring/ARCHITECTURE.md)
- [Remote debugging stub](src/vmm/src/gdb/ARCHITECTURE.md)

### Virtual devices — bus and paravirtual I/O

- [Non-paravirtual devices and MMIO bus](src/vmm/src/devices/ARCHITECTURE.md)
- [Paravirtual shared infrastructure](src/vmm/src/devices/virtio/ARCHITECTURE.md)
- [Paravirtual block](src/vmm/src/devices/virtio/block/ARCHITECTURE.md)
- [Paravirtual network](src/vmm/src/devices/virtio/net/ARCHITECTURE.md)
- [Paravirtual sockets](src/vmm/src/devices/virtio/vsock/ARCHITECTURE.md)
- [Balloon, entropy, and generated protocol constants](src/vmm/src/devices/virtio/balloon/ARCHITECTURE.md)

### Guest CPU architecture and ACPI tables

- [Guest CPU templates and feature control](src/vmm/src/cpu_config/ARCHITECTURE.md)
- [Architecture port — AArch64](src/vmm/src/arch/aarch64/ARCHITECTURE.md)
- [Architecture port — x86_64](src/vmm/src/arch/x86_64/ARCHITECTURE.md)
- [ACPI table generation library](src/acpi-tables/ARCHITECTURE.md)

### Guest-facing network services

- [Minimal TCP/IP stack for metadata paths](src/vmm/src/dumbo/ARCHITECTURE.md)
- [Metadata service over the virtual network detour](src/vmm/src/mmds/ARCHITECTURE.md)

### Workspace libraries and offline tools

- [Workspace-wide configuration helpers](src/utils/ARCHITECTURE.md)
- [Security policy compiler and snapshot manipulation tools](src/seccompiler/ARCHITECTURE.md)
- [CPU template helper](src/cpu-template-helper/ARCHITECTURE.md)
- [Developer linting and tracing instrumentation](src/clippy-tracing/ARCHITECTURE.md)

### Automated testing

- [Python integration test framework](tests/framework/ARCHITECTURE.md)
- [Host-side test helpers](tests/host_tools/ARCHITECTURE.md)
- [Non-functional integration tests](tests/integration_tests/ARCHITECTURE.md)
- [Functional integration tests](tests/integration_tests/functional/ARCHITECTURE.md)
- [Rust integration tests and benchmarks](src/vmm/tests/ARCHITECTURE.md)

### Engineering and CI tooling

- [Developer tooling and static resources](tools/ARCHITECTURE.md)

## How to read this corpus

Start with the monitor spine document for the end-to-end lifecycle from configuration to execution and snapshotting. Follow the virtual hardware and device documents for data-plane behavior, then the architecture-specific guest CPU documents for virtualization mechanics. Use the guest-facing service documents to understand how metadata is exposed inside the microVM without broadening the host attack surface. Use the testing documents to understand which guarantees are enforced automatically and how failures surface in continuous integration. The offline tools and engineering tooling documents complete the picture for operators and contributors who need to compile policy, rebase snapshots, or reproduce environments.
