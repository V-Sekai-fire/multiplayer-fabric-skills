---
name: verbalized-sampling
description: Increase LLM output diversity without retraining by asking the model to generate N responses with probabilities, then sampling from that distribution. Use when a single prompt yields repetitive or mode-collapsed outputs (creative generation, synthetic data, dialogue variety).
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Verbalized sampling — diversity without retraining

How to apply verbalized sampling \[zhang2025verbalized\] to any Claude prompt
so that outputs span the full distribution rather than collapsing to the most
probable response.

## Background

After alignment training, LLMs exhibit **mode collapse**: repeated calls to
the same prompt cluster around a narrow peak of high-probability outputs.
Verbalized sampling sidesteps this at inference time by asking the model to
explicitly enumerate `N` options and assign each a probability.  The caller
then draws one option from that distribution with a weighted random draw —
no fine-tuning required, no temperature hacks, no model changes.

Papers report **1.6–2.1× diversity gain** (distinct-n-gram ratio) over
standard prompting across creative tasks.

## 1. Rewrite the prompt

Prepend or append the following instruction to any existing prompt:

```
Generate {N} distinct responses to the task below and assign each a
probability (0 < p_i ≤ 1, ∑p_i = 1.0) reflecting how likely that response
is relative to the others.  Return JSON with this schema:

{"samples": [{"response": "<text>", "weight": <float>}, ...]}

Task: {original_prompt}
```

Replace `{N}` with the desired candidate count (5 is a good default).
Replace `{original_prompt}` with the original task text verbatim.

## 2. Call the model

Use `claude-opus-4-7` with `output_config` forcing JSON schema compliance so
the weighted list is machine-readable without post-hoc parsing:

```sh
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 4096,
    "output_config": {
      "format": {
        "type": "json_schema",
        "schema": {
          "type": "object",
          "properties": {
            "samples": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "response": {"type": "string"},
                  "weight":   {"type": "number"}
                },
                "required": ["response", "weight"],
                "additionalProperties": false
              }
            }
          },
          "required": ["samples"],
          "additionalProperties": false
        }
      }
    },
    "messages": [{"role": "user", "content": "<verbalized prompt from step 1>"}]
  }'
```

Cache the system prompt if it is large and stable — add
`"cache_control": {"type": "ephemeral"}` to the last system text block.

## 3. Sample from the distribution

Parse `samples`, normalise weights in case they do not sum exactly to 1.0,
then draw one entry:

```python
import json, random

data   = json.loads(api_response_text)   # text block from response
items  = data["samples"]
total  = sum(s["weight"] for s in items)
probs  = [s["weight"] / total for s in items]
chosen = random.choices(items, weights=probs, k=1)[0]["response"]
print(chosen)
```

For batch synthetic-data generation, repeat step 2–3 `M` times; each call
independently samples a different region of the distribution.

## 4. Measure diversity (optional gate)

Compare verbalized-sampling outputs against standard outputs using
distinct-n-gram ratio.  Expect ≥1.5× improvement on creative tasks:

```python
from collections import Counter

def distinct_ngrams(texts, n=2):
    all_ng, uniq_ng = 0, set()
    for t in texts:
        toks = t.split()
        ngs  = [tuple(toks[i:i+n]) for i in range(len(toks)-n+1)]
        all_ng  += len(ngs)
        uniq_ng |= set(ngs)
    return len(uniq_ng) / max(all_ng, 1)

baseline  = distinct_ngrams(standard_outputs)
verbalized = distinct_ngrams(verbalized_outputs)
print(f"diversity ratio: {verbalized/baseline:.2f}x")   # target ≥ 1.5
```

If the ratio is below 1.5, increase `N` or add an explicit instruction
asking the model to maximise variety across the candidates.

## 5. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Weights do not sum to 1.0 | Model rounds | Normalise in step 3 |
| All candidates are near-identical | `N` too small or prompt too constrained | Increase `N`; relax constraints |
| JSON parse error | Response truncated | Increase `max_tokens` |
| Low diversity ratio on factual tasks | Expected — VS gains are largest on open-ended creative tasks | Use VS selectively |

## Reference

This skill follows the Agent Skills format — see [references/references.bib](references/references.bib) (`agentskills2025specification`).

The diversity technique is from \[zhang2025verbalized\] — see [references/references.bib](references/references.bib).
