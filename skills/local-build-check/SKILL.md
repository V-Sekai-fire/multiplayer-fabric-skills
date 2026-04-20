---
name: local-build-check
description: Build multiplayer-fabric-godot locally to reproduce CI failures when GitHub Actions is clogged. Use when GHA queues are backed up and you need a faster answer on a failing branch.
license: MIT
status: tombstoned
metadata:
  author: V-Sekai-fire
  tombstoned_date: 2026-04-19
  tombstone_reason: Superseded; content migrated into ci-check and assembly workflow.
---

> **TOMBSTONED** — This skill is no longer maintained. Do not invoke it.
> Content has been migrated. See `ci-check` for the current workflow.

# SOP: Local build check (GHA fallback)

How to build `multiplayer-fabric-godot` locally to reproduce compiler errors
when GitHub Actions queues are too slow.

## Prerequisites

sccache MUST be used for all local builds. Install if missing:

```sh
# macOS — Homebrew is required; install it first if absent:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install sccache

# Linux:
cargo install sccache
```

Verify: `which sccache`

## Build commands

The `gscons` alias expands to:
`scons tests=yes dev_build=yes compiledb=yes accesskit=no cache_path="$HOME/.cache/scons-godot"`

Always add `ccache=sccache` and choose a toolchain:

```sh
cd multiplayer-fabric-godot

# MinGW (cross-compile for Windows, Linux/macOS host):
scons tests=yes dev_build=yes compiledb=yes accesskit=no \
  cache_path="$HOME/.cache/scons-godot" \
  platform=windows use_mingw=yes \
  ccache=sccache

# Clang (native, any platform):
scons tests=yes dev_build=yes compiledb=yes accesskit=no \
  cache_path="$HOME/.cache/scons-godot" \
  use_llvm=yes \
  ccache=sccache
```

Use the existing `multiplayer-fabric-godot` checkout as staging — do not
clone a second copy. Check out the failing branch first:

```sh
git -C multiplayer-fabric-godot checkout -b feat/<module>-NNN origin/feat/<module>-00(N-1)
# build, fix errors, run pre-commit
pre-commit run --files <changed-files>
git -C multiplayer-fabric-godot add <files>
git -C multiplayer-fabric-godot commit -m "Fix build error"
git -C multiplayer-fabric-godot push origin feat/<module>-NNN
git -C multiplayer-fabric-godot checkout multiplayer-fabric
```

## Checking all gitassembly branches independently

The gitassembly file uses `remotes/v-sekai-fire/<branch>` notation, but the
configured remote in the local checkout is **`origin`**. Always translate:

```
remotes/v-sekai-fire/feat/module-foo  →  origin/feat/module-foo
remotes/v-sekai-fire/master           →  origin/master
```

To build every branch listed in `multiplayer-fabric-merge/gitassembly` one by one:

```sh
GODOT=multiplayer-fabric-godot
SCONS_OPTS="tests=yes dev_build=yes compiledb=yes accesskit=no \
  cache_path=$HOME/.cache/scons-godot use_llvm=yes ccache=sccache"

for branch in \
  origin/master \
  origin/feat/module-keychain \
  origin/feat/module-lasso \
  origin/feat/engine-patches-002 \
  origin/feat/engine-patch-control-gui-input \
  origin/feat/module-http3-001 \
  origin/feat/module-multiplayer-fabric-001 \
  origin/feat/module-multiplayer-fabric-mmog-002 \
  origin/feat/module-open-telemetry \
  origin/feat/module-sandbox \
  origin/feat/module-speech \
  origin/feat/module-sqlite; do
  echo "=== $branch ==="
  git -C "$GODOT" checkout --detach "$branch"
  (cd "$GODOT" && scons $SCONS_OPTS) && echo "PASS: $branch" || echo "FAIL: $branch"
done

git -C "$GODOT" checkout multiplayer-fabric
```

With a warm SCons cache each branch takes ~12 s. sccache shows 0 requests when
SCons cache serves all objects (expected behavior — not a misconfiguration).

## After a successful local build

Check sccache hit rate: `sccache --show-stats`

Push the fix and let GHA confirm on the next run (see `ci-check`).

## Reference

This skill follows the Agent Skills format — see [references/references.bib](references/references.bib) (`agentskills2025specification`).
