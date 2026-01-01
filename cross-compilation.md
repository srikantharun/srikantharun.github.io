# Cross-Compilation with Contract Documents

**Hermetic Builds Using SSOT, cfg=exec, and Code Generation for Multi-Platform Targets**

---

## Introduction

Building software for multiple platforms—iOS, Android, PlayStation, Nintendo Switch, PC—from a single codebase is hard. Building it *hermetically* (reproducibly, with no host contamination) is harder. The key insight: **define your contract once, generate platform-specific code at build time**.

This post explores how to structure cross-platform builds using:
- **Contract Documents** (Single Source of Truth)
- **Build-time Code Generation** (cfg=exec pattern)
- **Templating Engines** (Jinja, Protobuf, Shader Compilers)

We'll use two concrete examples: **Protobuf for networking** and **Shader cross-compilation for GPUs**.

---

## SSOT vs BOM vs POM: What's the Difference?

| Concept | Purpose | Example |
|---------|---------|---------|
| **BOM** (Bill of Materials) | Version pinning | Maven BOM, npm lockfile |
| **POM** (Project Object Model) | Project structure | Maven pom.xml, Cargo.toml |
| **SSOT** (Single Source of Truth) | Schema/Contract definition | `.proto`, `.fbs`, `.hlsl` |

**BOM** says *"use protobuf v3.21.0"*.
**POM** says *"this project has these modules and dependencies"*.
**SSOT** says *"this is the actual message format everyone must agree on"*.

A **Contract Document** is an SSOT that defines the interface between systems—whether that's network protocols (Protobuf), GPU shaders (HLSL), or hardware registers (SystemRDL).

---

## The cfg=exec Pattern: Build Machine vs Target Machine

When cross-compiling, tools run on different machines than the code they produce:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BUILD MACHINE (x86_64 macOS)                 │
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │ protoc       │    │ shader_cc    │    │ flatc        │          │
│  │ (cfg=exec)   │    │ (cfg=exec)   │    │ (cfg=exec)   │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
│         │                   │                   │                   │
│         ▼                   ▼                   ▼                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │ .proto       │    │ .hlsl        │    │ .fbs         │          │
│  │ (contract)   │    │ (contract)   │    │ (contract)   │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
└─────────┼───────────────────┼───────────────────┼───────────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TARGET PLATFORMS                              │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ iOS        │  │ Android    │  │ PlayStation│  │ Nintendo   │    │
│  │ .swift     │  │ .java      │  │ .pssl      │  │ .nvn       │    │
│  │ .metal     │  │ .spv       │  │            │  │            │    │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

In Bazel, `cfg = "exec"` means: *"build this tool for the machine running Bazel, not the target platform"*.

```python
# Bazel rule attribute
"_protoc": attr.label(
    executable = True,
    cfg = "exec",  # Runs on build machine
    default = "@com_google_protobuf//:protoc",
),
```

---

## Example 1: Ansible + Jinja (Simple Templating)

Before diving into Bazel, let's see a simpler example: Ansible configuration management.

### The Contract (Jinja Template)

```yaml
# templates/nginx.conf.j2
server {
    listen {{ nginx_port }};
    server_name {{ server_name }};

    {% for backend in backends %}
    upstream {{ backend.name }} {
        server {{ backend.host }}:{{ backend.port }};
    }
    {% endfor %}
}
```

### The Variables (SSOT)

```yaml
# group_vars/production.yml
nginx_port: 443
server_name: api.example.com
backends:
  - name: app
    host: 10.0.1.10
    port: 8080
  - name: cache
    host: 10.0.1.20
    port: 6379
```

### Execution Model

```
┌─────────────────────┐         ┌─────────────────────┐
│ Ansible Controller  │         │ Target Servers      │
│ (your laptop)       │   SSH   │ (production VMs)    │
│                     │ ──────► │                     │
│ Jinja2 runs HERE    │         │ nginx.conf lands    │
│ (like cfg=exec)     │         │ HERE                │
└─────────────────────┘         └─────────────────────┘
```

**Key insight**: The template engine runs on the controller (build machine), generated configs deploy to targets. This is the same pattern as `cfg=exec`.

---

## Example 2: Bazel + Unity (Protobuf)

Game clients (Unity/C#), game servers (Rust), and analytics (Python) all need to speak the same protocol. Protobuf is the contract.

### The Contract (.proto)

```protobuf
// contracts/game_events.proto
syntax = "proto3";

package game.events;

message PlayerJoinedEvent {
  string player_id = 1;
  string display_name = 2;
  int64 timestamp_ms = 3;
  Position spawn_location = 4;
}

message Position {
  float x = 1;
  float y = 2;
  float z = 3;
}

message GameStateUpdate {
  repeated PlayerState players = 1;
  int64 tick = 2;
}

message PlayerState {
  string player_id = 1;
  Position position = 2;
  float health = 3;
  repeated string active_buffs = 4;
}
```

### Bazel Rule with cfg=exec

```python
# build_tools/proto.bzl
load("@rules_proto//proto:defs.bzl", "ProtoInfo")

def _multi_lang_proto_impl(ctx):
    proto_info = ctx.attr.proto[ProtoInfo]
    outputs = []

    # Generate C# for Unity client
    cs_out = ctx.actions.declare_file(ctx.attr.name + ".cs")
    ctx.actions.run(
        executable = ctx.executable._protoc,  # cfg=exec
        arguments = [
            "--csharp_out=" + cs_out.dirname,
            proto_info.direct_sources[0].path,
        ],
        inputs = proto_info.direct_sources,
        outputs = [cs_out],
    )
    outputs.append(cs_out)

    # Generate Rust for game server
    rs_out = ctx.actions.declare_file(ctx.attr.name + ".rs")
    ctx.actions.run(
        executable = ctx.executable._protoc,
        arguments = [
            "--rust_out=" + rs_out.dirname,
            proto_info.direct_sources[0].path,
        ],
        inputs = proto_info.direct_sources,
        outputs = [rs_out],
    )
    outputs.append(rs_out)

    # Generate Python for analytics
    py_out = ctx.actions.declare_file(ctx.attr.name + "_pb2.py")
    ctx.actions.run(
        executable = ctx.executable._protoc,
        arguments = [
            "--python_out=" + py_out.dirname,
            proto_info.direct_sources[0].path,
        ],
        inputs = proto_info.direct_sources,
        outputs = [py_out],
    )
    outputs.append(py_out)

    return [DefaultInfo(files = depset(outputs))]

multi_lang_proto = rule(
    implementation = _multi_lang_proto_impl,
    attrs = {
        "proto": attr.label(providers = [ProtoInfo]),
        "_protoc": attr.label(
            executable = True,
            cfg = "exec",  # protoc runs on BUILD machine
            default = "@com_google_protobuf//:protoc",
        ),
    },
)
```

### BUILD.bazel Usage

```python
load("@rules_proto//proto:defs.bzl", "proto_library")
load("//build_tools:proto.bzl", "multi_lang_proto")

proto_library(
    name = "game_events_proto",
    srcs = ["game_events.proto"],
)

multi_lang_proto(
    name = "game_events",
    proto = ":game_events_proto",
)

# Unity client uses C#
csharp_library(
    name = "game_events_csharp",
    srcs = [":game_events"],  # .cs file
)

# Game server uses Rust
rust_library(
    name = "game_events_rust",
    srcs = [":game_events"],  # .rs file
)

# Analytics uses Python
py_library(
    name = "game_events_python",
    srcs = [":game_events"],  # _pb2.py file
)
```

### Flow Diagram

```
                    game_events.proto (CONTRACT/SSOT)
                              │
                              ▼
                    ┌─────────────────┐
                    │     protoc      │
                    │   (cfg=exec)    │
                    │ runs on macOS   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │ .cs (C#)    │   │ .rs (Rust)  │   │ .py (Python)│
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │ Unity Game  │   │ Game Server │   │ Analytics   │
    │ iOS/Android │   │ Linux x64   │   │ Dashboard   │
    └─────────────┘   └─────────────┘   └─────────────┘
```

---

## Example 3: Shader Cross-Compilation

Shaders are the ultimate cross-compilation challenge. One shader definition must compile to:
- **Metal** (iOS, macOS)
- **SPIR-V** (Android/Vulkan)
- **DXIL** (Windows/DirectX 12)
- **PSSL** (PlayStation)
- **NVN** (Nintendo Switch)

### The Contract (HLSL or Custom Shader Language)

```hlsl
// shaders/pbr_lighting.hlsl
// PBR (Physically Based Rendering) lighting shader

struct VSInput {
    float3 position : POSITION;
    float3 normal : NORMAL;
    float2 texcoord : TEXCOORD0;
};

struct PSInput {
    float4 position : SV_POSITION;
    float3 worldPos : TEXCOORD0;
    float3 normal : TEXCOORD1;
    float2 texcoord : TEXCOORD2;
};

cbuffer SceneConstants : register(b0) {
    float4x4 viewProjection;
    float3 cameraPosition;
    float3 lightDirection;
    float3 lightColor;
};

Texture2D albedoMap : register(t0);
Texture2D normalMap : register(t1);
Texture2D metallicRoughnessMap : register(t2);
SamplerState defaultSampler : register(s0);

PSInput VSMain(VSInput input) {
    PSInput output;
    output.position = mul(viewProjection, float4(input.position, 1.0));
    output.worldPos = input.position;
    output.normal = input.normal;
    output.texcoord = input.texcoord;
    return output;
}

float4 PSMain(PSInput input) : SV_TARGET {
    float3 albedo = albedoMap.Sample(defaultSampler, input.texcoord).rgb;
    float3 N = normalize(input.normal);
    float3 L = normalize(-lightDirection);
    float3 V = normalize(cameraPosition - input.worldPos);
    float3 H = normalize(L + V);

    float NdotL = max(dot(N, L), 0.0);
    float3 diffuse = albedo * lightColor * NdotL;

    return float4(diffuse, 1.0);
}
```

### Bazel Rule for Shader Cross-Compilation

```python
# build_tools/shaders.bzl

ShaderInfo = provider(
    fields = {
        "metal": "Metal shader for iOS/macOS",
        "spirv": "SPIR-V for Android/Vulkan",
        "dxil": "DXIL for Windows/DX12",
    },
)

def _shader_library_impl(ctx):
    outputs = {}

    # Compile to Metal (iOS/macOS)
    metal_out = ctx.actions.declare_file(ctx.attr.name + ".metal")
    ctx.actions.run(
        executable = ctx.executable._shader_compiler,
        arguments = [
            "--input", ctx.file.src.path,
            "--output", metal_out.path,
            "--target", "metal",
            "--entry-vs", ctx.attr.vs_entry,
            "--entry-ps", ctx.attr.ps_entry,
        ],
        inputs = [ctx.file.src],
        outputs = [metal_out],
        progress_message = "Compiling shader to Metal: %s" % ctx.file.src.short_path,
    )
    outputs["metal"] = metal_out

    # Compile to SPIR-V (Android/Vulkan)
    spirv_out = ctx.actions.declare_file(ctx.attr.name + ".spv")
    ctx.actions.run(
        executable = ctx.executable._shader_compiler,
        arguments = [
            "--input", ctx.file.src.path,
            "--output", spirv_out.path,
            "--target", "spirv",
            "--entry-vs", ctx.attr.vs_entry,
            "--entry-ps", ctx.attr.ps_entry,
        ],
        inputs = [ctx.file.src],
        outputs = [spirv_out],
        progress_message = "Compiling shader to SPIR-V: %s" % ctx.file.src.short_path,
    )
    outputs["spirv"] = spirv_out

    # Compile to DXIL (Windows/DX12)
    dxil_out = ctx.actions.declare_file(ctx.attr.name + ".dxil")
    ctx.actions.run(
        executable = ctx.executable._shader_compiler,
        arguments = [
            "--input", ctx.file.src.path,
            "--output", dxil_out.path,
            "--target", "dxil",
            "--entry-vs", ctx.attr.vs_entry,
            "--entry-ps", ctx.attr.ps_entry,
        ],
        inputs = [ctx.file.src],
        outputs = [dxil_out],
        progress_message = "Compiling shader to DXIL: %s" % ctx.file.src.short_path,
    )
    outputs["dxil"] = dxil_out

    return [
        DefaultInfo(files = depset(outputs.values())),
        ShaderInfo(**outputs),
    ]

shader_library = rule(
    implementation = _shader_library_impl,
    attrs = {
        "src": attr.label(
            allow_single_file = [".hlsl", ".glsl", ".shader"],
            mandatory = True,
        ),
        "vs_entry": attr.string(default = "VSMain"),
        "ps_entry": attr.string(default = "PSMain"),
        "_shader_compiler": attr.label(
            executable = True,
            cfg = "exec",  # Compiler runs on BUILD machine
            default = "//tools:shader_compiler",
        ),
    },
)
```

### BUILD.bazel Usage

```python
load("//build_tools:shaders.bzl", "shader_library")

shader_library(
    name = "pbr_lighting",
    src = "pbr_lighting.hlsl",
    vs_entry = "VSMain",
    ps_entry = "PSMain",
)

# Platform-specific game builds
ios_application(
    name = "game_ios",
    deps = [":pbr_lighting"],  # Uses .metal
)

android_binary(
    name = "game_android",
    deps = [":pbr_lighting"],  # Uses .spv
)

windows_binary(
    name = "game_windows",
    deps = [":pbr_lighting"],  # Uses .dxil
)
```

### Shader Cross-Compilation Flow

```
                     pbr_lighting.hlsl (CONTRACT/SSOT)
                              │
                              ▼
                    ┌─────────────────┐
                    │ shader_compiler │
                    │   (cfg=exec)    │
                    │ runs on macOS   │
                    └────────┬────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ .metal      │       │ .spv        │       │ .dxil       │
│ (Metal)     │       │ (SPIR-V)    │       │ (DirectX)   │
└──────┬──────┘       └──────┬──────┘       └──────┬──────┘
       ▼                     ▼                     ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ iOS/macOS   │       │ Android     │       │ Windows     │
│ Metal GPU   │       │ Vulkan GPU  │       │ DX12 GPU    │
└─────────────┘       └─────────────┘       └─────────────┘
```

---

## Complete Architecture: Game Engine Build

Combining both patterns into a full game engine build:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CONTRACT DOCUMENTS (SSOT)                         │
│                                                                          │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                │
│   │ .proto       │   │ .hlsl        │   │ .fbs         │                │
│   │ (network)    │   │ (shaders)    │   │ (assets)     │                │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                │
└──────────┼──────────────────┼──────────────────┼────────────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     BUILD MACHINE (cfg=exec tools)                       │
│                                                                          │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                │
│   │ protoc       │   │ shader_cc    │   │ flatc        │                │
│   │ (x86_64)     │   │ (x86_64)     │   │ (x86_64)     │                │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                │
└──────────┼──────────────────┼──────────────────┼────────────────────────┘
           │                  │                  │
           ├──────────────────┼──────────────────┤
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         GENERATED CODE                                   │
│                                                                          │
│   ┌─────────┬─────────┬─────────┐   ┌─────────┬─────────┬─────────┐    │
│   │ .cs     │ .rs     │ .py     │   │ .metal  │ .spv    │ .dxil   │    │
│   │ (C#)    │ (Rust)  │ (Python)│   │ (Metal) │ (Vulkan)│ (DX12)  │    │
│   └────┬────┴────┬────┴────┬────┘   └────┬────┴────┬────┴────┬────┘    │
└────────┼─────────┼─────────┼─────────────┼─────────┼─────────┼──────────┘
         │         │         │             │         │         │
         ▼         ▼         ▼             ▼         ▼         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         TARGET PLATFORMS                                 │
│                                                                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐            │
│  │ iOS       │  │ Android   │  │ Windows   │  │ Server    │            │
│  │ Unity+C#  │  │ Unity+C#  │  │ Unity+C#  │  │ Rust      │            │
│  │ Metal     │  │ Vulkan    │  │ DX12      │  │ Linux     │            │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

| Principle | Description |
|-----------|-------------|
| **Contract First** | Define `.proto`, `.hlsl`, `.fbs` before implementation |
| **cfg=exec for Tools** | Code generators run on build machine, not target |
| **One Source, Many Outputs** | Single contract generates C#, Rust, Python, Metal, SPIR-V |
| **Hermetic Builds** | All tools versioned in Bazel, no host dependencies |
| **Platform Abstraction** | Game code uses generated types, never raw bytes |

---

## Comparison: Templating Approaches

| Approach | Contract | Generator | Targets | Use Case |
|----------|----------|-----------|---------|----------|
| Ansible + Jinja | YAML vars | Jinja2 | Config files | Infrastructure |
| Bazel + Protobuf | .proto | protoc | C#, Rust, Python, Go | Networking |
| Bazel + Shaders | .hlsl | DXC/SPIRV-Cross | Metal, SPIR-V, DXIL | Graphics |
| Bazel + FlatBuffers | .fbs | flatc | C++, C#, Rust | Game assets |

---

## Tools for Shader Cross-Compilation

| Tool | Input | Output | Notes |
|------|-------|--------|-------|
| **DXC** | HLSL | SPIR-V, DXIL | Microsoft's open-source compiler |
| **SPIRV-Cross** | SPIR-V | GLSL, MSL, HLSL | Khronos tool for conversion |
| **glslc** | GLSL | SPIR-V | Google's GLSL compiler |
| **metal** | Metal | AIR | Apple's shader compiler |
| **ShaderConductor** | HLSL | All | Microsoft's cross-compiler |

---

## Gists

| Gist | Description | Link |
|------|-------------|------|
| 1 | Multi-language Protobuf rule | [proto_multi_lang.bzl](https://gist.github.com/srikantharun/baafe67b43476a5d3022252a0c07416a) |
| 2 | Shader cross-compilation rule | [shader_library.bzl](https://gist.github.com/srikantharun/0ca7d45e5bfe7b08575f5063c7ef9a8c) |

---

*Building cross-platform games with Bazel? Let's connect on [LinkedIn](https://linkedin.com/) or [X](https://x.com/).*
