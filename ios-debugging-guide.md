---
layout: post
title: "iOS Debugging Deep Dive: Debug Symbols, IDE Setup, and Crash Troubleshooting"
date: 2026-03-07
categories: [ios, debugging, lldb, bazel, xcode]
---

**Understanding Apple's Debug Symbol Architecture, VSCode Integration, and Bazel Workflows**

Debugging iOS applications presents unique challenges compared to Linux or Windows. Apple's "lazy" DWARF scheme, the distinction between dSYM bundles and N_OSO debug maps, and the lack of `--fission` support create a learning curve for developers coming from other platforms. This guide distills practical knowledge for debugging iOS simulator and device applications, with emphasis on Bazel-based build systems and VSCode integration.

---

## The Apple Debug Symbol Constraint

### Why `--fission` and `.dwo` Files Don't Work on macOS

One of the most common misconceptions when moving from Linux to macOS development:

> **`--fission` and `.dwo` files are LINUX ONLY, not supported on macOS**

On Linux, the `-gsplit-dwarf` flag (exposed via Bazel's `--fission`) produces separate `.dwo` files containing debug information, which can later be combined into `.dwp` (DWARF package) files. This reduces link times and enables distributed debug info.

**macOS uses a fundamentally different approach:**

| Feature | Linux | macOS/iOS |
|---------|-------|-----------|
| Split debug format | `.dwo` files | Not supported |
| Package format | `.dwp` files | `.dSYM` bundles |
| Debug info location | Separate files | Inside `.o` files OR dSYM |
| Linker behavior | Combines debug sections | Creates debug map entries |
| Debug tool | `dwp` utility | `dsymutil` |

As documented in [MaskRay's analysis](https://maskray.me/blog/2022-10-30-distribution-of-debug-information), on macOS `-gsplit-dwarf` has different behavior and will not produce `.dwo` files.

### Apple's "Lazy" DWARF Scheme

Apple developed a unique approach to debug information that prioritizes lazy loading. From the [DWARF Standards Wiki](https://wiki.dwarfstd.org/Apple's_%22Lazy%22_DWARF_Scheme.md):

```
┌─────────────────────────────────────────────────────────────────┐
│                    Apple Debug Symbol Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Source Files          Object Files           Final Binary     │
│   ┌─────────┐           ┌─────────┐           ┌─────────┐       │
│   │  a.c    │──compile──│  a.o    │           │         │       │
│   │  b.c    │──compile──│  b.o    │──link────│  app    │       │
│   │  c.c    │──compile──│  c.o    │           │         │       │
│   └─────────┘           └─────────┘           └─────────┘       │
│                              │                     │             │
│                              │    N_OSO entries    │             │
│                              │◄────────────────────┤             │
│                              │   (debug map)       │             │
│                              │                     │             │
│                              ▼                     ▼             │
│                    ┌─────────────────────────────────┐          │
│                    │         dsymutil                │          │
│                    │  (creates .dSYM bundle)         │          │
│                    └─────────────────────────────────┘          │
│                                     │                            │
│                                     ▼                            │
│                           ┌─────────────────┐                   │
│                           │   app.dSYM      │                   │
│                           │  (standalone)   │                   │
│                           └─────────────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Two modes of operation:**

1. **N_OSO Debug Map Mode**: The linker creates `N_OSO` stab entries in `__LINKEDIT` pointing to original `.o` file paths. DWARF debug symbols live INSIDE the `.o` files. LLDB needs those `.o` files locally to resolve symbols at debug time.

2. **dSYM Bundle Mode**: `dsymutil` processes the debug map and `.o` files to create a standalone `.dSYM` bundle with all addresses remapped to final locations.

You can inspect N_OSO entries using:
```bash
dsymutil -s MyApp | grep N_OSO
```

### The UUID Safeguard

Every Apple binary is stamped with a 128-bit unique identifier (`LC_UUID` load command). This UUID is copied into the dSYM during creation, ensuring reliable matching:

```bash
# Check binary UUID
dwarfdump --uuid MyApp.app/MyApp

# Check dSYM UUID
dwarfdump --uuid MyApp.app.dSYM/Contents/Resources/DWARF/MyApp

# They must match for symbolication to work
```

---

## IDE Support for iOS Debugging

### The Challenge: No Native VSCode iOS Debugging

Unlike Android (which has excellent VSCode support), iOS debugging has traditionally required Xcode. However, several projects now enable VSCode-based iOS debugging.

### Option 1: vscode-ios-debug Extension

The [vscode-ios-debug](https://github.com/nisargjhaveri/vscode-ios-debug) extension provides:

- Pick connected device or installed simulator for debugging
- Seamless debugging via VSCode Remote Development
- All CodeLLDB debugging features

**Installation:**
```bash
code --install-extension nisargjhaveri.ios-debug
```

**Sample launch.json:**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "ios",
      "request": "launch",
      "name": "iOS: Launch App",
      "appPath": "${workspaceFolder}/bazel-bin/candycrushsaga_sim.app",
      "bundleId": "com.midasplayer.test.candycrushsaga",
      "iosTarget": "simulator"
    }
  ]
}
```

### Option 2: SweetPad Extension

[SweetPad](https://sweetpad.hyzyla.dev/docs/debug/) provides lightweight CodeLLDB integration for iOS:

**Setup steps:**

1. Install CodeLLDB extension
2. Configure LLDB backend in `settings.json`:
```json
{
  "lldb.library": "/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Versions/A/LLDB"
}
```
3. Press F5 to build, launch simulator, and attach debugger

### Option 3: CodeLLDB Direct Configuration

For Bazel-built iOS apps, configure [CodeLLDB](https://github.com/vadimcn/codelldb) directly:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "attach",
      "name": "Attach to iOS Simulator",
      "pid": "${command:pickProcess}",
      "initCommands": [
        "platform select ios-simulator",
        "settings set target.source-map ./ ${workspaceFolder}/"
      ],
      "sourceMap": {
        ".": "${workspaceFolder}"
      }
    }
  ]
}
```

### Troubleshooting: Breakpoints Show "Resolved locations: 0"

Per [Yrom's debugging guide](https://yrom.net/blog/2023/04/22/debug-ios-app-in-vscode/), this usually means debug symbol object paths don't match VSCode source paths. Solutions:

1. Set correct `lldb.library` path
2. Configure source mapping in launch.json
3. Ensure dSYM is generated and accessible

---

## Bazel iOS Debugging Configuration

### The `setup_msvc` Clarification

**Important:** `bazel run //bz-tools:setup_msvc -- release live --clean --open` is **NOT for iOS**. This command generates Windows Visual Studio solutions and is completely irrelevant for Apple platform development.

For iOS/macOS development in this codebase:
- Use `bazel run //bz-tools:setup_vscode` for VSCode configuration
- Use `xcodeproj` targets for Xcode project generation
- iOS simulator: `bazel run //:install_ios_sim`

### Generating dSYM Files with rules_apple

Per [rules_apple documentation](https://github.com/bazelbuild/rules_apple/blob/master/doc/common_info.md), enable dSYM generation:

```bash
# Generate dSYM for top-level target
bazel build //ios:candycrushsaga_sim \
  --apple_generate_dsym \
  --ios_multi_cpus=arm64

# Generate dSYMs for all dependencies
bazel build //ios:candycrushsaga_sim \
  --apple_generate_dsym \
  --output_groups=+dsyms
```

### Debug Configuration in .bazelrc

```bash
# Recommended debug configuration
build:debug --spawn_strategy=local
build:debug --compilation_mode=dbg
build:debug --strip=never
build:debug --features=oso_prefix_is_pwd
build:debug -c dbg

# iOS-specific debug settings
build:ios-debug --config=debug
build:ios-debug --apple_generate_dsym
build:ios-debug --define=apple.add_debugger_entitlement=yes
```

### The `oso_prefix_is_pwd` Feature

This critical feature addresses Bazel sandbox path issues. From [Keith Smiley's analysis](https://www.smileykeith.com/2025/09/21/understanding-apple-debug-info/):

The `-oso_prefix` linker argument strips sandbox paths from N_OSO entries, replacing them with `.` (relative to current directory). Without this, LLDB cannot find `.o` files because paths point to deleted sandbox directories.

### LLDB Settings for Bazel Builds

Based on [rules_ios LLDB settings](https://github.com/bazel-ios/rules_ios/blob/master/tools/xcodeproj_shims/installers/lldb-settings.sh):

```bash
# ~/.lldbinit for Bazel projects
settings set target.source-map ./ /path/to/workspace/
platform settings -w /path/to/workspace/
settings set target.sdk-path /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk
```

---

## Crash Symbolication Workflow

### Manual Symbolication with `atos`

When you have a crash address and need to symbolicate manually:

```bash
# Basic usage
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x1003d8000 0x1003e2abc

# With inlined function support (-i flag)
atos -arch arm64 -i -o MyApp.app/MyApp -l 0x100000000 0x100001234
```

Per [Nutrient's advanced symbolication guide](https://www.nutrient.io/guides/ios/troubleshooting/advanced-symbolication/):

- **Load address** (`-l`): The base address where the binary was loaded (from crash log)
- **Symbol address**: The actual crash address to symbolicate
- **Architecture** (`-arch`): Must match the crashed device (arm64, x86_64)

### Xcode Organizer Workflow

1. Ensure crash files have `.crash` extension
2. Verify dSYM UUID matches binary UUID
3. Import to Xcode Organizer
4. Xcode auto-symbolicates if dSYM is in Spotlight index

### Symbolicator Script (Project-Specific)

The codebase includes a Python symbolicator at `tools/scripts/Symbolicator/Symbolicator.py`:

```python
# Usage pattern
python Symbolicator.py \
  --dsym candycrushsaga.app.dSYM.tar.gz \
  --crash crash_report.txt \
  --output symbolicated_crash.txt
```

This tool:
- Loads dSYM from tar.gz archives
- Matches dSYM UUID to crash log UUID
- Produces symbolicated crash logs with readable function names

### Troubleshooting Symbolication Failures

| Symptom | Cause | Solution |
|---------|-------|----------|
| Addresses not resolved | UUID mismatch | Verify UUIDs with `dwarfdump --uuid` |
| Partial symbolication | Missing framework dSYMs | Build with `--output_groups=+dsyms` |
| "No matching arch" | Wrong architecture | Check `file` command output, use correct `-arch` |
| dSYM not found | Spotlight not indexed | Run `mdimport MyApp.app.dSYM` |

---

## Command-Line iOS Debugging (Without Xcode UI)

### ios-deploy

The [ios-deploy](https://github.com/ios-control/ios-deploy) tool enables command-line debugging:

```bash
# Install
brew install ios-deploy

# Install and debug on device
ios-deploy --debug --bundle MyApp.app

# Launch with LLDB attached
ios-deploy -d -b MyApp.app
```

### simctl + LLDB Direct Attach

```bash
# Boot simulator
xcrun simctl boot "iPhone 15 Pro"

# Install app
xcrun simctl install booted ./bazel-bin/candycrushsaga_sim.app

# Launch app
xcrun simctl launch booted com.midasplayer.test.candycrushsaga

# Attach LLDB
lldb
(lldb) platform select ios-simulator
(lldb) process attach --name candycrushsaga --waitfor
```

### Remote Debugging Setup

For debugging from another machine or advanced scenarios:

```bash
# On device/simulator (via debugserver)
debugserver *:4445 --attach=PID

# On development machine
lldb
(lldb) platform select remote-ios
(lldb) process connect connect://localhost:4445
```

---

## Practical Debugging Recipes

### Recipe 1: Debug Bazel-Built iOS Simulator App

```bash
# 1. Build with debug symbols
bazel build //:candycrushsaga_sim \
  --config=ios-debug \
  --apple_generate_dsym

# 2. Install to simulator
bazel run //:install_ios_sim

# 3. Launch simulator app
xcrun simctl launch booted com.midasplayer.test.candycrushsaga

# 4. Attach with LLDB
lldb
(lldb) process attach --name candycrushsaga
(lldb) settings set target.source-map ./ /path/to/candycrushsaga/
(lldb) breakpoint set --file GameScene.cpp --line 100
(lldb) continue
```

### Recipe 2: Symbolicate Production Crash

```bash
# 1. Extract dSYM from build artifacts
tar -xzf candycrushsaga.app.dSYM.tar.gz

# 2. Verify UUID match
dwarfdump --uuid candycrushsaga.app.dSYM

# 3. Symbolicate crash address
atos -arch arm64 -i \
  -o candycrushsaga.app.dSYM/Contents/Resources/DWARF/candycrushsaga \
  -l 0x104a00000 \
  0x104a12345
```

### Recipe 3: VSCode iOS Simulator Debugging

1. Install extensions:
   - [iOS Debug](https://marketplace.visualstudio.com/items?itemName=nisargjhaveri.ios-debug)
   - [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)

2. Configure `settings.json`:
```json
{
  "lldb.library": "/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Versions/A/LLDB"
}
```

3. Create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "attach",
      "name": "Attach iOS Simulator",
      "program": "${workspaceFolder}/bazel-bin/candycrushsaga_sim.app/candycrushsaga",
      "pid": "${command:pickProcess}",
      "sourceMap": {
        ".": "${workspaceFolder}"
      },
      "initCommands": [
        "settings set target.source-map ./ ${workspaceFolder}/"
      ]
    }
  ]
}
```

4. Build, launch simulator, then F5 in VSCode to attach

---

## Common Pitfalls and Solutions

### Pitfall 1: "No debug info" After Bazel Build

**Cause:** Bazel sandbox paths in N_OSO entries point to deleted directories.

**Solution:**
```bash
# Enable OSO prefix rewriting
bazel build //target --features=oso_prefix_is_pwd
```

### Pitfall 2: dSYM Generated But LLDB Can't Use It

**Cause:** dSYM not in LLDB search path.

**Solution:**
```bash
(lldb) settings append target.debug-file-search-paths /path/to/dsyms/
(lldb) add-dsym /path/to/MyApp.app.dSYM
```

### Pitfall 3: Breakpoints Work But Variables Show `<unavailable>`

**Cause:** Optimization level too high, variables optimized out.

**Solution:**
```bash
# Build with -O0
bazel build //target -c dbg --copt=-O0
```

### Pitfall 4: Source Files Not Found

**Cause:** Compilation happened in different directory than debugging.

**Solution:**
```bash
(lldb) settings set target.source-map /original/compile/path /current/source/path
```

---

## Summary

Debugging iOS applications requires understanding Apple's unique debug symbol architecture:

1. **No `.dwo` support**: macOS uses N_OSO debug maps OR dSYM bundles, not split DWARF
2. **UUID matching**: Binaries and dSYMs are tied by 128-bit UUIDs
3. **Bazel considerations**: Use `--features=oso_prefix_is_pwd` and `--apple_generate_dsym`
4. **VSCode is possible**: Extensions like vscode-ios-debug and SweetPad enable non-Xcode debugging
5. **Command-line tools**: `ios-deploy`, `simctl`, and direct LLDB attachment work without Xcode UI

The `setup_msvc` command is Windows-only and irrelevant for iOS development. For iOS debugging in Bazel projects, focus on proper dSYM generation and LLDB source mapping.

---

## References

### Apple Debug Architecture
- [Apple's "Lazy" DWARF Scheme](https://wiki.dwarfstd.org/Apple's_%22Lazy%22_DWARF_Scheme.md)
- [Understanding Apple Debug Info - Keith Smiley](https://www.smileykeith.com/2025/09/21/understanding-apple-debug-info/)
- [LLDB Symbols on macOS](https://lldb.llvm.org/use/symbols.html)
- [Distribution of Debug Information - MaskRay](https://maskray.me/blog/2022-10-30-distribution-of-debug-information)
- [Building Your App to Include Debugging Information - Apple](https://developer.apple.com/documentation/xcode/building-your-app-to-include-debugging-information)

### VSCode iOS Debugging
- [vscode-ios-debug Extension](https://github.com/nisargjhaveri/vscode-ios-debug)
- [SweetPad Debugging Guide](https://sweetpad.hyzyla.dev/docs/debug/)
- [CodeLLDB Extension](https://github.com/vadimcn/codelldb)
- [Debug iOS App in VSCode - Yrom](https://yrom.net/blog/2023/04/22/debug-ios-app-in-vscode/)

### Crash Symbolication
- [Symbolication: Beyond the Basics - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10211/)
- [Advanced Crash Report Symbolication - Nutrient](https://www.nutrient.io/guides/ios/troubleshooting/advanced-symbolication/)
- [Symbolicating iOS Crash Reports - Medium](https://medium.com/the-swift-blog/symbolicating-ios-crash-reports-e97ad0d6b4dc)

### Bazel Integration
- [rules_apple dSYM Documentation](https://github.com/bazelbuild/rules_apple/blob/master/doc/common_info.md)
- [rules_apple dSYM Discussion](https://github.com/bazelbuild/rules_apple/discussions/1767)
- [rules_ios LLDB Settings](https://github.com/bazel-ios/rules_ios/blob/master/tools/xcodeproj_shims/installers/lldb-settings.sh)
- [Bazel LLDB Debugging Issue](https://github.com/bazelbuild/bazel/issues/26230)

### Command-Line Tools
- [ios-deploy](https://github.com/ios-control/ios-deploy)
- [LLDB Documentation](https://lldb.llvm.org/)

