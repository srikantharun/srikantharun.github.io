# Technical Blog

Engineering insights on ML compilers, hardware simulation, and cross-platform toolchains.

---

## Posts

### [Bazel Module Extensions: A Practical Guide](./bzlmod-extensions)
**When and How to Use Module Extensions in Bzlmod (Bazel 7+/8+)**

Practical patterns for module extensions: conditional repository generation, cross-module dependency aggregation, platform detection, lazy downloads, and code organization. With Mermaid diagrams and implementation examples.

---

### [Building the Linux Kernel on macOS with Bazel](./building-linux-kernel-macos)
**Cross-Compilation Challenges, Sandbox Differences, and Hermetic Solutions**

How to build the Linux kernel on macOS using Bazel's hermetic build system. Covers case-sensitivity issues, missing headers (elf.h, byteswap.h), GNU vs BSD tool differences, Darwin sandbox symlink resolution, and platform-conditional dependencies.

---

### [GitLab CI/CD Components: Build Once, Use Everywhere](./gitlab-cicd-components)
**A Practical Guide to Creating, Publishing, and Consuming Reusable Pipeline Components**

How to build reusable CI/CD components, publish them to GitLab's CI/CD Catalog registry, and consume them across projects. Covers the complete workflow from development to versioned releases.

[View Curated Components Repository](https://github.com/srikantharun/gitlab-curated-components)

---

### [Functional Simulation for Multi-Tile AI Accelerators](./functional-simulation)
**Bridging the Gap Between RTL and Silicon**

How to build fast, behavioral simulations using multi-instance QEMU with socket-based NoC emulation. Covers tmux orchestration, verification KPIs (latency, bandwidth, utilization), and Perfetto trace analysis.

---

### [Dynamic Shapes in Static ML Compilers](./dynamic-shapes)
**Handling MoE Routing, Variable Sequences, and JIT Rematerialization**

Techniques for supporting dynamic tensor shapes in ahead-of-time ML compilers: symbolic shapes, bucketed compilation, worklist-based MoE dispatch, and tile rematerialization strategies.

---

### [MLIR Optimization Passes](./mlir-optimization)
**A Deep Dive into StableHLO, Linalg, and Beyond**

Comprehensive overview of MLIR optimization passes from high-level StableHLO through Linalg to low-level code generation. Covers canonicalization, CSE, DCE, fusion, and bufferization.

---

### [Hermetic Clang Toolchains Across All Platforms](./hermetic-clang-toolchains)
**Solving Sysroot & C++23 Compatibility on Linux, macOS, Android & iOS**

Building C++ with Bazel across heterogeneous platforms with hermetic toolchains. Addresses glibc compatibility, libstdc++ vs libc++, and cross-compilation for mobile targets.

---

### [Cross-Compilation Deep Dive](./cross-compilation)
**Advanced Bazel Patterns for Multi-Architecture Builds**

Detailed patterns for cross-compiling with Bazel: sysroot management, platform detection, and toolchain configuration for ARM64, x86_64, Android, and iOS targets.

---

## About

These posts explore challenges in building ML infrastructure: from compilers that handle dynamic workloads to simulation environments that catch bugs before silicon.

