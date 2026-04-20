# SOP: Extract a directory into an independent submodule

How to pull a vendored directory out of an existing repo, publish it as a
standalone GitHub repo, and wire it back in as a git submodule.

## Steps

### 1. Create the GitHub repo

```sh
gh repo create V-Sekai-fire/multiplayer-fabric-<name> \
  --public \
  --description "<one-line description>"
```

### 2. Copy and push the content

If the directory has no embedded `.git`:

```sh
cp -r <source-path> /tmp/multiplayer-fabric-<name>
cd /tmp/multiplayer-fabric-<name>
git init && git checkout -b main
git add -A
git commit -m "Extract <name> as independent repo from <source-repo>"
git remote add origin https://github.com/V-Sekai-fire/multiplayer-fabric-<name>.git
git push -u origin main
```

### 3. Replace the vendored copy with a submodule

Create a new versioned branch on the source godot branch
(see `branch_versioning.md`), then:

```sh
git checkout -b feat/<module>-NNN origin/feat/<module>-00(N-1)
git rm -r <path/to/directory>
rm -rf <path/to/directory>   # remove any leftover untracked files
git submodule add https://github.com/V-Sekai-fire/multiplayer-fabric-<name>.git <path/to/directory>
git add .gitmodules <path/to/directory>
git commit -m "Replace vendored <name> with submodule multiplayer-fabric-<name>"
git push origin feat/<module>-NNN
```

### 4. Update gitassembly

In `multiplayer-fabric-merge` on `main`:

```sh
# Edit gitassembly: update the branch reference to the new NNN version
git add gitassembly
git commit -m "Use feat/<module>-NNN with <name> as submodule"
git push
```

### 5. Register in the root repo

```sh
cd /path/to/multiplayer-fabric
git submodule add https://github.com/V-Sekai-fire/multiplayer-fabric-<name>.git multiplayer-fabric-<name>
git add .gitmodules multiplayer-fabric-<name> README.md
git commit -m "Add multiplayer-fabric-<name> submodule"
git push
```

### 6. Update README.md submodule map

Add a row to the table in `README.md`:

```
| `multiplayer-fabric-<name>` | <Language> | <Role> |
```

### 7. Run sync check

See `sync_check.md`.

## Pitfalls

- **Empty GitHub repo**: `git submodule add` on an empty repo fails. Push at
  least one commit before adding the submodule.
- **Leftover directory**: After `git rm -r`, the directory may still exist on
  disk. Delete it with `rm -rf` before calling `git submodule add`.
- **Tracking wrong upstream**: After pushing a new branch, set tracking
  explicitly with `git branch --set-upstream-to=origin/<branch> <branch>`.
