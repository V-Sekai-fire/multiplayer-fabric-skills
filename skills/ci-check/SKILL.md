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

```sh
gh run list --repo V-Sekai-fire/multiplayer-fabric-godot --limit 20 \
  --json status,conclusion,name,headBranch,createdAt
```

Look for any `"conclusion":"failure"` entries.

For each failure, retrieve the log:

```sh
gh run view <run-id> --log-failed 2>&1 | grep -E "(error|##\[error\]|diff)" | head -40
```

## 3. Common failure: static_checks spelling/formatting diff

The `static_checks.yml` workflow runs a formatting check over changed files.
If it exits with code 1 and shows a diff, the diff is what the formatter
wants to change. Apply the change locally, commit to the failing branch,
and push.

Example: `capitalised` → `capitalized` in a Markdown file flagged by the
spell/format step.

```sh
git -C multiplayer-fabric-godot checkout <failing-branch>
# fix the file
git -C multiplayer-fabric-godot add <file>
git -C multiplayer-fabric-godot commit -m "Fix formatting per static checks"
git -C multiplayer-fabric-godot push
# return to previous branch
git -C multiplayer-fabric-godot checkout -
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
