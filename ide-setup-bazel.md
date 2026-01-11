# VS Code vs CLion for Bazel C++ Monorepos

**Choosing the Right IDE for Large-Scale C++ Development**

---

When working on large C++ monorepos with Bazel, IDE choice significantly impacts productivity. This post compares VS Code (with clangd) and CLion (with the Bazel plugin) for real-world Bazel development—covering setup, performance, and daily workflows.

## Quick Comparison

| Aspect | VS Code + clangd | CLion + Bazel Plugin |
|--------|------------------|---------------------|
| **Cost** | Free | $249/year (or free for OSS) |
| **Setup Complexity** | Medium | Low |
| **Index Speed** | Fast (clangd) | Slower (full project sync) |
| **Memory Usage** | ~2-4GB | ~4-8GB+ |
| **Refactoring** | Basic | Advanced |
| **Debugging** | Via CodeLLDB | Native, excellent |
| **Bazel Integration** | Manual (compile_commands.json) | Native plugin |
| **Remote Dev** | Excellent (SSH, containers) | Good (Gateway) |

---

## Part 1: VS Code + clangd Setup

VS Code with clangd provides fast, lightweight C++ intelligence by leveraging `compile_commands.json` generated from Bazel.

### Step 1: Install Extensions

```
- clangd (llvm-vs-code-extensions.vscode-clangd)
- CodeLLDB (for debugging)
- Bazel (optional, for BUILD file syntax)
```

### Step 2: Generate compile_commands.json

Use [hedron's compile-commands-extractor](https://github.com/hedronvision/bazel-compile-commands-extractor):

```python
# MODULE.bazel
bazel_dep(name = "hedron_compile_commands", dev_dependency = True)
git_override(
    module_name = "hedron_compile_commands",
    remote = "https://github.com/hedronvision/bazel-compile-commands-extractor.git",
    commit = "4f28899228fb3ad0126897876f147ca15026151e",
)
```

```python
# BUILD.bazel (workspace root)
load("@hedron_compile_commands//:refresh_compile_commands.bzl", "refresh_compile_commands")

refresh_compile_commands(
    name = "refresh_compile_commands",
    targets = {
        "//src/...": "",
        "//lib/...": "",
    },
)
```

Generate and refresh:

```bash
bazel run //:refresh_compile_commands
```

### Step 3: Configure clangd

Create `.clangd` in your workspace root:

```yaml
CompileFlags:
  Add:
    - "-Wno-unknown-warning-option"
  Remove:
    - "-fno-canonical-system-headers"

InlayHints:
  Enabled: true
  ParameterNames: true
  DeducedTypes: true

Diagnostics:
  UnusedIncludes: Strict
```

### Step 4: VS Code Settings

```json
{
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=iwyu",
    "--completion-style=detailed",
    "-j=8"
  ],
  "clangd.path": "/usr/bin/clangd",
  "files.watcherExclude": {
    "**/bazel-*/**": true
  }
}
```

### Common Issues & Fixes

**Issue: "argument unused during compilation: '-c'"**

This warning appears when clangd queries headers. Fixed in [PR #276](https://github.com/hedronvision/bazel-compile-commands-extractor/pull/276).

**Issue: Cross-platform build fails with missing toolchain**

When using `--platforms` for cross-compilation, `refresh_compile_commands` fails. Fixed by adding `manual` tag in [PR #276](https://github.com/hedronvision/bazel-compile-commands-extractor/pull/276).

**Issue: Headers not found after adding new files**

Re-run `bazel run //:refresh_compile_commands` to regenerate the compile commands.

---

## Part 2: CLion + Bazel Plugin Setup

CLion with the official Bazel plugin provides deep integration but requires more resources.

### Step 1: Install Bazel Plugin

`Settings → Plugins → Marketplace → "Bazel for CLion"`

### Step 2: Import Project

1. `File → Import Bazel Project`
2. Select workspace root
3. Choose targets to sync (or use project view file)

### Step 3: Create Project View (.bazelproject)

```
directories:
  src
  lib
  include

targets:
  //src/...
  //lib/...

additional_languages:
  python

build_flags:
  --config=clang
```

### Step 4: Sync Project

`Bazel → Sync Project with BUILD Files`

This can take 10-30+ minutes on large monorepos.

### Optimizing CLion Performance

```
# Help → Edit Custom VM Options
-Xms2g
-Xmx8g
-XX:+UseG1GC
-XX:ReservedCodeCacheSize=512m
```

Reduce sync scope in `.bazelproject`:

```
# Only sync what you're actively working on
targets:
  //src/myproject/...
  -//src/myproject/tests/...
```

---

## Part 3: Feature Comparison

### Code Navigation

| Feature | VS Code + clangd | CLion |
|---------|------------------|-------|
| Go to Definition | ✅ Fast | ✅ Fast |
| Find Usages | ✅ Good | ✅ Excellent (cross-language) |
| Call Hierarchy | ✅ | ✅ |
| Type Hierarchy | ❌ Limited | ✅ |
| Include Hierarchy | ✅ | ✅ |

### Refactoring

| Feature | VS Code + clangd | CLion |
|---------|------------------|-------|
| Rename | ✅ | ✅ Better cross-file |
| Extract Function | ❌ | ✅ |
| Extract Variable | ❌ | ✅ |
| Change Signature | ❌ | ✅ |
| Move to File | ❌ | ✅ |

### Debugging

| Feature | VS Code (CodeLLDB) | CLion |
|---------|-------------------|-------|
| Breakpoints | ✅ | ✅ |
| Conditional Breakpoints | ✅ | ✅ |
| Watch Expressions | ✅ | ✅ |
| Memory View | ❌ | ✅ |
| Disassembly | ✅ Limited | ✅ Excellent |
| Bazel Target Debug | Manual setup | ✅ Native |

### Bazel Integration

| Feature | VS Code | CLion |
|---------|---------|-------|
| BUILD Syntax | Extension | ✅ Native |
| Run Targets | Terminal | ✅ UI |
| Debug Targets | Manual | ✅ One-click |
| Query/cquery | Terminal | ✅ UI |
| Build on Save | Manual | ✅ Configurable |

---

## Part 4: Recommendations

### Choose VS Code + clangd if:

- **Working on a subset** of a large monorepo
- **Memory constrained** (laptops, remote VMs)
- **Fast iteration** is priority (clangd indexes quickly)
- **Remote development** via SSH/containers
- **Cost matters** (free)
- **Cross-platform** (same setup on Linux/macOS/Windows)

### Choose CLion if:

- **Heavy refactoring** is common
- **Debugging complex issues** regularly
- **Team standardization** needed
- **Full project understanding** required
- **Budget available** for licenses
- **Java/Kotlin/Python** mixed with C++ (JetBrains ecosystem)

### Hybrid Approach

Many teams use both:

1. **VS Code** for quick edits, code review, remote work
2. **CLion** for deep debugging, complex refactoring

Both can coexist—VS Code uses `compile_commands.json` while CLion uses its own index.

---

## Part 5: Automation & CI Integration

### Pre-commit Hook for compile_commands.json

```bash
#!/bin/bash
# .git/hooks/post-merge
if git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD | grep -qE '(BUILD|\.bzl)$'; then
    echo "BUILD files changed, refreshing compile_commands.json..."
    bazel run //:refresh_compile_commands 2>/dev/null &
fi
```

### CI Validation

```yaml
# .github/workflows/ide.yml
name: IDE Config Validation
on: [pull_request]
jobs:
  compile-commands:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate compile_commands.json
        run: bazel run //:refresh_compile_commands
      - name: Verify no errors
        run: |
          # Check clangd can parse the compilation database
          clangd --check=src/main.cpp --log=error
```

---

## Summary

| Scenario | Recommendation |
|----------|----------------|
| Large monorepo, many contributors | **CLion** (standardization) |
| Small team, fast iteration | **VS Code** (lightweight) |
| Remote development | **VS Code** (excellent SSH) |
| Heavy debugging | **CLion** (native debugger) |
| Cost-sensitive | **VS Code** (free) |
| Mixed language (C++/Java/Python) | **CLion** (JetBrains ecosystem) |

Both setups work well with Bazel. The key is generating accurate `compile_commands.json` for VS Code or maintaining a focused `.bazelproject` for CLion.

---

**Related Posts:**
- [Hermetic Clang Toolchains Across All Platforms](./hermetic-clang-toolchains)
- [Cross-Compilation Deep Dive](./cross-compilation)

**Tools Referenced:**
- [hedron/bazel-compile-commands-extractor](https://github.com/hedronvision/bazel-compile-commands-extractor)
- [bazelbuild/intellij](https://github.com/bazelbuild/intellij) (CLion plugin)
- [clangd](https://clangd.llvm.org/)
