---
name: ci-check
description: Verify gitassembly branch references are consistent and no branch in multiplayer-fabric-godot has failing GitHub Actions runs. Use after any push to a feature branch or after editing gitassembly. When GHA is clogged, run a local mingw build instead.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: CI check — gitassembly sync and GitHub Actions

How to verify that `gitassembly` is consistent and that no branch in
`multiplayer-fabric-godot` has failing GitHub Actions runs.

## 1. Review gitassembly

```sh
cat multiplayer-fabric-merge/gitassembly
```

Verify:
- Every `merge` line references a branch that actually exists on the remote.
- Branch names ending in `-NNN` match the latest version number.
- Comments accurately describe what the branch contributes.

```sh
# Check that all referenced branches exist on origin
git -C multiplayer-fabric-godot fetch origin
git -C multiplayer-fabric-godot branch -r | grep -E "feat/|docs/" | sort
```

## 2. Check GitHub Actions

Only check branches that exist in `https://github.com/V-Sekai-fire/multiplayer-fabric-godot`:

```sh
# Get latest run per branch, show only failures on gitassembly branches
gh run list --repo V-Sekai-fire/multiplayer-fabric-godot --limit 40 \
  --json status,conclusion,headBranch,databaseId,createdAt \
  | jq 'group_by(.headBranch) | .[]
        | {branch: .[0].headBranch,
           latest: (sort_by(.createdAt) | last
                    | {id: .databaseId, conclusion: .conclusion, status: .status})}
        | select(.latest.conclusion == "failure")'
```

Cross-reference results against gitassembly: ignore failures on `archived/*`,
superseded branch versions, and branches not listed in gitassembly.

For each relevant failure, retrieve the log — always pass `--repo`:

```sh
gh run view <run-id> --repo V-Sekai-fire/multiplayer-fabric-godot \
  --log-failed 2>&1 | grep -E "(##\[group\]|##\[error\])" | head -40
```

## 3. Common failure: static_checks spelling/formatting diff

The `static_checks.yml` workflow runs a formatting check over changed files.
If it exits with code 1 and shows a diff, the diff is what the formatter
wants to change. Apply the change locally, commit to the failing branch,
and push.

Example: `capitalised` → `capitalized` in a Markdown file flagged by the
spell/format step.

Use the existing `multiplayer-fabric-godot` checkout as staging — do not
clone a second copy. Create a versioned branch (see `branch-versioning`),
fix the file, run pre-commit, then push:

```sh
git -C multiplayer-fabric-godot checkout -b feat/<module>-NNN origin/feat/<module>-00(N-1)
# fix the file
pre-commit run --files <file>          # MUST pass before committing
git -C multiplayer-fabric-godot add <file>
git -C multiplayer-fabric-godot commit -m "Fix formatting per static checks"
git -C multiplayer-fabric-godot push origin feat/<module>-NNN
# update gitassembly in multiplayer-fabric-merge
# return to previous branch
git -C multiplayer-fabric-godot checkout multiplayer-fabric
```

## 4. Verify fix

Re-run the check (GitHub triggers automatically on push). Poll until clear:

```sh
gh run list --repo V-Sekai-fire/multiplayer-fabric-godot --limit 10 \
  --json status,conclusion,headBranch | \
  jq '.[] | select(.conclusion == "failure")'
```

Empty output = no failing runs.

## 5. Return to previous branch

Always return the godot working tree to its expected state after a fix:

```sh
git -C multiplayer-fabric-godot checkout multiplayer-fabric
```

## Fallback: local build when GitHub Actions is clogged

If GHA queues are backed up and you need a faster answer, build locally
using the `gscons` alias. Always pass `ccache=sccache` — sccache is the
compiler cache for this project and MUST be used whenever building locally.

```sh
cd multiplayer-fabric-godot

# gscons expands to:
# scons tests=yes dev_build=yes compiledb=yes accesskit=no cache_path="$HOME/.cache/scons-godot"

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

Verify sccache is installed: `which sccache`. Install if missing:

```sh
# macOS — Homebrew is required; install it first if absent:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install sccache

# Linux:
cargo install sccache
```

Check hit rate after a build with `sccache --show-stats`.

Run on the failing branch's checkout to reproduce compiler errors locally.
After a clean local build, push the fix and let GHA confirm on the next run.
