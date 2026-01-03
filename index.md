# Technical Blog

Engineering insights on ML compilers, hardware simulation, and cross-platform toolchains.

---

## Posts

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

