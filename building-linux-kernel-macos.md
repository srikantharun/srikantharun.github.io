# Building the Linux Kernel on macOS with Bazel

**Cross-Compilation Challenges, Sandbox Differences, and Hermetic Solutions**

Building the Linux kernel on macOS presents unique challenges that don't exist on Linux hosts. This post documents the issues we encountered and the solutions that made it work with Bazel's hermetic build system.

---

## Why Build Linux on macOS?

```mermaid
flowchart LR
    subgraph "Development Team"
        A[👨‍💻 macOS Developer]
        B[👩‍💻 macOS Developer]
        C[🖥️ Linux CI Server]
    end

    subgraph "Build Targets"
        D[🐧 ARM64 Linux Kernel]
        E[📦 Kernel Modules]
        F[🔧 Device Drivers]
    end

    subgraph "Use Cases"
        G[🎮 FunSim - Functional Simulation]
        H[🧪 Driver Testing]
        I[📱 Embedded Deployment]
    end

    A --> D
    B --> D
    C --> D
    D --> G
    E --> H
    F --> I
```

For teams developing embedded systems, kernel drivers, or simulation environments, developers on macOS need to build ARM64/x86_64 Linux kernels without switching to Linux workstations.

---

## Issue 1: Case-Sensitive Filesystem

### The Problem

macOS uses a **case-insensitive** filesystem by default. The Linux kernel contains files that differ only by case:

```mermaid
flowchart TD
    subgraph "Linux Kernel Source Tree"
        A["include/uapi/linux/netfilter/"]
        A --> B["xt_DSCP.h"]
        A --> C["xt_dscp.h"]
    end

    subgraph "Extraction on macOS (Case-Insensitive)"
        D["tar -xzf linux.tar.gz"]
        D --> E["xt_DSCP.h extracted"]
        E --> F["xt_dscp.h extracted"]
        F --> G["⚠️ Overwrites xt_DSCP.h!"]
        G --> H["❌ Build fails: missing header"]
    end

    subgraph "Extraction on Linux (Case-Sensitive)"
        I["tar -xzf linux.tar.gz"]
        I --> J["xt_DSCP.h ✓"]
        I --> K["xt_dscp.h ✓"]
        J --> L["✅ Both files preserved"]
        K --> L
    end
```

### Solutions

**Option A: Case-Sensitive APFS Volume**

```bash
# Create a dedicated case-sensitive volume
diskutil apfs addVolume /dev/disk3 "Case-sensitive APFS" linux

cd /Volumes/linux
tar -xzf linux-6.x.tar.gz
```

**Option B: Bazel's Sandbox**

Bazel's sandbox preserves case because it symlinks to original paths rather than copying:

```mermaid
flowchart LR
    subgraph "Original Source (Case-Preserved)"
        A["@linux//:all_srcs"]
        B["xt_DSCP.h"]
        C["xt_dscp.h"]
    end

    subgraph "Bazel Sandbox"
        D["execroot/"]
        E["symlink → xt_DSCP.h"]
        F["symlink → xt_dscp.h"]
    end

    A --> D
    B --> E
    C --> F
    E --> G["✅ Case preserved"]
    F --> G
```

---

## Issue 2: Missing System Headers

### The Problem

Linux kernel build tools expect headers that don't exist on macOS:

```mermaid
flowchart TD
    subgraph "Kernel Build Tools"
        A["objtool"]
        B["resolve_btfids"]
        C["bpftool"]
    end

    subgraph "Required Headers"
        D["elf.h - ELF format definitions"]
        E["byteswap.h - Byte order macros"]
        F["endian.h - Endianness detection"]
    end

    subgraph "Linux System"
        G["✅ /usr/include/elf.h"]
        H["✅ /usr/include/byteswap.h"]
        I["✅ /usr/include/endian.h"]
    end

    subgraph "macOS System"
        J["❌ Not available"]
        K["❌ Not available"]
        L["❌ Not available"]
    end

    A --> D
    B --> D
    C --> E
    C --> F

    D --> G
    D --> J
    E --> H
    E --> K
    F --> I
    F --> L
```

### Solution: bee-headers

The [bee-headers](https://github.com/bee-headers/headers) project provides BSD-licensed implementations:

```mermaid
flowchart LR
    subgraph "bee-headers Package"
        A["elf.h"]
        B["byteswap.h"]
        C["endian.h"]
    end

    subgraph "Bazel Integration"
        D["http_archive"]
        E["copy_to_directory"]
        F["HOSTCFLAGS injection"]
    end

    subgraph "Build Tools"
        G["objtool"]
        H["bpftool"]
    end

    A --> D
    B --> D
    C --> D
    D --> E
    E --> F
    F --> G
    F --> H
```

**Implementation:**

```python
# MODULE.bazel
http_archive(
    name = "bee_headers",
    build_file = "//:third_party/bee_headers.BUILD",
    url = "https://github.com/bee-headers/headers/archive/<commit>.tar.gz",
)
```

```python
# Inject via HOSTCFLAGS
env = select({
    "//:aarch64_macos": {
        "HOSTCFLAGS": "-I$$EXT_BUILD_ROOT/$(location @bee_headers)",
    },
    "//conditions:default": {},
})
```

---

## Issue 3: GNU vs BSD Tool Incompatibilities

### The Problem

Linux kernel Makefiles assume GNU tools. macOS ships BSD variants with different syntax:

```mermaid
flowchart TD
    subgraph "Kernel Makefile Commands"
        A["sed -i 's/foo/bar/' file"]
        B["cp --reflink=auto src dst"]
        C["find . -printf '%f'"]
        D["date --iso-8601"]
    end

    subgraph "GNU Tools (Linux)"
        E["✅ Works"]
        F["✅ Works"]
        G["✅ Works"]
        H["✅ Works"]
    end

    subgraph "BSD Tools (macOS)"
        I["❌ Different -i syntax"]
        J["❌ --reflink not supported"]
        K["❌ -printf not available"]
        L["❌ Different flags"]
    end

    A --> E
    A --> I
    B --> F
    B --> J
    C --> G
    C --> K
    D --> H
    D --> L
```

### Solution: Hermetic GNU Toolchains

```mermaid
flowchart TB
    subgraph "Bazel Toolchain Registry"
        A["@rules_coreutils//cp"]
        B["@sed (from rules_foreign_cc)"]
        C["@coreutils (GNU coreutils)"]
    end

    subgraph "PATH Construction"
        D["Hermetic tools first"]
        E["System PATH last"]
    end

    subgraph "Build Environment"
        F["PATH=$HERMETIC:$SYSTEM"]
        G["GNU sed found first ✅"]
        H["GNU cp found first ✅"]
    end

    A --> D
    B --> D
    C --> D
    D --> F
    E --> F
    F --> G
    F --> H
```

**Implementation:**

```python
PATH_MACOS_PATHS = [
    "$$(dirname $$EXT_BUILD_ROOT/$(location @sed))",
]

env = {
    "PATH": ":".join(PATH_COMMON_PATHS + PATH_MACOS_PATHS + ["$$PATH"]),
}
```

---

## Issue 4: Darwin Sandbox Symlink Resolution (The BPF Problem)

### The Problem

This was the most subtle issue. The kernel's BPF build uses relative paths that break on macOS:

```mermaid
flowchart TD
    subgraph "Source Tree Structure"
        A["linux/"]
        A --> B["kernel/bpf/Makefile"]
        A --> C["tools/scripts/Makefile.include"]
        B -->|"include ../tools/..."| C
    end

    subgraph "Linux Sandbox Behavior"
        D["execroot/linux/"]
        D --> E["kernel/bpf/ (directory)"]
        D --> F["tools/ (directory)"]
        E -->|"../tools/"| F
        F --> G["✅ Resolves correctly"]
    end

    subgraph "macOS Sandbox Behavior"
        H["execroot/"]
        H --> I["kernel/bpf/ → symlink"]
        I -->|"points to"| J["/original/source/kernel/bpf/"]
        J -->|"../tools/"| K["/original/source/tools/"]
        K -->|"NOT in sandbox"| L["❌ File not found"]
    end
```

### Detailed Breakdown

```mermaid
sequenceDiagram
    participant Make as Makefile
    participant Sandbox as Bazel Sandbox
    participant FS as Filesystem

    Note over Make,FS: Linux Sandbox (Works)
    Make->>Sandbox: include ../tools/scripts/Makefile.include
    Sandbox->>FS: Resolve from execroot/kernel/bpf/
    FS-->>Sandbox: execroot/tools/scripts/Makefile.include
    Sandbox-->>Make: ✅ File found

    Note over Make,FS: macOS Darwin Sandbox (Fails)
    Make->>Sandbox: include ../tools/scripts/Makefile.include
    Sandbox->>FS: Follow symlink to /original/kernel/bpf/
    FS->>FS: Resolve ../ from /original/kernel/bpf/
    FS-->>Sandbox: /original/tools/ (outside sandbox!)
    Sandbox-->>Make: ❌ Permission denied / Not found
```

### Solution: Disable BPF via Patch

Since BPF isn't needed for our use case (kernel modules, simulation), we disable it:

```diff
# 01-macos.patch - Rename Build file to disable BPF
diff --git a/tools/lib/bpf/Build b/tools/lib/bpf/_Build
rename from tools/lib/bpf/Build
rename to tools/lib/bpf/_Build
```

```mermaid
flowchart LR
    subgraph "Before Patch"
        A["tools/lib/bpf/Build"]
        A --> B["Kernel finds Build file"]
        B --> C["Attempts BPF build"]
        C --> D["❌ Symlink resolution fails"]
    end

    subgraph "After Patch"
        E["tools/lib/bpf/_Build"]
        E --> F["Kernel ignores _Build"]
        F --> G["BPF build skipped"]
        G --> H["✅ Build succeeds"]
    end
```

---

## Issue 5: libelf and elfutils

### The Problem

The kernel's `objtool` requires `libelf`. On Linux, this comes from `elfutils`, but it doesn't build on macOS:

```mermaid
flowchart TD
    subgraph "objtool Dependencies"
        A["objtool"]
        A --> B["libelf.a"]
        B --> C["elfutils package"]
    end

    subgraph "elfutils on Linux"
        D["./configure && make"]
        D --> E["✅ Builds successfully"]
    end

    subgraph "elfutils on macOS"
        F["./configure && make"]
        F --> G["❌ __attribute__((alias)) not supported"]
        F --> H["❌ posix_fallocate missing"]
        F --> I["❌ tdestroy missing"]
    end

    C --> D
    C --> F
```

### Solution: Platform-Conditional Dependencies

```mermaid
flowchart TD
    subgraph "Bazel select()"
        A["Platform Detection"]
        A -->|"Linux"| B["Include @elfutils//:elf"]
        A -->|"macOS"| C["Exclude elfutils"]
    end

    subgraph "Linux Build"
        B --> D["objtool links libelf"]
        D --> E["✅ Full functionality"]
    end

    subgraph "macOS Build"
        C --> F["objtool uses stubs"]
        F --> G["✅ Builds (limited features)"]
    end
```

```python
# Platform-conditional dependencies
_PKG_CONFIG_DEPS = COMMON_DEPS + select({
    "//:aarch64_macos": [],  # Skip elfutils
    "//conditions:default": ["@elfutils//:elf"],
})
```

---

## Complete Build Architecture

```mermaid
flowchart TB
    subgraph "Input Sources"
        A["@linux - Kernel Source"]
        B["@rust_tools - Rust Toolchain"]
        C["@llvm_toolchain - Clang/LLVM"]
    end

    subgraph "Platform Detection"
        D{"//:aarch64_macos?"}
    end

    subgraph "macOS-Specific Dependencies"
        E["@bee_headers - elf.h, etc."]
        F["@sed - GNU sed"]
        G["@rules_coreutils - GNU cp"]
        H["01-macos.patch - Disable BPF"]
    end

    subgraph "Common Dependencies"
        I["@openssl - crypto libs"]
        J["@zlib - compression"]
        K["@bison - parser generator"]
        L["@flex - lexer generator"]
    end

    subgraph "Linux-Specific Dependencies"
        M["@elfutils - libelf"]
    end

    subgraph "kbuild_configure_make"
        N["Environment Setup"]
        O["PATH construction"]
        P["HOSTCFLAGS injection"]
        Q["configure && make"]
    end

    subgraph "Output"
        R["ARM64 Linux Kernel Image"]
        S["Kernel Headers (kdir)"]
        T["Module Build Support"]
    end

    A --> D
    B --> N
    C --> N

    D -->|Yes| E
    D -->|Yes| F
    D -->|Yes| G
    D -->|Yes| H
    D -->|No| M

    E --> P
    F --> O
    G --> O
    H --> A

    I --> N
    J --> N
    K --> N
    L --> N
    M --> N

    N --> Q
    O --> Q
    P --> Q

    Q --> R
    Q --> S
    Q --> T
```

---

## Build Flow Timeline

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Bazel as Bazel
    participant Sandbox as Darwin Sandbox
    participant Make as Kernel Make

    Dev->>Bazel: bazel build @linux//:Image

    rect rgb(240, 240, 255)
        Note over Bazel: Repository Phase
        Bazel->>Bazel: Fetch @linux source
        Bazel->>Bazel: Fetch @bee_headers
        Bazel->>Bazel: Fetch @llvm_toolchain
        Bazel->>Bazel: Apply 01-macos.patch
    end

    rect rgb(255, 240, 240)
        Note over Bazel,Sandbox: Analysis Phase
        Bazel->>Bazel: Resolve select() for macOS
        Bazel->>Bazel: Include bee_headers, exclude elfutils
        Bazel->>Bazel: Construct build environment
    end

    rect rgb(240, 255, 240)
        Note over Sandbox,Make: Execution Phase
        Bazel->>Sandbox: Create sandbox with symlinks
        Sandbox->>Make: Run configure (defconfig)
        Make->>Make: Find hermetic tools via PATH
        Make->>Make: Find headers via HOSTCFLAGS
        Make->>Sandbox: Build kernel (-j8)
        Sandbox->>Bazel: Return Image + kdir
    end

    Bazel->>Dev: ✅ bazel-bin/.../Image
```

---

## Key Takeaways

```mermaid
mindmap
  root((Linux on macOS))
    Case Sensitivity
      Bazel sandbox handles it
      Manual builds need APFS volume
    Missing Headers
      bee-headers provides elf.h
      Inject via HOSTCFLAGS
    GNU vs BSD Tools
      Hermetic toolchains
      PATH prepending
    Sandbox Symlinks
      Darwin resolves differently
      Patch problematic paths
    Conditional Deps
      select() per platform
      Skip elfutils on macOS
```

---

## Quick Reference: Manual Build Prerequisites

```bash
# Install GNU tools
brew install coreutils findutils gnu-sed gnu-tar grep llvm lld make pkg-config

# Install Linux headers for macOS
brew tap bee-headers/bee-headers
brew install bee-headers/bee-headers/bee-headers
source bee-init

# Create case-sensitive volume
diskutil apfs addVolume /dev/disk3 "Case-sensitive APFS" linux

# Build
cd /Volumes/linux
curl -L https://github.com/torvalds/linux/archive/v6.x.tar.gz | tar -xz
cd linux-6.x
make LLVM=1 defconfig
make LLVM=1 -j$(sysctl -n hw.physicalcpu)
```

---

## References

- [bee-headers project](https://github.com/bee-headers/headers) - BSD-licensed Linux headers for macOS
- [Linux kernel BPF path fix](https://github.com/torvalds/linux/commit/150a27328b681425c8cab239894a48f2aeb870e9)
- [Building kernel tools from source tree](https://stackoverflow.com/questions/57286857/how-to-compile-tool-and-samples-from-within-the-kernel-source-tree)
- [rules_foreign_cc](https://github.com/bazelbuild/rules_foreign_cc) - Bazel rules for building non-Bazel projects
- [Rust for Linux](https://github.com/Rust-for-Linux/linux) - Linux kernel with Rust support

---

*Published: January 2026*
