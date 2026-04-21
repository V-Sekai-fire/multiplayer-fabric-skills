---
name: platform-header-check
description: Verify a C/C++ header in multiplayer-fabric-godot parses cleanly on macOS (clang), Windows (mingw), and Web (emscripten) without a full Godot build. Use after adding or moving headers to catch include-path errors before pushing to CI.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Platform header check (macOS / Windows / Web)

Fast preprocessor-only check that a header resolves on all three platforms
using the local toolchains. No Godot build required — completes in seconds.

## Prerequisites

All three must be on PATH:

```sh
which clang++                     # macOS
which x86_64-w64-mingw32-g++     # Windows (brew install mingw-w64)
which em++                        # Web (brew install emscripten)
```

## Check a single header

Replace `<path/to/header.h>` with the path relative to the Godot root:

```sh
GODOT=/Users/ernest.lee/Desktop/multiplayer-fabric/multiplayer-fabric-godot
HEADER="thirdparty/misc/predictive_bvh.h"   # adjust as needed

for entry in "clang++ macOS" "x86_64-w64-mingw32-g++ Windows" "em++ Web"; do
  cc=$(echo $entry | cut -d' ' -f1)
  label=$(echo $entry | cut -d' ' -f2)
  $cc -std=c++17 -E -x c++ \
    -I"$GODOT" -I"$GODOT/thirdparty/misc" \
    - <<< "#include \"$HEADER\"" \
    > /dev/null 2>&1 && echo "$label: OK" || echo "$label: FAIL"
done
```

Expected output when the include path is correct:

```
macOS: OK
Windows: OK
Web: OK
```

## Check the fabric_zone.h consumer

To verify the actual include site in `fabric_zone.h`:

```sh
GODOT=/Users/ernest.lee/Desktop/multiplayer-fabric/multiplayer-fabric-godot

for entry in "clang++ macOS" "x86_64-w64-mingw32-g++ Windows" "em++ Web"; do
  cc=$(echo $entry | cut -d' ' -f1)
  label=$(echo $entry | cut -d' ' -f2)
  $cc -std=c++17 -E -x c++ \
    -I"$GODOT" -I"$GODOT/thirdparty/misc" \
    - <<< '#include "thirdparty/misc/predictive_bvh.h"' \
    > /dev/null 2>&1 && echo "$label: OK" || echo "$label: FAIL"
done
```

## What this catches

- Missing or mis-spelled `#include` path after a header is moved or vendored
- Platform-specific preprocessor errors (e.g. MSVC-only pragmas via mingw)
- Emscripten rejecting C++ syntax (use `em++`, not `emcc`, for C++ headers)

## What this does NOT catch

- Linker errors or missing `.cpp` implementations
- Full Godot module compile errors (use `local-build-check` / `ci-check` for those)

## After fixing an include path

Run this check, then push. CI will confirm on the full build.

```sh
git -C multiplayer-fabric-godot add <changed-files>
git -C multiplayer-fabric-godot commit -m "Fix include path for <header>"
git -C multiplayer-fabric-godot push
```

## Reference

This skill follows the Agent Skills format — see [references/references.bib](references/references.bib) (`agentskills2025specification`).
