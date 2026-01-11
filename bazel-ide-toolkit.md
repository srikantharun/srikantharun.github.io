---
layout: post
title: "Bazel IDE Toolkit: Seamless C++ IntelliSense for Bazel Projects"
date: 2025-01-11
categories: [bazel, vscode, tooling]
---

# Bazel IDE Toolkit: Seamless C++ IntelliSense for Bazel Projects

One of the biggest pain points for developers working with Bazel-based C++ projects is getting proper IDE support. Unlike CMake which generates `compile_commands.json` natively, Bazel requires additional tooling to enable features like code completion, go-to-definition, and real-time error checking.

I've published **[Bazel IDE Toolkit](https://marketplace.visualstudio.com/items?itemName=srikantharun.bazel-ide-toolkit)** - a VS Code extension that automatically keeps your `compile_commands.json` in sync with your Bazel build graph.

## The Problem

When you open a C++ file in a Bazel project without `compile_commands.json`:
- No autocomplete suggestions
- `#include` paths show red squiggles
- Go-to-definition doesn't work
- No real-time error checking

This happens because clangd (the language server) has no idea how Bazel organizes your build - the include paths, compiler flags, and dependencies are all managed by Bazel's sandboxed build system.

## The Solution

The Bazel IDE Toolkit extension:
1. **Watches** for changes to BUILD files, `.bzl` files, WORKSPACE, and MODULE.bazel
2. **Automatically regenerates** `compile_commands.json` when changes are detected
3. **Shows status** in the VS Code status bar so you know when refresh is happening

## Setup Guide

### Step 1: Add hedron_compile_commands to MODULE.bazel

Add the following to your `MODULE.bazel`:

```starlark
bazel_dep(name = "hedron_compile_commands", dev_dependency = True)
git_override(
    module_name = "hedron_compile_commands",
    remote = "https://github.com/hedronvision/bazel-compile-commands-extractor.git",
    commit = "4f28899228fb3ad0126897876f147ca15026151e",
)
```

### Step 2: Add refresh target to BUILD.bazel

Add to your root `BUILD.bazel`:

```starlark
load("@hedron_compile_commands//:refresh_compile_commands.bzl", "refresh_compile_commands")

refresh_compile_commands(
    name = "refresh_compile_commands",
    targets = {
        "//...": "",
    },
)
```

For large monorepos, you may want to specify specific targets instead of `//...`:

```starlark
refresh_compile_commands(
    name = "refresh_compile_commands",
    exclude_external_sources = True,
    targets = {
        "//src/...": "",
        "//lib/...": "",
        "//tools/...": "",
    },
)
```

### Step 3: Install VS Code Extensions

Install two extensions:

1. **[clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd)** - The language server that provides IntelliSense
2. **[Bazel IDE Toolkit](https://marketplace.visualstudio.com/items?itemName=srikantharun.bazel-ide-toolkit)** - Auto-refresh compile_commands.json

```bash
code --install-extension llvm-vs-code-extensions.vscode-clangd
code --install-extension srikantharun.bazel-ide-toolkit
```

### Step 4: Initial Generation

Generate `compile_commands.json` for the first time:

```bash
bazel run //:refresh_compile_commands
```

This may take a few minutes for large projects as it needs to analyze the build graph.

## Using the Extension

Once installed, the extension activates automatically when you open a Bazel workspace (detected by presence of WORKSPACE, WORKSPACE.bazel, or MODULE.bazel).

### Status Bar

Look for **"Bazel"** in the bottom status bar:
- `$(sync) Bazel` - Idle, click to manually refresh
- `$(sync~spin) Bazel` - Refresh in progress
- `$(check) Bazel` - Last refresh successful
- `$(error) Bazel` - Last refresh failed

### Commands

Press `Cmd+Shift+P` (or `Ctrl+Shift+P` on Linux/Windows) and type "Bazel:" to see available commands:

| Command | Description |
|---------|-------------|
| `Bazel: Refresh compile_commands.json` | Manually trigger refresh |
| `Bazel: Toggle Auto-Refresh` | Enable/disable automatic refresh |
| `Bazel: Select Platform` | Choose target platform for cross-compilation |

### Settings

Configure the extension in VS Code settings:

```json
{
  "bazelIdeToolkit.autoRefresh": true,
  "bazelIdeToolkit.debounceMs": 2000,
  "bazelIdeToolkit.targets": "//...",
  "bazelIdeToolkit.showStatusBar": true
}
```

## How It Works

```
BUILD file changed
       ↓
Extension detects change (file watcher)
       ↓
Wait for debounce period (default 2s)
       ↓
Run: bazel run @hedron_compile_commands//:refresh_all
       ↓
compile_commands.json updated
       ↓
clangd reloads automatically
       ↓
IntelliSense updated!
```

## Tips for Large Monorepos

### Use Specific Targets

Instead of `//...`, specify the packages you're actively working on:

```starlark
refresh_compile_commands(
    name = "refresh_compile_commands",
    targets = {
        "//myproject/src/...": "",
        "//myproject/lib/...": "",
    },
)
```

### Exclude External Sources

If you don't need IntelliSense for external dependencies:

```starlark
refresh_compile_commands(
    name = "refresh_compile_commands",
    exclude_external_sources = True,
    targets = {"//...": ""},
)
```

### Disable Remote Cache for Local Development

If your project uses a remote cache you don't have access to:

```bash
bazel run //:refresh_compile_commands --remote_cache=
```

### Increase Debounce Time

For projects where BUILD files change frequently:

```json
{
  "bazelIdeToolkit.debounceMs": 5000
}
```

## Troubleshooting

### "No compile_commands generator found"

Make sure you've added both:
1. `hedron_compile_commands` to MODULE.bazel
2. `refresh_compile_commands` target to BUILD.bazel

### Refresh fails with package errors

Some packages in your workspace may have errors. Specify working packages explicitly in the `targets` attribute instead of using `//...`.

### clangd not picking up changes

Try reloading the VS Code window (`Cmd+Shift+P` → "Reload Window") or restart clangd (`Cmd+Shift+P` → "clangd: Restart language server").

## Links

- **VS Code Extension**: [marketplace.visualstudio.com/items?itemName=srikantharun.bazel-ide-toolkit](https://marketplace.visualstudio.com/items?itemName=srikantharun.bazel-ide-toolkit)
- **GitHub Repository**: [github.com/srikantharun/bazel-ide-toolkit](https://github.com/srikantharun/bazel-ide-toolkit)
- **hedron_compile_commands**: [github.com/hedronvision/bazel-compile-commands-extractor](https://github.com/hedronvision/bazel-compile-commands-extractor)

## Contributing

Found a bug or have a feature request? Open an issue on [GitHub](https://github.com/srikantharun/bazel-ide-toolkit/issues).

---

*This extension was created to solve a real pain point I experienced working on large Bazel C++ monorepos. I hope it helps improve your development experience too!*
