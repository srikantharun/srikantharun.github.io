# Functional Simulation for Multi-Tile AI Accelerators

**Bridging the Gap Between RTL and Silicon**

---

## The Simulation Gap

When developing custom AI accelerators, teams face a fundamental timing problem:

| Simulation Type | Speed | Fidelity | When Available |
|-----------------|-------|----------|----------------|
| **RTL Simulation** | ~1-10 Hz | Cycle-accurate | Early design |
| **FPGA Emulation** | ~1-10 MHz | Near-accurate | Mid development |
| **Real Silicon** | GHz | Ground truth | Post tape-out |

RTL simulation is too slow for running real ML workloads. Real hardware doesn't exist yet. FPGA emulation requires expensive boards and long build times.

**Functional simulation** fills this gap: fast enough to run real firmware, accurate enough to validate system behavior.

```mermaid
graph LR
    subgraph "Development Timeline"
        A[RTL Design] --> B[Functional Sim]
        B --> C[FPGA Emulation]
        C --> D[Silicon Bring-up]
    end

    subgraph "Speed vs Fidelity"
        E["RTL: 1 Hz<br/>Cycle-accurate"]
        F["FunSim: 100 MHz+<br/>Behavioral"]
        G["Silicon: GHz<br/>Ground truth"]
    end

    A -.-> E
    B -.-> F
    D -.-> G
```

---

## What We Model (and What We Skip)

Functional simulation isn't about modeling every transistor. It's about modeling **what matters for software validation**:

```mermaid
graph TD
    subgraph "Modeled (Behavioral)"
        M1[Network-on-Chip Topology]
        M2[Tile MMIO Interfaces]
        M3[Memory Controller Behavior]
        M4[Interrupt Delivery]
        M5[Accelerator Compute Results]
    end

    subgraph "Abstracted Away"
        S1[Pipeline Hazards]
        S2[Cache Coherence Protocols]
        S3[Clock Domain Crossings]
        S4[Power Management States]
    end

    subgraph "Goal"
        G[Firmware runs correctly<br/>System behavior matches spec]
    end

    M1 --> G
    M2 --> G
    M3 --> G
```

### What to Model in QEMU (Beyond CPU Emulation)

| Component | Why Model It? | How (QEMU + Extensions) |
|-----------|---------------|-------------------------|
| **Custom NoC** | Validates data routing, congestion, arbitration | QEMU TCG + custom MMIO devices; route traffic via sockets |
| **Memory Subsystem** | Tests bandwidth limits, bank conflicts | Emulate memory controllers with latency/bandwidth caps |
| **Accelerator Tiles** | Verify tile scheduling, DMA, interrupts | Model as QEMU platform devices; expose register interface |
| **Inter-Chip Communication** | Simulate multi-chip/module setups | Loopback TCP/UDP or vhost-user between QEMU instances |
| **Clock & Power Domains** | Validate DVFS, idle states | Inject synthetic delays or throttling based on workload |

```mermaid
flowchart TD
    subgraph "QEMU Instance"
        CPU[CPU Model<br/>RISC-V/ARM]

        subgraph "Custom Device Models"
            NOC[NoC Controller<br/>MMIO + Socket Backend]
            MEM[Memory Controller<br/>Bandwidth Limiting]
            ACC[Accelerator Model<br/>Compute + DMA]
            INT[Interrupt Controller<br/>Level/Edge Triggered]
        end

        subgraph "Backends"
            SOCK[Socket I/O<br/>Inter-Instance]
            FILE[File I/O<br/>Logging]
            SHM[Shared Memory<br/>Fast Path]
        end
    end

    CPU <--> NOC
    CPU <--> MEM
    CPU <--> ACC
    CPU <--> INT

    NOC <--> SOCK
    ACC <--> FILE
    MEM <--> SHM
```

### Key Modeling Decisions

| Component | Model Approach | Why |
|-----------|---------------|-----|
| **CPU Cores** | Full ISA emulation (QEMU TCG) | Firmware must execute correctly |
| **NoC** | Message-passing with configurable latency | Validates routing and deadlock-freedom |
| **Accelerator Units** | Functional compute (golden model) | Validates data flow, not timing |
| **Memory** | Bandwidth-capped flat address space | Validates addressing and bandwidth limits |
| **Interrupts** | Level-triggered delivery | Validates firmware interrupt handlers |
| **DMA Engines** | Async copy with completion interrupt | Validates firmware DMA programming |

---

## Architecture: Multi-Instance QEMU Simulation

For multi-tile systems, we run **separate QEMU instances** connected via sockets. This approach:

- Allows independent tile development and debugging
- Scales to arbitrary tile counts
- Enables distributed simulation across machines

```mermaid
flowchart TB
    subgraph "Host Machine"
        subgraph "QEMU Instance: Host Controller"
            HC[x86/ARM CPU Model]
            HD[PCIe Device Model]
            HM[Host Memory]
        end

        subgraph "QEMU Instance: Tile 0"
            T0C[RISC-V CPU]
            T0A[Accelerator Model]
            T0M[Local Memory]
            T0N[NoC Interface]
        end

        subgraph "QEMU Instance: Tile 1"
            T1C[RISC-V CPU]
            T1A[Accelerator Model]
            T1M[Local Memory]
            T1N[NoC Interface]
        end

        subgraph "QEMU Instance: Tile N"
            TNC[RISC-V CPU]
            TNA[Accelerator Model]
            TNM[Local Memory]
            TNN[NoC Interface]
        end
    end

    subgraph "Inter-Instance Communication"
        RT[Routing Table]
        S0[Socket :65000]
        S1[Socket :65001]
        SN[Socket :6500N]
    end

    HD <--> S0
    T0N <--> S0
    T0N <--> S1
    T1N <--> S1
    T1N <--> SN
    TNN <--> SN

    RT -.->|"Tile ID → Socket"| S0
    RT -.-> S1
    RT -.-> SN
```

---

## Socket-Based NoC Emulation

Each QEMU instance exposes a **chardev socket** for inter-tile communication. A routing table maps tile IDs to network addresses:

```mermaid
sequenceDiagram
    participant T0 as Tile 0<br/>(Port 65000)
    participant RT as Routing Table
    participant T1 as Tile 1<br/>(Port 65001)
    participant T2 as Tile 2<br/>(Port 65002)

    Note over T0,T2: Tile 0 sends message to Tile 2

    T0->>RT: Lookup destination: Tile 2
    RT-->>T0: 127.0.0.1:65002

    T0->>T2: TCP packet (payload + header)
    T2->>T2: Process in firmware

    T2->>RT: Lookup destination: Tile 0
    RT-->>T2: 127.0.0.1:65000

    T2->>T0: Response packet
```

### Routing Table Format

A simple CSV maps tile IDs to socket endpoints:

```csv
# tile_id, die, x, y, host, port
0, 0, 0, 0, 127.0.0.1, 65000
1, 0, 0, 1, 127.0.0.1, 65001
2, 0, 1, 0, 127.0.0.1, 65002
3, 0, 1, 1, 127.0.0.1, 65003
600, 0, 0, 0, 127.0.0.1, 65100
```

This approach enables:
- **Local simulation**: All instances on localhost
- **Distributed simulation**: Instances across machines
- **Kubernetes deployment**: Each tile as a pod

---

## Orchestration with tmux

Managing multiple QEMU instances manually is tedious. We use **tmux** to:

1. Launch all instances in parallel
2. Display each tile in its own pane/window
3. Enable interactive debugging
4. Aggregate logs for analysis

```mermaid
flowchart LR
    subgraph "tmux Session: qemu"
        subgraph "Window 0: Host"
            P0[QEMU x86<br/>Host Controller]
        end

        subgraph "Window 1: Tiles"
            P1[Pane: Tile 0]
            P2[Pane: Tile 1]
            P3[Pane: Tile 2]
            P4[Pane: Tile 3]
        end
    end

    JSON[Config JSON] --> Orchestrator
    Orchestrator --> |"tmux new-session"| P0
    Orchestrator --> |"tmux split-window"| P1
    Orchestrator --> |"tmux split-window"| P2
    Orchestrator --> |"tmux split-window"| P3
    Orchestrator --> |"tmux split-window"| P4
```

### Configuration-Driven Simulation

A JSON configuration defines the simulation topology:

```json
{
  "tmux_split": "yes",
  "routing_table": "./routing_table.csv",
  "deployment": [{
    "host": {
      "memory": "512M",
      "port": 65100
    },
    "tiles": [
      {"tile_id": 0, "port": 65000, "firmware": "tile-0.elf", "cpus": 4},
      {"tile_id": 1, "port": 65001, "firmware": "tile-1.elf", "cpus": 4},
      {"tile_id": 2, "port": 65002, "firmware": "tile-2.elf", "cpus": 4},
      {"tile_id": 3, "port": 65003, "firmware": "tile-3.elf", "cpus": 4}
    ]
  }]
}
```

---

## Metrics That Matter (Verification KPIs)

Functional simulation isn't just about "it boots." We capture metrics that hardware architects and firmware teams actually care about:

### Key Performance Indicators

| Metric | Why It Matters | How to Measure |
|--------|----------------|----------------|
| **NoC Latency (avg/max)** | Predicts end-to-end inference delay | Timestamp packets at source/sink via shared memory or logs |
| **Memory Bandwidth Utilization** | Reveals bottlenecks before tape-out | Track bytes read/written per tile via QEMU device counters |
| **Tile Idle vs. Active %** | Measures compute efficiency | Log tile state transitions (busy/idle) from QEMU device |
| **Interrupt Latency** | Impacts real-time response | Inject event → measure host ISR response time |
| **End-to-End Throughput** | Ultimate system-level KPI (tokens/sec) | Drive workload from host → time completion |
| **Determinism / Reproducibility** | Critical for debug | Ensure identical seed → identical logs (use `-icount` in QEMU) |

```mermaid
graph TD
    subgraph "Instrumentation Points"
        I1[Firmware Trace Macros]
        I2[QEMU Device Counters]
        I3[Socket Traffic Logs]
        I4[Memory Access Hooks]
    end

    subgraph "Collected Metrics"
        M1[NoC Latency<br/>per hop avg/max]
        M2[Tile Utilization<br/>busy vs idle %]
        M3[Memory Bandwidth<br/>GB/s per tile]
        M4[Queue Depths<br/>backpressure detection]
        M5[Interrupt Latency<br/>cycles to ISR]
        M6[Throughput<br/>tokens/second]
    end

    subgraph "Analysis Tools"
        A1[Perfetto Traces]
        A2[Aggregated Statistics]
        A3[Bottleneck Detection]
        A4[Regression Comparison]
    end

    I1 --> M1
    I1 --> M2
    I2 --> M3
    I2 --> M5
    I3 --> M4
    I4 --> M6

    M1 --> A1
    M2 --> A1
    M3 --> A2
    M4 --> A3
    M5 --> A2
    M6 --> A4
```

### Metric Collection Architecture

```mermaid
flowchart LR
    subgraph "QEMU Tile Instance"
        FW[Firmware] --> |"TRACE_BEGIN/END"| UART[Serial Output]
        DEV[Device Model] --> |"Counter Registers"| MMIO[MMIO Interface]
        NOC[NoC Model] --> |"Packet Timestamps"| SOCK[Socket Log]
    end

    subgraph "Collection"
        UART --> LOG[Per-Tile Logs]
        MMIO --> STATS[Statistics Dump]
        SOCK --> PCAP[Traffic Capture]
    end

    subgraph "Analysis Pipeline"
        LOG --> MERGE[Log Merger]
        STATS --> MERGE
        PCAP --> MERGE
        MERGE --> PERFETTO[Perfetto JSON]
        MERGE --> CSV[Metrics CSV]
        MERGE --> REPORT[Summary Report]
    end
```

### Trace-Based Analysis

Firmware emits structured trace events that are converted to Chrome Trace Format for visualization:

```
TS:000653209372 T:04 C:01 PERFETTO_TAG_BEGIN:COMPUTE
TS:000653215000 T:04 C:01 PERFETTO_TAG_END:COMPUTE
TS:000653215100 T:04 C:01 PERFETTO_TAG_BEGIN:SEND
TS:000653220000 T:04 C:01 PERFETTO_TAG_END:SEND
```

This enables:
- **Timeline visualization** in Perfetto/Chrome Tracing
- **Session-level analysis**: Track end-to-end latency
- **Tile-level analysis**: Identify stragglers

```mermaid
gantt
    title Tile Execution Timeline (Perfetto View)
    dateFormat X
    axisFormat %s

    section Tile 0
    COMPUTE     :0, 100
    SEND        :100, 120
    WAIT        :120, 200
    COMPUTE     :200, 300

    section Tile 1
    WAIT        :0, 50
    RECEIVE     :50, 70
    COMPUTE     :70, 170
    SEND        :170, 190

    section Tile 2
    WAIT        :0, 100
    RECEIVE     :100, 120
    COMPUTE     :120, 220
    DONE        :220, 230
```

---

## Validation Strategy

### Test Hierarchy

```mermaid
graph TD
    subgraph "Unit Tests"
        U1[Single-tile compute]
        U2[Memory operations]
        U3[Interrupt handling]
    end

    subgraph "Integration Tests"
        I1[Two-tile communication]
        I2[Loopback: send → receive → verify]
        I3[Multi-hop routing]
    end

    subgraph "System Tests"
        S1[Full topology boot]
        S2[ML workload execution]
        S3[Error injection & recovery]
    end

    subgraph "Correlation"
        C1[Compare vs RTL golden]
        C2[Compare vs FPGA results]
        C3[Compare vs silicon]
    end

    U1 --> I1
    U2 --> I1
    I1 --> S1
    I2 --> S2
    S2 --> C1
    C1 --> C2
    C2 --> C3
```

### Loopback Tests

The simplest integration test validates the full data path:

```mermaid
sequenceDiagram
    participant Host
    participant Tile0
    participant Tile1
    participant Tile2
    participant Tile3

    Host->>Tile0: Send test vector
    Tile0->>Tile0: Compute (e.g., MVM)
    Tile0->>Tile1: Forward result
    Tile1->>Tile1: Compute
    Tile1->>Tile2: Forward result
    Tile2->>Tile2: Compute
    Tile2->>Tile3: Forward result
    Tile3->>Tile3: Compute
    Tile3->>Host: Return final result

    Host->>Host: Verify against golden
```

### Correlation Metrics

Track these across simulation levels to build confidence:

| Metric | FunSim | FPGA | Silicon | Status |
|--------|--------|------|---------|--------|
| Compute result | 0xDEADBEEF | 0xDEADBEEF | 0xDEADBEEF | PASS |
| Message count | 1024 | 1024 | 1024 | PASS |
| Relative ordering | A→B→C | A→B→C | A→B→C | PASS |
| Cycle count | N/A | 50,000 | 48,500 | ~3% delta |

---

## Practical Implementation Patterns

### 1. QEMU Device Properties

Pass configuration to QEMU devices via `-global`:

```bash
qemu-system-riscv64 \
  -global driver=noc,property=routing_table,value=./routing.csv \
  -global driver=noc,property=tile_id,value=0 \
  -chardev socket,id=noc_socket,host=0.0.0.0,port=65000,server=on,wait=off
```

### 2. Per-Tile Logging

Capture per-core output to separate files:

```bash
-chardev file,id=log_core0,path=log_tile0_core0.txt -serial chardev:log_core0
-chardev file,id=log_core1,path=log_tile0_core1.txt -serial chardev:log_core1
```

### 3. tmux Orchestration Pattern

```python
# Pseudocode for orchestrator
def launch_simulation(config):
    # Create tmux session
    subprocess.run(["tmux", "new-session", "-d", "-s", "sim"])

    # Generate routing table from config
    generate_routing_table(config)

    # Launch each tile in a pane
    for i, tile in enumerate(config["tiles"]):
        cmd = build_qemu_command(tile)
        if i == 0:
            subprocess.run(["tmux", "send-keys", "-t", "sim", cmd, "C-m"])
        else:
            subprocess.run(["tmux", "split-window", "-v", "-t", "sim"])
            subprocess.run(["tmux", "send-keys", "-t", f"sim.{i}", cmd, "C-m"])

    # Attach for interactive use
    subprocess.run(["tmux", "attach", "-t", "sim"])
```

---

## Key Takeaways

```mermaid
mindmap
  root((Functional Simulation))
    Speed
      100MHz+ virtual time
      Real firmware execution
      Interactive debugging
    Fidelity
      Behavioral accuracy
      Message ordering
      Interrupt delivery
    Scalability
      Multi-instance QEMU
      Socket interconnect
      Distributed deployment
    Validation
      Loopback tests
      Trace analysis
      Silicon correlation
```

1. **Model what matters**: Focus on interfaces and behavior, not microarchitecture
2. **Socket-based NoC**: Simple, scalable inter-instance communication
3. **Configuration-driven**: JSON defines topology, orchestrator handles complexity
4. **Trace everything**: Perfetto-compatible traces enable deep analysis
5. **Correlate early**: Compare FunSim results with RTL/FPGA before silicon

---

## Further Reading

- [QEMU Documentation](https://www.qemu.org/docs/master/) - Device modeling and chardev sockets
- [Perfetto](https://perfetto.dev/) - Trace visualization
- [tmux Manual](https://man7.org/linux/man-pages/man1/tmux.1.html) - Terminal multiplexing

---

*The best simulation is one that finds bugs before they're burned into silicon.*
