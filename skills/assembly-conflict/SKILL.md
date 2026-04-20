---
name: assembly-conflict
description: Fix merge conflicts that stop the multiplayer-fabric-merge assembler. Use when `elixir update_godot_v_sekai.exs` halts with a conflict error and reports a branch that needs manual resolution.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Assembly conflict resolution

How to fix a merge conflict that stops `elixir update_godot_v_sekai.exs`
from completing.

## Diagnosis

The assembler stops with:

```
git-assembler: error while merging remotes/v-sekai-fire/<branch> into multiplayer-fabric
git-assembler: fix/commit then re-run git-assembler
```

Check the conflict type from the assembler output:

| Conflict type | Cause | Fix |
|---|---|---|
| `add/add` in a source file | Two branches both added the same file | Move file to whichever branch merges first; remove from later branch |
| `content` in `.gitignore` | Multiple branches appended to the same base | Use `union` merge driver |
| `content` in any other file | Genuine divergence | Resolve manually, commit, re-run |

## Fix: union merge driver for .gitignore

Add to `.git/info/attributes` of the `multiplayer-fabric-merge` working tree
(this file is local, never committed):

```
.gitignore merge=union
```

The `union` driver is built into git — no extra config needed.

## Fix: add/add file conflict

1. Identify which earlier branch introduced the file:
   ```sh
   git -C multiplayer-fabric-merge show remotes/v-sekai-fire/<branch>:<file> | wc -l
   ```
2. Remove the file from the later branch by creating a new versioned branch
   (see `branch-versioning`).
3. Update `gitassembly` to reference the new branch.

## Fix: content conflict (manual resolve)

The assembler leaves the repo mid-merge. Resolve in-place:

```sh
cd multiplayer-fabric-merge
# edit the conflicting file, remove <<<<< ===== >>>>> markers
git add <file>
git commit --no-edit
# re-run the assembler
elixir update_godot_v_sekai.exs
```

## Reset to main (abort)

If you need to start over:

```sh
git -C multiplayer-fabric-merge merge --abort
git -C multiplayer-fabric-merge checkout main
```
