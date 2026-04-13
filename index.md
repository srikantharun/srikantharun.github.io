# Technical Blog

Engineering insights on ML compilers, hardware simulation, cross-platform toolchains, and game analytics.

---

## Posts

### Game Analytics Series

#### [Part 1: Client-Side Event Tracking Architecture](./game-analytics-part1-event-tracking)
**Building a Robust, Offline-Capable Event Collection System for Mobile Games**

Deep dive into event tracking SDK architecture: SQLite persistence, batch sending with exponential backoff, JSON-RPC transport, and platform-specific considerations for iOS and Android.

---

#### [Part 2: Streaming Pipeline - Kafka to BigQuery](./game-analytics-part2-data-pipeline)
**Building a Scalable Data Pipeline with Kafka, Dataflow, BigQuery, and Airflow**

End-to-end data pipeline design: Kafka ingestion, Apache Beam stream processing, BigQuery schema design with partitioning/clustering, and Airflow DAG orchestration for daily ETL.

---

#### [Part 3: ML-Powered Game Analytics](./game-analytics-part3-ml-predictions)
**Building AI Models for Level Difficulty Prediction, Player Behavior, and Automated Testing**

Practical ML applications: difficulty prediction with gradient boosting, player churn prediction with XGBoost and SHAP explanations, and automated level testing with reinforcement learning agents.

---

### [iOS Debugging Deep Dive: Debug Symbols, IDE Setup, and Crash Troubleshooting](./ios-debugging-guide)
**Understanding Apple's Debug Symbol Architecture, VSCode Integration, and Bazel Workflows**

Comprehensive guide to iOS debugging covering Apple's "lazy" DWARF scheme, N_OSO debug maps vs dSYM bundles, why `--fission` is Linux-only, VSCode extensions for iOS simulator debugging (vscode-ios-debug, SweetPad, CodeLLDB), Bazel dSYM generation, and crash symbolication workflows.

---

### [Building XCFrameworks for Gaming Applications](./xcframework-gaming)
**Scaling iOS Modules with Bazel: rules_swift, Platform Patterns, and Cross-Platform CI**

Patterns for integrating XCFrameworks in multi-platform gaming codebases. Covers platform-conditional dependencies with `select()`, `target_compatible_with` for iOS-only code, and CI strategies for validating 6+ platforms from a unified build graph.

---

### [Building AI-Powered iOS Applications with XCFrameworks](./xcframeworks-bazel-gaming)
**Integrating OpenCV, CoreML, and Vendor SDKs for Cross-Platform Computer Vision Apps**

Patterns for integrating iOS-only XCFrameworks in cross-platform game codebases. Covers platform-conditional dependencies, target_compatible_with, objc_library for Objective-C++, and CI strategies for validating builds across iOS, macOS, Android, and more.

---

### [Bazel IDE Toolkit: Seamless C++ IntelliSense](./bazel-ide-toolkit)
**Auto-refresh compile_commands.json for Bazel Projects**

A VS Code extension that automatically keeps your compile_commands.json in sync with your Bazel build graph. Setup guide, configuration options, and tips for large monorepos.

---

### [VS Code vs CLion for Bazel C++ Monorepos](./ide-setup-bazel)
**Choosing the Right IDE for Large-Scale C++ Development**

Comparing VS Code (with clangd) and CLion (with Bazel plugin) for real-world Bazel development. Covers setup, compile_commands.json generation, performance tuning, and when to choose each IDE.

---

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

