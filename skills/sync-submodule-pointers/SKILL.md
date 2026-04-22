---
name: sync-submodule-pointers
description: Advance the multiplayer-fabric monorepo's submodule pointers to match the current HEAD of each submodule and push. Use after committing to one or more submodules to keep the root pointer in sync.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Sync submodule pointers

Bring the multiplayer-fabric root repo's recorded submodule commits up to date
with each submodule's current HEAD, then push.

## 1. Find stale pointers

From the monorepo root:

```sh
git status --short
```

Any line beginning with ` M <submodule>` or ` m <submodule>` is a submodule
whose pointer in the root repo differs from what is checked out locally.

- ` M` — the submodule HEAD has moved (new commits)
- ` m` — the submodule has uncommitted working-tree changes; commit those first

## 2. Commit submodule changes (if ` m`)

For any submodule with working-tree changes, commit and push those first:

```sh
cd <submodule>
git add <files>
git commit -m "<message>"
git push
cd ..
```

## 3. Advance the pointer

After all submodule commits are pushed, stage and commit the pointer updates in
the root repo:

```sh
# Stage every stale pointer at once
git add <submodule-a> <submodule-b> ...

# Write a commit message that names what changed in each
git commit -m "Sync submodules: <brief summary per submodule>"
git push
```

### Naming the commit

Use one of these patterns:

```
Sync submodules: deploy Burrito wiring, zone-backend format fix
Sync submodules: update skills pointer
Sync submodules: advance all to main HEAD
```

Never use `Update submodule pointer` alone — name what changed.

## 4. Verify

```sh
git status --short
```

No ` M` or ` m` lines remain.

```sh
git log --oneline @{u}..HEAD
```

One commit ahead of origin (the pointer update). Push if not already pushed.

## Full one-liner (when all submodules are already pushed)

```sh
git submodule update --remote <submodule-a> <submodule-b> && \
git add <submodule-a> <submodule-b> && \
git commit -m "Sync submodules: <summary>" && \
git push
```

## When to use `git submodule update --remote` vs `git add`

| Situation | Command |
|---|---|
| You pushed new commits to a submodule's remote and want the root to track them | `git submodule update --remote <path>` then `git add` |
| You already have the right commit checked out locally in the submodule | `git add <path>` directly |
| You want to pull *all* submodules to their remote HEAD | `git submodule update --remote` (no path — updates all) |

## Pass condition

`git status --short` shows no ` M` or ` m` lines and
`git log --oneline @{u}..HEAD` is empty after push.
