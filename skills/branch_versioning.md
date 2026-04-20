# SOP: Branch versioning

When to create a versioned branch (`feat/foo-001`, `feat/foo-002`) instead
of force-pushing, and how to wire it up end-to-end.

## When to version

- The remote branch is protected and rejects force-push.
- A team admin has archived the branch name (e.g. moved `feat/module-http3` → `archived/module-http3`) asynchronously.
- The branch tip diverges from what the assembly needs and a clean history
  matters.

Never bypass branch protection with `--no-verify` or admin override.

## Steps

1. **Create the new branch** from the current remote tip or a known-good base:

   ```sh
   git checkout -b feat/foo-002 origin/feat/foo-001
   # make changes
   git commit -m "Describe the change"
   git push origin feat/foo-002
   ```

2. **Update gitassembly** in `multiplayer-fabric-merge` (on `main`):

   ```sh
   # Edit gitassembly: replace feat/foo-001 with feat/foo-002
   git add gitassembly
   git commit -m "Use feat/foo-002 — <reason>"
   git push
   ```

3. **Fix local upstream tracking**:

   ```sh
   git branch --set-upstream-to=origin/feat/foo-002 feat/foo-002
   ```

4. **Run sync check** (see `sync_check.md`).

## Naming convention

`feat/<module>-NNN` where NNN is a zero-padded three-digit integer starting
at `001`. The number is a snapshot counter, not a semantic version.

## Do not delete old branches

Old versioned branches are the audit trail. Leave them on origin; do not
archive or delete unless explicitly agreed.
