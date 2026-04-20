---
name: refetch-godot-branch
description: Fetch origin and hard-reset multiplayer-fabric-godot to origin/multiplayer-fabric. Use when the local multiplayer-fabric branch needs to be refreshed after the git assembler runs, after a force-push to origin/multiplayer-fabric, or whenever the working tree must exactly match the assembled upstream.
---

# Refetch multiplayer-fabric-godot to origin

```sh
cd /Users/ernest.lee/Desktop/multiplayer-fabric/multiplayer-fabric-godot
git fetch origin
git checkout multiplayer-fabric
git reset --hard origin/multiplayer-fabric
```

Verify:

```sh
git log --oneline -3
git status
```

Expected: clean working tree, HEAD matches `origin/multiplayer-fabric`.
