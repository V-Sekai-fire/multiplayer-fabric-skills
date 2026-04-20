# SOP: Sync check

Verify all submodules and the root repo are fully pushed to origin.

## Steps

```sh
# From the root multiplayer-fabric directory:
git submodule foreach 'git log --oneline @{u}..HEAD 2>/dev/null | head -3 || echo "(no upstream)"'
echo "---ROOT---"
git log --oneline @{u}..HEAD
```

Any submodule with output has unpushed commits. For each:

```sh
git -C <submodule-path> push
```

Then update the root submodule pointer and push:

```sh
git add <submodule-path>
git commit -m "Update <name> submodule pointer"
git push
```

## Tracking fix

If a branch shows `[origin/wrong-branch: ahead N]`, fix the upstream:

```sh
git -C <submodule-path> branch --set-upstream-to=origin/<correct-branch> <local-branch>
```

## Pass condition

Both commands produce no output.
