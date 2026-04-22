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
sibling modules' `mix.exs`, `project.godot`, `*.yml`, and `*.toml`. A submodule
with zero cross-refs looks removable but may be a load-bearing library if its
consumers are in another layer (e.g. `aria-storage` appears unreferenced until
you check zone-backend and zone-console).

Note findings that indicate mass with no load:

| Signal | Weight boost |
|---|---|
| All-zero hashes (never shipped) | +0.20 |
| Only boilerplate commits (initial, CONTRIBUTING, sentence-case) | +0.15 |
| CMake / build paths point outside the repo | +0.10 |
| Self-described "work in progress" or "not finished" | +0.08 |
| Duplicate of another mechanism already in the repo | +0.12 |
| Couples to an external third-party API or platform with no infra consumer | +0.15 |
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

Change the status prefix in the title line only. Use:
- `Rejected` — for `Draft` or `Proposed` ADRs that were never adopted
- `Superseded` — for `Accepted` ADRs replaced by a different decision

```sh
# Edit the title line: "# Draft: …" → "# Rejected: …"
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
