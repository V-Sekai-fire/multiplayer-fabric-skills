---
name: ci-check
description: Verify gitassembly branch references are consistent and no branch in multiplayer-fabric-godot has failing GitHub Actions runs. Use after any push to a feature branch or after editing gitassembly.
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
wants to change.

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

If GHA queues are backed up, use the `local-build-check` skill instead.

## Reference

This skill follows the Agent Skills format — see [references/references.bib](references/references.bib) (`agentskills2025specification`).
