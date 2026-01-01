# MLIR Optimization Passes: A Developer's Guide

**Understanding Multi-Level IR Transformations with Graphics Analogies**

---

## What is MLIR?

MLIR (Multi-Level Intermediate Representation) is a compiler infrastructure that enables progressive lowering from high-level abstractions to hardware-specific code. Unlike traditional compilers with a single IR level, MLIR supports **multiple dialects** that can coexist and interoperate.

Think of it like a game engine's asset pipeline: source art (PSD, FBX) transforms through multiple stages (texture compression, mesh optimization, shader compilation) before becoming runtime-ready assets.

```
High-Level ML Model (TensorFlow, PyTorch, JAX)
         ↓
    StableHLO Dialect (portable tensor operations)
         ↓
    Linalg Dialect (structured linear algebra)
         ↓
    SCF/Affine Dialect (loops and control flow)
         ↓
    Vector Dialect (SIMD abstractions)
         ↓
    LLVM/SPIR-V Dialect (hardware targets)
```

---

## Core Optimization Passes

### 1. Canonicalization: Simplify Before You Optimize

**What it does**: Applies greedy pattern rewrites to normalize IR into a canonical form, enabling further optimizations.

**Key operations**:
- Constant folding: `add(2, 3)` → `5`
- Identity elimination: `mul(x, 1)` → `x`
- Commutative normalization: `add(4, x)` → `add(x, 4)`

**Graphics Analogy**: Like pre-baking view-projection matrices at compile time. Instead of computing `view * projection * model` per vertex every frame, you fold the constant matrices during shader compilation.

```bash
mlir-opt input.mlir --canonicalize
```

**Pro tip**: Canonicalization is "best-effort"—it doesn't guarantee full normalization. Running it too aggressively early can preclude hardware-specific optimizations later.

---

### 2. CSE: Eliminate Redundant Work

**What it does**: Common Subexpression Elimination identifies when identical operations are performed multiple times and eliminates duplicates.

**How it works**:
1. Hash each operation by opcode + operands
2. Check if an equivalent operation already exists
3. Replace duplicates with references to the original

**Graphics Analogy**: Deduplicating shader texture fetches. If `texture2D(albedo, uv)` appears three times with the same UV, CSE keeps one fetch and reuses the result—reducing GPU texture unit pressure.

```bash
mlir-opt input.mlir --cse
```

**Impact**: Typically reduces operation count by 10-30% depending on IR structure.

---

### 3. DCE: Remove the Dead Weight

**What it does**: Dead Code Elimination removes operations whose results are never used.

**Key mechanisms**:
- Removes values with no consumers
- Eliminates unreachable basic blocks
- Cleans up after other transformations

**Graphics Analogy**: Stripping unused shader variants. A game might compile 100 material permutations but only ship the 20 actually referenced by draw calls. DCE is critical for binary size—removing debug visualizations, unused LODs, or experimental features.

```bash
mlir-opt input.mlir --remove-dead-values
```

---

### 4. SCCP: Propagate What You Know

**What it does**: Sparse Conditional Constant Propagation tracks constant values across control flow, optimistically assuming everything is constant until proven otherwise.

**Difference from canonicalization**:

| Aspect | Canonicalize | SCCP |
|--------|-------------|------|
| Scope | Local patterns | Cross-control-flow |
| Mechanism | Rewrite rules | Dataflow analysis |
| Dead code | No | Marks unreachable paths |

**Graphics Analogy**: Per-material shader specialization. If a material has `metallic = 1.0` (fully metallic), SCCP can specialize the entire PBR calculation—eliminating the diffuse path and branches, producing a smaller, faster shader.

```bash
mlir-opt input.mlir --sccp
```

---

### 5. Elementwise Fusion: The Memory Bandwidth Killer

**What it does**: Fuses consecutive element-wise operations into single kernels, eliminating intermediate memory round-trips.

**Before fusion**:
```mlir
%tmp1 = add %a, %b        // Write to memory
%tmp2 = mul %tmp1, %c     // Read from memory, write again
%out = sigmoid %tmp2      // Read again, write final
```

**After fusion**:
```mlir
%out = fused_op(%a, %b, %c) {
  // All operations in registers, single memory write
}
```

**Graphics Analogy**: On mobile GPUs (Mali, Adreno), memory bandwidth is the primary bottleneck. Fusing `color = albedo * lighting * ao * shadow` into one pass instead of four eliminates three intermediate texture writes—potentially 3-4x bandwidth savings.

```bash
mlir-opt input.mlir --linalg-fuse-elementwise-ops
```

**This is often the single most impactful optimization for ML workloads.**

---

### 6. Bufferization: From Functional to Concrete

**What it does**: Converts tensor semantics (immutable, SSA-based) to buffer semantics (mutable, addressable memory).

**Why it matters**: Tensors are functional abstractions. Hardware needs concrete memory addresses. Bufferization bridges this gap.

**Modern approach**: One-Shot Bufferize (recommended since MLIR 2023):
- Analyzes entire function at once
- Uses destination-passing style (DPS) for in-place updates
- Determines buffer aliasing through SSA analysis

```bash
mlir-opt input.mlir --one-shot-bufferize="bufferize-function-boundaries"
```

**Memory management pipeline**:
```
one-shot-bufferize
    ↓
ownership-based-buffer-deallocation
    ↓
buffer-deallocation-simplification
    ↓
canonicalize
```

**Graphics Analogy**: Like modern render graphs in Vulkan/DX12. Instead of dynamic allocation, you plan exact resource lifetimes and memory aliasing at compile time—enabling GPU memory to be reused across render passes.

---

## Transform Dialect: Scripting Your Optimizations

The Transform Dialect (standardized 2024-2025) lets you express optimization schedules as MLIR operations themselves—no compiler rebuilds needed.

**Key concepts**:
- **Payload IR**: The IR being optimized
- **Transform IR**: The optimization script
- **Handles**: References to payload operations

**Example**: Tile a matrix multiply and vectorize:
```mlir
transform.sequence failures(propagate) {
^bb0(%module: !transform.any_op):
  // Find the matmul operation
  %matmul = transform.structured.match
      ops{["linalg.matmul"]} in %module

  // Tile it for cache efficiency
  %tiled, %loops = transform.structured.tile_using_for
      %matmul [64, 64, 64]

  // Vectorize the inner computation
  transform.structured.vectorize %tiled
      vector_sizes [16, 16, 16]
}
```

**Graphics Analogy**: Like Unreal's Material Graph or Unity's Shader Graph with exposed compilation parameters. Artists (or engineers) tune optimization recipes per-target without modifying the compiler source.

**Benefits**:
- Rapid iteration on optimization strategies
- Enable auto-tuning over transformation space
- Different schedules for different hardware targets

---

## Hermetic Bazel Integration

Building MLIR-based compilers reproducibly requires hermetic toolchains.

### LLVM/MLIR as Bazel Dependency

```python
# MODULE.bazel
bazel_dep(name = "llvm-project", version = "17.0.0")

# Or use the overlay directly
llvm = use_extension("@llvm-project//utils/bazel:extensions.bzl", "llvm")
```

### Hermetic Toolchain Options

| Approach | Pros | Cons |
|----------|------|------|
| `toolchains_llvm` | Easy setup, bzlmod compatible | Limited customization |
| `hermetic_cc_toolchain` (Zig) | Truly hermetic, no system deps | Zig ecosystem maturity |
| Custom sysroot | Full control | Maintenance burden |

### Recommended .bazelrc

```bash
# Performance for MLIR builds
build --nobuild_runfile_links
build --nolegacy_external_runfiles

# Modern toolchain resolution
build --incompatible_enable_cc_toolchain_resolution

# Parallel compilation
build --jobs=auto

# Cache LLVM builds aggressively
build --disk_cache=~/.cache/bazel
```

### BUILD.bazel for MLIR Tools

```python
load("@llvm-project//mlir:tblgen.bzl", "gentbl_cc_library", "td_library")

# Define your dialect's tablegen files
td_library(
    name = "MyDialectTdFiles",
    srcs = ["MyDialect.td"],
    deps = ["@llvm-project//mlir:OpBaseTdFiles"],
)

# Generate C++ from tablegen
gentbl_cc_library(
    name = "MyDialectIncGen",
    tbl_outs = [
        (["-gen-dialect-decls"], "MyDialect.h.inc"),
        (["-gen-dialect-defs"], "MyDialect.cpp.inc"),
        (["-gen-op-decls"], "MyOps.h.inc"),
        (["-gen-op-defs"], "MyOps.cpp.inc"),
    ],
    tblgen = "@llvm-project//mlir:mlir-tblgen",
    td_file = "MyDialect.td",
    deps = [":MyDialectTdFiles"],
)

# Build your pass library
cc_library(
    name = "MyPasses",
    srcs = ["MyPasses.cpp"],
    hdrs = ["MyPasses.h"],
    deps = [
        ":MyDialectIncGen",
        "@llvm-project//mlir:IR",
        "@llvm-project//mlir:Pass",
        "@llvm-project//mlir:Transforms",
    ],
)
```

---

## A Complete Optimization Pipeline

Here's a typical pipeline for lowering from high-level tensor operations to LLVM:

```bash
mlir-opt input.mlir \
  # Phase 1: High-level optimizations
  --canonicalize \
  --cse \
  --sccp \

  # Phase 2: Tensor-level fusion
  --linalg-fuse-elementwise-ops \
  --linalg-fold-unit-extent-dims \

  # Phase 3: Bufferization
  --one-shot-bufferize="bufferize-function-boundaries" \
  --buffer-deallocation-pipeline \

  # Phase 4: Loop optimizations
  --convert-linalg-to-loops \
  --affine-loop-fusion \
  --affine-loop-unroll="unroll-factor=4" \

  # Phase 5: Lowering to LLVM
  --convert-scf-to-cf \
  --convert-arith-to-llvm \
  --convert-func-to-llvm \
  --reconcile-unrealized-casts \

  -o output.ll
```

---

## Performance Impact Summary

| Pass | Typical Improvement | When to Use |
|------|---------------------|-------------|
| Canonicalize | 5-15% | Always, multiple times |
| CSE | 10-30% op reduction | After any transformation |
| DCE | Binary size reduction | Before final codegen |
| SCCP | Enables specialization | With known constants |
| Elementwise Fusion | 2-4x bandwidth | Memory-bound workloads |
| Loop Tiling | Cache efficiency | Dense linear algebra |
| Vectorization | 4-16x throughput | SIMD-friendly ops |

---

## Key Takeaways

1. **Multi-level design is powerful**: Different abstraction levels enable different optimizations. Don't lower too early.

2. **Fusion is king**: For ML workloads, memory bandwidth often dominates. Elementwise fusion can be the single biggest win.

3. **Order matters**: Run canonicalize/CSE frequently. Bufferize late. Hardware-specific opts last.

4. **Transform dialect enables iteration**: Express optimization schedules in IR, iterate without rebuilding.

5. **Hermetic builds enable reproducibility**: Pin LLVM/MLIR versions, use consistent toolchains across platforms.

---

## Further Reading

- [MLIR Documentation](https://mlir.llvm.org/docs/)
- [Linalg Dialect](https://mlir.llvm.org/docs/Dialects/Linalg/)
- [Transform Dialect Tutorial](https://mlir.llvm.org/docs/Tutorials/transform/)
- [One-Shot Bufferization](https://mlir.llvm.org/docs/Bufferization/)

---

*Building ML compilers with MLIR? The key is understanding when each optimization applies—and having the discipline to profile before optimizing.*
