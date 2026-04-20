# SOP: Dependency audit (known knowns / known unknowns / unknown unknowns)

How to review a plan or TODO for hidden risks before starting implementation.

## Three categories

| Category | Definition | Action |
|---|---|---|
| Known known | Verified to exist and work | Document, proceed |
| Known unknown | Identified gap or unverified assumption | File as a blocker, resolve before the cycle that needs it |
| Unknown unknown | Risk the plan doesn't mention | Discovered only by reading source and searching externally |

## Checklist

For each external dependency named in the plan:

- [ ] Does the GitHub repo exist and have real code (not a stub)?
- [ ] Is the public API the plan calls actually defined?
- [ ] Does the library pin to an OTP/Elixir/Rust version compatible with
      the rest of the stack?
- [ ] Are there version conflicts with other deps in the same `mix.exs`?

For each internal component the plan calls:

- [ ] Does the function/method exist at the named path?
- [ ] Does the HTTP endpoint exist in the router and controller?
- [ ] Is the wire protocol constant defined at the expected value?

For cross-language boundaries (Elixir ↔ C++, Rust ↔ Lean):

- [ ] Do both sides agree on the hash algorithm (not just the name)?
- [ ] Are byte-order and struct layout explicitly documented?
- [ ] Is there a round-trip test that crosses the boundary?

For infrastructure:

- [ ] Is the Docker image published and non-empty?
- [ ] Are environment variables documented in `.env.sample`?
- [ ] Is there a local-dev setup path that doesn't require production keys?

## Process

1. Read the plan thoroughly.
2. For each named thing, grep the codebase first; web-search second.
3. Record findings in three sections in the plan document or a separate
   audit file.
4. Convert each known unknown into a concrete task added before the cycle
   that depends on it.
5. Re-run the audit after any significant plan change.

## Example audit command

```sh
# Check if a function exists across Elixir source
grep -r "def upload_asset" multiplayer-fabric-zone-console/

# Check if a C++ symbol exists
grep -r "fetch_asset" multiplayer-fabric-godot/modules/

# Check if an HTTP route exists
grep -r "manifest" multiplayer-fabric-zone-backend/lib/uro_web/router.ex
```
