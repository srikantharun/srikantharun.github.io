# Dynamic Shapes in Static ML Compilers

**Handling MoE Routing, Variable Sequences, and JIT Rematerialization**

---

## The Fundamental Tension

Static ML compilers optimize aggressively by assuming fixed tensor shapes at compile time. But modern LLMs demand flexibility:

- **Variable sequence lengths**: User prompts range from 10 to 100,000 tokens
- **MoE routing**: Different experts activate per token
- **Early exit**: Skip layers when confidence is high
- **Speculative decoding**: Variable number of draft tokens

How do we preserve static compiler optimizations while supporting dynamic behavior?

```mermaid
graph LR
    subgraph "Static Compiler Wants"
        A[Fixed Shapes] --> B[Aggressive Fusion]
        B --> C[Memory Planning]
        C --> D[Vectorization]
    end

    subgraph "LLMs Need"
        E[Variable Seq Len] --> F[Dynamic Routing]
        F --> G[Early Exit]
        G --> H[Adaptive Compute]
    end

    A -.->|Tension| E
    D -.->|Tension| H
```

---

## Three Categories of Dynamism

Not all dynamic shapes are equal. Understanding the category determines the solution:

```mermaid
graph TD
    subgraph "Type 1: Bounded Dynamic"
        A1[Sequence Length] --> A2["1 ≤ seq ≤ 4096"]
        A2 --> A3[Compile for max, mask unused]
    end

    subgraph "Type 2: Data-Dependent"
        B1[MoE Routing] --> B2["8 of 256 experts per token"]
        B2 --> B3[Runtime worklist dispatch]
    end

    subgraph "Type 3: Control-Dependent"
        C1[Early Exit] --> C2["Exit at layer N if confident"]
        C2 --> C3[Conditional execution graphs]
    end
```

| Type | Example | Compile-Time Knowledge | Runtime Overhead |
|------|---------|----------------------|------------------|
| **Bounded** | Seq length ≤ 4096 | Max bound known | Padding waste |
| **Data-Dependent** | MoE routing | Distribution unknown | Dispatch overhead |
| **Control-Dependent** | Early exit | Condition unknown | Branch misprediction |

---

## Technique 1: Symbolic Shapes + Runtime Specialization

Compile with symbolic dimensions, specialize at runtime when actual shapes are known.

```mermaid
flowchart LR
    subgraph "Compile Time"
        A["Model IR<br/>batch=?, seq=?"] --> B["Symbolic Analysis"]
        B --> C["Template Code<br/>(shape-polymorphic)"]
    end

    subgraph "Runtime"
        D["Actual Shape<br/>batch=32, seq=512"] --> E["JIT Specialization"]
        E --> F["Optimized Kernel<br/>(shape-specific)"]
    end

    C --> E
```

### Trade-offs

| Approach | Compilation | Runtime | Memory |
|----------|-------------|---------|--------|
| **Fully Static** | Once, ahead-of-time | Fastest | Optimal |
| **JIT per Shape** | On first encounter | Fast after warmup | Kernel cache grows |
| **Fully Dynamic** | Minimal | Slowest | Minimal |

### Implementation Pattern

```mlir
// Symbolic shape in MLIR
func.func @attention(%q: tensor<?x?x64xf16>,   // [batch, seq, head_dim]
                     %k: tensor<?x?x64xf16>,
                     %v: tensor<?x?x64xf16>)
    -> tensor<?x?x64xf16> {
  // Compiler generates shape-polymorphic code
  // Runtime specializes for actual [32, 512, 64]
}
```

---

## Technique 2: Worklist-Based Dynamic Dispatch (MoE)

For Mixture-of-Experts, compile static expert kernels but route dynamically at runtime.

```mermaid
flowchart TD
    subgraph "Compile Time (Static)"
        E1["Expert 1 Kernel"]
        E2["Expert 2 Kernel"]
        E3["Expert ... Kernel"]
        E256["Expert 256 Kernel"]
    end

    subgraph "Runtime (Dynamic)"
        T["Tokens<br/>[128 tokens]"] --> R["Router Network"]
        R --> S["Affinity Scores<br/>[128 × 256]"]
        S --> TopK["Top-K Selection<br/>(K=8 per token)"]
        TopK --> W["Worklist Construction"]
    end

    W --> |"Token 1 → E5, E42, E198..."| E1
    W --> |"Token 2 → E12, E77, E201..."| E2

    subgraph "Gather Results"
        G["Weighted Sum<br/>per token"]
    end

    E1 --> G
    E2 --> G
    E256 --> G
```

### Key Insight: Static Kernels, Dynamic Routing

```python
# Pseudo-code for MoE dispatch
def moe_forward(tokens, router, experts):
    # Static: All expert kernels pre-compiled
    # Dynamic: Which experts to call per token

    scores = router(tokens)                    # [batch, num_experts]
    top_k_experts = scores.topk(k=8, dim=-1)   # Dynamic selection

    # Build worklist: which tokens go to which experts
    worklist = build_worklist(top_k_experts)   # Runtime construction

    # Dispatch (can be parallel across experts)
    results = []
    for expert_id, token_indices in worklist:
        expert_tokens = tokens[token_indices]
        expert_output = experts[expert_id](expert_tokens)  # Static kernel
        results.append((token_indices, expert_output))

    return gather_and_combine(results, scores)
```

### Memory Layout Considerations

```mermaid
graph LR
    subgraph "Data Parallel View"
        DP["Batch = 8<br/>(per device)"]
    end

    subgraph "Expert Parallel View"
        EP["Batch = 128<br/>(all devices)"]
    end

    subgraph "Per-Expert View"
        PE["Variable tokens<br/>(0 to 128)"]
    end

    DP -->|"All-to-All"| EP
    EP -->|"Routing"| PE
```

---

## Technique 3: JIT Tile Rematerialization

When memory pressure is high, evict computed tiles and recompute on demand.

```mermaid
flowchart TD
    subgraph "Standard Execution"
        A["Compute Tile A"] --> MA["Store in Memory"]
        B["Compute Tile B"] --> MB["Store in Memory"]
        C["Compute Tile C"] --> MC["Store in Memory"]
        MA --> D["Use A, B, C"]
        MB --> D
        MC --> D
    end
```

```mermaid
flowchart TD
    subgraph "With Rematerialization"
        A2["Compute Tile A"] --> MA2["Store A"]
        B2["Compute Tile B"] --> MB2["Store B"]
        C2["Compute Tile C"] --> MC2["Store C"]

        MP["Memory Pressure!"] --> EV["Evict Tile A"]

        D2["Need Tile A"] --> RE["Recompute A"]
        RE --> USE["Use A, B, C"]
        MB2 --> USE
        MC2 --> USE
    end
```

### When to Rematerialize vs. Store

| Factor | Favor Recompute | Favor Store |
|--------|-----------------|-------------|
| **Compute Cost** | Cheap (elementwise) | Expensive (matmul) |
| **Memory Pressure** | High | Low |
| **Reuse Count** | Low (used once) | High (used many times) |
| **Bandwidth** | Memory-bound | Compute-bound |

### Decision Function

```python
def should_rematerialize(tile, memory_state):
    compute_cost = estimate_flops(tile)
    memory_cost = tile.size_bytes
    reuse_count = count_future_uses(tile)
    pressure = memory_state.utilization

    # Heuristic: rematerialize if cheap and memory-pressured
    if pressure > 0.9 and compute_cost < THRESHOLD:
        if reuse_count <= 2:
            return True
    return False
```

---

## Technique 4: Padded Execution with Masking

The simplest approach: pad all inputs to maximum size, mask invalid positions.

```mermaid
flowchart LR
    subgraph "Variable Inputs"
        I1["Seq: 128"]
        I2["Seq: 512"]
        I3["Seq: 256"]
    end

    subgraph "Padded Batch"
        P1["Seq: 512 (128 + 384 pad)"]
        P2["Seq: 512"]
        P3["Seq: 512 (256 + 256 pad)"]
    end

    subgraph "Masked Execution"
        M["Attention with Mask"]
        O["Output (ignore padding)"]
    end

    I1 --> P1
    I2 --> P2
    I3 --> P3
    P1 --> M
    P2 --> M
    P3 --> M
    M --> O
```

### Trade-offs

| Pros | Cons |
|------|------|
| Simple implementation | Wasted compute on padding |
| Static shapes everywhere | Memory overhead |
| No runtime dispatch | Poor for high variance |
| Batch-friendly | Latency = max sequence |

### When Padding Works Well

- Sequences cluster around similar lengths
- Batch size is large (amortizes padding overhead)
- Hardware prefers uniform shapes (TPUs, matrix cores)

---

## Technique 5: Bucketed Compilation

Compile kernels for discrete shape "buckets", select at runtime.

```mermaid
flowchart TD
    subgraph "Compile Time"
        B1["Kernel: seq ≤ 128"]
        B2["Kernel: seq ≤ 512"]
        B3["Kernel: seq ≤ 2048"]
        B4["Kernel: seq ≤ 8192"]
    end

    subgraph "Runtime"
        S["Actual seq = 300"] --> D{"Select Bucket"}
        D -->|"300 ≤ 512"| B2
    end

    B2 --> E["Execute with padding to 512"]
```

### Bucket Selection Strategy

```python
BUCKETS = [128, 256, 512, 1024, 2048, 4096, 8192]

def select_bucket(seq_len):
    for bucket in BUCKETS:
        if seq_len <= bucket:
            return bucket
    return seq_len  # Fallback: compile new kernel
```

---

## Case Study: MoE in DeepSeek-V3

DeepSeek-V3 demonstrates production-grade dynamic shape handling:

```mermaid
graph TD
    subgraph "Model Architecture"
        P["671B Total Parameters"]
        A["37B Activated per Token"]
        E["256 Routed Experts"]
        K["8 Active per Token"]
    end

    subgraph "Compilation Strategy"
        S1["Static: Expert FFN kernels"]
        S2["Static: Attention (MLA)"]
        D1["Dynamic: Router scores"]
        D2["Dynamic: Worklist construction"]
    end

    P --> S1
    P --> S2
    E --> D1
    K --> D2
```

### The Hybrid Approach

| Component | Static/Dynamic | Reason |
|-----------|---------------|--------|
| Expert FFN weights | Static | Fixed per expert |
| Expert kernel code | Static | Same computation, different data |
| Router network | Static | Fixed architecture |
| Routing decision | Dynamic | Data-dependent per token |
| Token-to-expert mapping | Dynamic | Changes every forward pass |
| Expert output gathering | Dynamic | Variable tokens per expert |

---

## Combining Techniques: A Practical Pipeline

Real compilers combine multiple techniques:

```mermaid
flowchart TD
    subgraph "Stage 1: Analysis"
        A["Identify Dynamic Dimensions"]
        A --> B{"Bounded?"}
        B -->|Yes| C["Use Buckets + Padding"]
        B -->|No| D{"Data-Dependent?"}
        D -->|Yes| E["Worklist Dispatch"]
        D -->|No| F["JIT Specialization"]
    end

    subgraph "Stage 2: Memory Planning"
        C --> G["Static Allocation"]
        E --> H["Dynamic Allocation + Rematerialization"]
        F --> I["On-demand Allocation"]
    end

    subgraph "Stage 3: Execution"
        G --> J["Execute with Masks"]
        H --> K["Runtime Dispatch Loop"]
        I --> L["JIT Compile + Execute"]
    end
```

---

## Performance Comparison

| Technique | Compilation Time | Runtime Overhead | Memory Efficiency | Best For |
|-----------|------------------|------------------|-------------------|----------|
| **Fully Static** | Fast | None | Optimal | Fixed workloads |
| **Symbolic + JIT** | Deferred | Warmup | Good | Variable batch |
| **Bucketed** | N × buckets | Selection | Moderate | Sequence length |
| **Padded** | Fast | Mask overhead | Poor | Low variance |
| **Worklist MoE** | Per expert | Dispatch | Variable | Sparse activation |
| **Rematerialization** | Once | Recompute | Optimal | Memory-bound |

---

## Key Takeaways

```mermaid
mindmap
  root((Dynamic Shapes))
    Bounded Dynamic
      Bucket compilation
      Padding with masks
      Max-shape allocation
    Data-Dependent
      Worklist dispatch
      Static kernels
      Runtime routing
    Control-Dependent
      Conditional graphs
      Early exit paths
      Speculative execution
    Hybrid Solutions
      Combine techniques
      Profile-guided selection
      JIT rematerialization
```

1. **Categorize your dynamism**: Bounded, data-dependent, or control-dependent require different solutions

2. **Static where possible**: Compile expert kernels statically, route dynamically

3. **Worklists enable MoE**: Decouple static computation from dynamic dispatch

4. **Rematerialization trades compute for memory**: Essential for long sequences

5. **No silver bullet**: Production systems combine multiple techniques based on workload

---

## Further Reading

- [XLA Dynamic Shapes](https://www.tensorflow.org/xla/shapes) - TensorFlow's approach
- [TorchDynamo](https://pytorch.org/docs/stable/dynamo/) - PyTorch's JIT compiler
- [vLLM PagedAttention](https://arxiv.org/abs/2309.06180) - Dynamic KV cache management
- [Megablocks MoE](https://arxiv.org/abs/2211.15841) - Efficient MoE implementation

---

*Static compilation and dynamic execution aren't opposites—they're partners. The art is knowing where to draw the line.*
