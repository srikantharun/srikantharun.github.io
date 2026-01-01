# Hermetic Clang Toolchains Across All Platforms

**Solving Sysroot & C++23 Compatibility on Linux, macOS, Android & iOS**

---

Building C++ with Bazel across heterogeneous platforms—Ubuntu, Rocky Linux 8, macOS, Android, and iOS—presents unique challenges. Our monorepo uses C++23 features like `std::expected` and `std::format`. Getting these to work hermetically across all targets required solving sysroot mismatches, standard library gaps, and SDK alignment issues.

## Platform Overview

| Platform | Architectures | Sysroot/SDK | C++ stdlib | Challenge |
|----------|--------------|-------------|------------|-----------|
| Ubuntu 24.04 | x86_64, ARM64 | Ubuntu sysroot | libstdc++ | Baseline |
| Rocky 8 | x86_64, ARM64 | Rocky sysroot | **libc++** | Missing C++23 in GCC 10 |
| macOS 15 | ARM64 | Xcode SDK | libc++ | SDK version alignment |
| Android | arm64-v8a, x86_64 | NDK r26+ | libc++ | API level selection |
| iOS | arm64, simulator | Xcode SDK | libc++ | Device vs simulator |

---

## Part 1: Linux — Ubuntu vs Rocky 8

### The Problem

Ubuntu 24.04 ships glibc 2.39 and GCC 13; Rocky 8 has glibc 2.28 and GCC 10. Building with an Ubuntu sysroot on Rocky 8 produces runtime failures:

```
./binary: /lib64/libc.so.6: version `GLIBC_2.34' not found
```

Rocky 8's `libstdc++` also lacks C++20/23 features: `std::is_scoped_enum_v`, `std::bit_cast`, `std::expected`, `std::format`.

### Solution: Auto-Detect Distro + Per-Distro Sysroots

We detect the Linux distribution at Bazel fetch time by reading `/etc/os-release`:

```python
# Excerpt from linux_distro.bzl
os_release = rctx.read("/etc/os-release")
if "Rocky" in os_release:
    distro = "rocky"
elif "Ubuntu" in os_release:
    distro = "ubuntu"
```

Each distro gets its own toolchain with matching sysroot. Bazel's constraint system automatically selects the right one.

### Solution: Switch Rocky 8 to libc++

LLVM's `libc++` ships full C++23 support regardless of host GCC version. We rebuild Rocky's sysroot *without* `libstdc++` headers:

```bash
rm -rf "${SYSROOT}/usr/include/c++"
```

Then configure the toolchain:

```python
llvm.toolchain(
    name = "llvm_toolchain_rocky",
    stdlib = {"": "builtin-libc++"},
    cxx_flags = {"": ["-stdlib=libc++"]},
)
```

**See:** [Gist 1: linux_distro.bzl](https://gist.github.com/srikantharun/655bb0654064e18afed64cda9bb2998e) | [Gist 2: rebuild_sysroot.sh](https://gist.github.com/srikantharun/6e94d98e6700facddfa85811836a0275)

---

## Part 2: macOS — Xcode SDK Alignment

### The Setup

macOS uses Xcode's SDK as sysroot and `libc++` by default—no `libstdc++` conflicts. The challenge is SDK version alignment:

```python
# .bazelrc
build:macos --apple_platform_type=macos
build:macos --macos_minimum_os=15.0
build:macos --macos_sdk_version=15.0
build:macos_arm64 --cpu=darwin_arm64
```

### Key Points

- Use `-isysroot $(xcrun --show-sdk-path)` implicitly via Bazel's Apple rules
- Match `macos_minimum_os` with deployment target
- C++23 works out of the box with Apple Clang or LLVM Clang

---

## Part 3: Android — NDK Cross-Compilation

### The Setup

Android NDK ships Clang + libc++ with full C++23 support. No `libstdc++` conflicts here either—Google made the right choice years ago.

```python
# MODULE.bazel
bazel_dep(name = "rules_android_ndk", version = "0.1.2")

android_ndk = use_extension(
    "@rules_android_ndk//:extension.bzl",
    "android_ndk",
)
android_ndk.configure(api_level = 24)  # Android 7.0+
use_repo(android_ndk, "androidndk")
```

### Platform Definitions

```python
# build_tools/platform/BUILD.bazel
platform(
    name = "android_arm64",
    constraint_values = [
        "@platforms//os:android",
        "@platforms//cpu:aarch64",
    ],
)

platform(
    name = "android_x86_64",
    constraint_values = [
        "@platforms//os:android",
        "@platforms//cpu:x86_64",
    ],
)
```

### Building for Android

```bash
# Build for Android ARM64
bazel build //app:mylib --platforms=//build_tools/platform:android_arm64

# Build for Android x86_64 (emulator)
bazel build //app:mylib --platforms=//build_tools/platform:android_x86_64
```

### Key Points

- **API Level**: Determines minimum Android version and available libc features
- **NDK Version**: r26+ recommended for C++23 (`__cpp_lib_expected`)
- **ABI**: arm64-v8a for modern devices, x86_64 for emulators
- **No sysroot rebuilding needed**: NDK sysroot is already hermetic

**See:** [Gist 3: android_toolchain.bzl](https://gist.github.com/srikantharun/c7390d59c97bf65685c61707e39e60fd)

---

## Part 4: iOS — Device vs Simulator

### The Challenge

iOS requires separate builds for:
- **Device** (arm64): Real iPhones/iPads
- **Simulator** (arm64 on Apple Silicon, x86_64 on Intel Macs)

### Configuration

```python
# .bazelrc
# iOS Device (ARM64)
build:ios_device --apple_platform_type=ios
build:ios_device --ios_minimum_os=15.0
build:ios_device --cpu=ios_arm64

# iOS Simulator (ARM64 for Apple Silicon)
build:ios_simulator --apple_platform_type=ios
build:ios_simulator --ios_minimum_os=15.0
build:ios_simulator --cpu=ios_sim_arm64

# iOS Simulator (x86_64 for Intel Macs)
build:ios_simulator_x86 --apple_platform_type=ios
build:ios_simulator --ios_minimum_os=15.0
build:ios_simulator_x86 --cpu=ios_x86_64
```

### Platform Definitions

```python
platform(
    name = "ios_arm64",
    constraint_values = [
        "@platforms//os:ios",
        "@platforms//cpu:aarch64",
        ":ios_device",
    ],
)

platform(
    name = "ios_sim_arm64",
    constraint_values = [
        "@platforms//os:ios",
        "@platforms//cpu:aarch64",
        ":ios_simulator",
    ],
)
```

### Building for iOS

```bash
# Build for iOS device
bazel build //app:mylib --config=ios_device

# Build for iOS simulator
bazel build //app:mylib --config=ios_simulator

# Build universal framework (device + simulator)
bazel build //app:MyFramework_universal
```

### Key Points

- **Code Signing**: Required for device builds, automatic for simulator
- **Bitcode**: Deprecated as of Xcode 14, no longer needed
- **Minimum iOS**: 15.0+ recommended for modern Swift interop
- **libc++**: Default on iOS, full C++23 support with Xcode 15+

**See:** [Gist 4: ios_build_config.bazelrc](https://gist.github.com/srikantharun/3f310e834237d5f50b12bd090a31565e)

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    Bazel Build System                           │
├─────────────────────────────────────────────────────────────────┤
│  Host Detection (fetch time)                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ /etc/os-rel │  │ uname -s    │  │ xcrun       │             │
│  │ Ubuntu?Rocky│  │ Darwin?     │  │ --sdk-path  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
├─────────┼────────────────┼────────────────┼─────────────────────┤
│         ▼                ▼                ▼                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Toolchain Selection                         │   │
│  ├──────────────┬──────────────┬──────────────┬────────────┤   │
│  │ Ubuntu       │ Rocky 8      │ macOS        │ Cross      │   │
│  │ libstdc++    │ libc++       │ libc++       │ Compile    │   │
│  │ glibc 2.39   │ glibc 2.28   │ Xcode SDK    │ Android/iOS│   │
│  └──────────────┴──────────────┴──────────────┴────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  Target Platforms                                               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │Linux   │ │Linux   │ │macOS   │ │Android │ │iOS     │        │
│  │x86_64  │ │ARM64   │ │ARM64   │ │arm64-v8│ │arm64   │        │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Never assume glibc/libstdc++ compatibility** — glibc 2.28 vs 2.39 breaks binaries at runtime.

2. **Per-distro hermetic sysroots** beat universal sysroots — auto-detect at fetch time.

3. **libc++ enables C++23 everywhere** — Rocky 8, Android, iOS all benefit from LLVM's modern stdlib.

4. **Android/iOS are easier** — Both use libc++ by default; no stdlib conflicts.

5. **Automate platform detection** — Developers shouldn't need `--config=rocky` or `--config=android_arm64`.

---

## Gists

| Gist | Description | Link |
|------|-------------|------|
| 1 | Linux distro auto-detection | [linux_distro.bzl](https://gist.github.com/srikantharun/655bb0654064e18afed64cda9bb2998e) |
| 2 | Rocky 8 sysroot rebuild (no libstdc++) | [rebuild_sysroot.sh](https://gist.github.com/srikantharun/6e94d98e6700facddfa85811836a0275) |
| 3 | Android NDK toolchain setup | [android_toolchain.bzl](https://gist.github.com/srikantharun/c7390d59c97bf65685c61707e39e60fd) |
| 4 | iOS device/simulator configs | [ios_build_config.bazelrc](https://gist.github.com/srikantharun/3f310e834237d5f50b12bd090a31565e) |

---

*Building cross-platform C++ with Bazel? Let's connect on [LinkedIn](https://linkedin.com/) or [X](https://x.com/).*
