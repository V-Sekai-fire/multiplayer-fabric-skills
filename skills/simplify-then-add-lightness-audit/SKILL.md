---
name: simplify-then-add-lightness-audit
description: Run a verbalized-sampling audit of the multiplayer-fabric monorepo against the Simplify-Then-Add-Lightness ADR. Surfaces one removal candidate per run with a weighted distribution across all submodules and uncommitted dead weight. Use when you want to reduce complexity in a principled, randomised way rather than acting on intuition alone.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Simplify-Then-Add-Lightness audit

One-command codebase audit using verbalized sampling \[zhang2025verbalized\]
against the Colin Chapman principle encoded in the project ADR
\[vsekaisimplify2026\].

## 0. Read the ADR (once per session)

Fetch the current ADR so the audit stays anchored to it:

```
/fetch https://v-sekai.github.io/manuals/decisions/20260422-simplify-then-add-lightness.html
```

Key tests from the ADR to apply to every candidate:

- Would a new engineer need to understand this to ship?
- Does it add a release obligation (version string, hash, formula) with no
  production payoff?
- Is there a simpler thing already present that does the same job?
- Does it couple two systems that should be independent?

### Strategic Delegation — mandatory pre-filter

Before scoring any candidate, classify it:

- **Core uncertainty** (defines whether the project works at all — character
  motion, UX interaction, spatial partitioning, transport protocol): these stay
  in-house regardless of commit count or consumer count. Do not nominate them.
- **Mature, well-understood component** (databases, S3 storage, OS libraries,
  external game APIs): these are delegation candidates.
- **License-constrained fork** (e.g. BSL-licensed upstream replaced by own
  build): the fork is load-bearing by compliance necessity. Do not nominate it
  for removal without a licence-compatible replacement already in hand.

Known permanent exclusions for this repo:

| Submodule | Reason to keep |
|---|---|
| `cockroach` | BSL licence on upstream — own build is the compliant alternative |
| `multiplayer-fabric-humanoid-project` | In-house character motion research (core uncertainty) |
| `multiplayer-fabric-interaction-system` | In-house UX/XR interaction research (core uncertainty) |
| `multiplayer-fabric-artifacts-mmog` | Taskweft test harness on a stable external platform — validates the planner before the home platform is ready (Risk Sequencing) |
| `multiplayer-fabric-rx` | Integration environment validating the full Godot stack (VRM, XR, game framework, humanoid) works together — core uncertainty for stack cohesion |
| `multiplayer-fabric-baker` | Godot asset baker — should feed into desync for storage; missing integration is a bug, not dead weight |
| `multiplayer-fabric-desync` | Content-addressed storage (casync) — intended storage backend for baker output; keep until baker→desync integration is complete |
| `multiplayer-fabric-sandbox` | Proof-of-concept UGC programming language (RISC-V sandbox) — core uncertainty for user-generated content runtime |
| `multiplayer-fabric-webtransport` | Vendored fork — upstream requires patches that cannot be upstreamed; own build is the only viable option |
| `multiplayer-fabric-skills` | Project SOP library — Claude Code skills evolve with the codebase; removing it severs the process documentation from the code it governs |
| `multiplayer-fabric-predictive-bvh` | Codegen source for `multiplayer-fabric-godot/thirdparty/misc/predictive_bvh.h` — the fabric cannot run without this generated header |
| `ZoneConsole.AccessKit` (`zone-console/lib/zone_console/access_kit.ex`) | Planned integration point for native accessibility tree (NSAccessibility / AT-SPI2 / UI Automation) — needs plumbing into Godot, zones, and backends; stub + test are forward scaffolding, not dead weight |
| `Uro.IdentityProofController` + `Uro.UserRelations` identity_proof functions | Planned web-of-trust feature — to be wired via ReBAC (Taskweft.ReBAC) with HTTP routes; unrouted now but not dead weight |
| `Uro.Uploaders.UserIcon` (`zone-backend/lib/uro/uploaders/user_icon.ex`) | Planned Waffle→S3→desync upload integration — needs wiring to S3 bucket and desync chunk store; zero callers now but load-bearing once the upload pipeline is wired |
| `Uro.EnsureUserNotLockedPlug` (`zone-backend/lib/uro/plug/ensure_user_not_locked_plug.ex`) | Planned account-lock security check — to be wired into router pipelines once account locking is implemented; not dead weight |
| `ZoneConsole.FabricMMOGKeyStore` (`zone-console/lib/zone_console/fabric_mmog_keystore.ex`) | Planned AES-128 asset key store for the baker encryption pipeline (`encrypt_pck` currently disabled); uses OS keychain (correct — zone-console has no DB); Elixir port of `modules/keychain/fabric_mmog_keystore.cpp`; not dead weight |
| `Taskweft.HRR.*` (`multiplayer-fabric-taskweft/lib/taskweft/hrr/`) | Episodic memory layer for the RECTGTN HTN planner: past plan episodes stored as HRR vectors in SQLite; retriever scores candidates by cosine similarity to current entity state, enriching `EntityPlanner.plan/2` with relevant context. Integration path: probe HRR memory → inject ranked episodes into domain state → plan → store result. Do not remove even with zero callers. |

## 0.5. Stall check — when simplification alone is not unblocking progress

If the user reports the project is still not moving after repeated removal runs,
shift the distribution framing from **dead-weight removal** to
**feedback loop compression** (ADR principle: Feedback Loop Compression +
Sequence Risks).

Run these additional checks before generating the distribution:

```sh
# Commit velocity per submodule — last commit date
for mod in <submodules>; do
  git -C $mod log -1 --format="%ar %s"
done

# Are critical-path services gated behind optional profiles?
grep -n "profiles:" <docker-compose files>

# Are e2e / smoke tests wired to CI, or manual-only?
find . -maxdepth 4 -name "*.spec.ts" -o -name "*e2e*" | grep -v ".git"
grep -r "playwright\|cypress\|smoke" .github/workflows/ 2>/dev/null

# Are core integrations wired? (e.g. baker→desync, zone-server→CI)
# For each pair of related submodules, grep each for the other's name
```

The slowest feedback loop is the gap between "engineer makes a change" and
"the critical assumption (e.g. players can connect) is verified." Identify it
from git history: look for active repos with zero cross-references, optional
service profiles, and e2e tests that only run manually.

Reframe the distribution task when stalled:

```
name ONE barrier to remove that would compress the feedback loop between
a code change and verifying the critical assumption. "Remove" includes:
removing an optional-profile gate, removing dead config that hides service
health, or removing the isolation between two systems that must be wired.
Flag candidates that are actually ADDs disguised as removals — those require
deliberate addition, not removal.
```

Return to standard removal framing once the critical feedback loop is closed.

## 1. Inventory the codebase

Run these before generating the distribution so weights are grounded in facts,
not assumptions:

```sh
# Active submodules
git submodule status

# Dead files (bak, build artefacts, committed test output)
find . -maxdepth 4 \( -name "*.bak" -o -name "test-results" -o -name "docs.bak" \) \
  -not -path "./.git/*" -not -path "./*/\.git/*"

# Placeholder / never-shipped release manifests
grep -r '"0000000000000000000000000000000000000000000000000000000000000000"' \
  --include="*.rb" --include="*.json" -l 2>/dev/null

# Submodules with no consumers (nothing imports them)
# For each addon submodule, grep the rest of the tree for its directory name
```

Before scoring, verify actual consumers — grep for the submodule name across
sibling modules' `mix.exs`, `project.godot`, `*.yml`, `*.toml`, `*.h`, `*.c`,
and `*.cpp`. A submodule with zero cross-refs looks removable but may be a
load-bearing library if its consumers are in another layer (e.g. `aria-storage`
appears unreferenced until you check zone-backend and zone-console;
`multiplayer-fabric-predictive-bvh` generates a C header consumed by
`multiplayer-fabric-godot/thirdparty/misc/predictive_bvh.h`).

Note findings that indicate mass with no load:

| Signal | Weight boost |
|---|---|
| All-zero hashes (never shipped) | +0.20 |
| Only boilerplate commits (initial, CONTRIBUTING, sentence-case) | +0.15 |
| CMake / build paths point outside the repo | +0.10 |
| Self-described "work in progress" or "not finished" | +0.08 |
| Duplicate of another mechanism already in the repo | +0.12 |
| Couples to an external third-party API or platform with no infra consumer | +0.15 |
| Active repo with zero cross-references to its intended consumer | +0.18 |
| Service gated behind optional profile — never exercised by default | +0.18 |
| E2e / smoke test exists but is manual-only, not wired to CI | +0.15 |
| Legal / licence compliance risk | immediate escalation |

## 2. Generate the verbalized distribution

Apply the verbalized-sampling template \[zhang2025verbalized\] with N=5:

```
Generate 5 distinct responses to the task below and assign each a
probability (0 < p_i ≤ 1, ∑p_i = 1.0) reflecting how likely that response
is the best removal candidate. Return JSON:

{"samples": [{"response": "<name + 2-sentence justification>", "weight": <float>}, ...]}

Task: Given the multiplayer-fabric monorepo inventory above and the
Simplify-Then-Add-Lightness ADR, name ONE thing to remove and justify it
in two sentences aligned with the ADR principles.

Previously removed this session (exclude): <list>
```

Normalise weights in case they do not sum to exactly 1.0, then draw:

```python
import random
total  = sum(s["weight"] for s in items)
probs  = [s["weight"] / total for s in items]
chosen = random.choices(items, weights=probs, k=1)[0]
```

## 3. Present the distribution and sampled result

Show the full weighted table to the user so they can override the draw.
Format:

```
| # | Candidate | Weight |
|---|-----------|--------|
| 1 | …         | 0.XX   |
…

Sampled result: **Remove X** — <justification>.
```

Ask for confirmation before acting.

## 4. Execute the removal

### Submodule removal

```sh
# 1. Archive the upstream GitHub repo (prevents dangling refs)
gh repo archive <org>/<repo> --yes

# 2. Deinit and remove from the monorepo
git submodule deinit <path>
git rm <path>

# 3. Commit and push
git commit -m "Remove <name> — <one-line reason>"
git push
```

### Dead-file removal

```sh
git rm -r <path>
git commit -m "Remove <path> — <one-line reason>"
git push
```

### ADR closure (never delete ADRs)

Change the status prefix in the title line only. Only close an ADR when a
**concrete replacement already exists** — do not close drafts merely because
they are old or were never adopted.

Use:
- `Superseded` — when a specific newer decision replaces this one (name it in the commit message)

Do NOT use `Rejected`. Draft ADRs that were never adopted stay as `Draft`
until a superseding decision exists.

```sh
# Edit the title line: "# Draft: …" → "# Superseded: …"
# Edit the title line: "# Accepted: …" → "# Superseded: …"
git -C manuals add decisions/<file>.md
git -C manuals commit -m "Close <file> — <one-line reason>"
git -C manuals push
git add manuals
git commit -m "Sync submodules: close ADR <name>"
git push
```

### Banned image / dependency replacement

Edit the offending file, replace the reference, verify it starts, then:

```sh
git add <file>
git commit -m "Replace <banned-thing> with <legal-alternative>"
git push
```

After any submodule change, update the parent repo pointer:

```sh
cd <monorepo-root>
git add <submodule-path>
git commit -m "Sync submodules: <summary>"
git push
```

## 5. Loop

Re-run from step 1 after each removal. The distribution shifts as mass
is removed; earlier high-weight candidates may drop once their peer
redundancies are gone.

Track what has been removed in the session to exclude from future draws.

## Session log format

Keep a running list in the conversation:

```
Removed this session:
- Nix (flake.nix + nix/) from zone-backend — duplicate of Docker/Mix build
- multiplayer-fabric-fly — third deployment target, Kubernetes already covers it
- docs.bak/ from multiplayer-fabric-deploy — stale design notes
- scoop-multiplayer-fabric — zero-hash manifests, never shipped
- homebrew-multiplayer-fabric — all five formulas had zero hashes
```

## Reference

This skill follows the Agent Skills format — see
[references/references.bib](references/references.bib)
(`agentskills2025specification`).

Verbalized sampling technique from \[zhang2025verbalized\] — see
[references/references.bib](references/references.bib).

ADR source: \[vsekaisimplify2026\] — see
[references/references.bib](references/references.bib).
