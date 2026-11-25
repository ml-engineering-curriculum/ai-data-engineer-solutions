# Exercise 04 — DPO Preference-Pair Synthesis Pipeline (Solution)

> **Exercise-prompt note.** The learning file for this exercise
> (`exercise-04-dpo-preference-pair-synthesis-pipeline.md`) is a
> scaffolded stub. This solution is grounded in Chapter 06
> (`06-preference-pair-synthesis-for-dpo-orpo-kto.md`), which is
> fully written, and in the exercise contract from `README.md`. Any
> future divergence from the exercise prompt should re-ground
> against that chapter.
> <!-- pending-verify: confirm the exercise prompt once it lands; the
> scaffolded stub does not yet pin pair count, judge shape, or
> hand-off format. -->

## 1. Solution overview

This is an **implementation** exercise. The learner builds a DPO-
shaped preference-pair synthesis pipeline: multi-generator candidate
sampling, judge-model scoring with position-swap control, pair
construction by a documented policy, difficulty-stratification
bucketing, and a pair-balance report ready for hand-off to
`fine-tuning-engineer`.

Deliverables:

- `outputs/preference_pairs.parquet` — one row per pair with
  `prompt`, `chosen`, `rejected`, scores, `difficulty_bucket`,
  `pair_source`.
- `filters/judge_scores.parquet` — per-candidate judge scores
  (including both position-swap directions for position-bias control).
- `filters/pair_balance_report.yaml` — difficulty bucket counts,
  per-source counts, length ratios, and judge-vs-human calibration
  summary.
- `recipes/preference_pair_config.yaml` — pinned generator mix,
  judge identity, pair-construction policy, difficulty buckets.
- `filters/judge_vs_human_calibration.md` — a small sample (≥ 100
  pairs) hand-labelled to establish judge-human agreement on this
  prompt pool.

Governance framing. The preference-pair operation sets the training
signal that the policy is literally optimised against. Signal bias
is the failure mode that compounds — the policy learns the bias and
the bias cannot be removed after training without replaying DPO.

- OWASP Machine Learning Security Top 10 anchor: **ML02:2023 Data
  Poisoning Attack** — judge-position bias and length bias poison
  the policy by rewarding surface artifacts instead of substantive
  quality.
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data**
  (judge-reward hacking propagates to the policy); **AML.T0018.000
  Manipulate AI Model (Modify AI Model)** (the DPO training step is
  the modification; the pair corpus is the modification surface).
- NIST AI Risk Management Framework controls: **MEASURE 2.3** (AI
  system performance evaluated as intended — judge fidelity is the
  measurement surface); **MEASURE 2.11** (fairness and bias —
  length bias and position bias are measurable bias signals);
  **GOVERN 1.2** (role-and-responsibility accountability for the
  hand-off to `fine-tuning-engineer`).

Cite the framework URLs rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── preference_pair_config.yaml
├── code/
│   ├── generate_candidates.py       # multi-generator fan-out
│   ├── judge.py                      # pairwise judge w/ position swap
│   ├── pair_policies.py              # best-vs-worst, best-vs-median, adjacent
│   ├── buckets.py                    # difficulty bucketing
│   ├── balance_report.py             # pair-balance summary
│   └── calibrate_judge.py            # judge-vs-human sampling harness
├── inputs/
│   ├── prompt_pool.parquet            # from exercise-01 or -02
│   └── prompt_pool.parquet.sha256
├── outputs/
│   ├── preference_pairs.parquet
│   └── preference_pairs.parquet.sha256
└── filters/
    ├── candidates.parquet             # all candidates with judge scores
    ├── judge_scores.parquet
    ├── pair_balance_report.yaml
    └── judge_vs_human_calibration.md
```

### 2.2 `code/generate_candidates.py` — multi-generator fan-out

```python
# code/generate_candidates.py
# Chapter 06 §"Candidate diversity". Fan out across generators +
# temperatures + system-prompt variants to produce N structurally
# different candidates per prompt.
from __future__ import annotations

from dataclasses import dataclass


@dataclass(slots=True)
class GeneratorSpec:
    id: str
    model_id: str
    temperature: float
    system_prompt_variant: str | None   # None | "concise" | "verbose" | ...
    candidates_per_prompt: int


DEFAULT_MIX: list[GeneratorSpec] = [
    GeneratorSpec("student_policy_checkpoint_42",
                   "internal:student-v0", 0.8, None, 1),
    GeneratorSpec("teacher_default",
                   "anthropic:claude-3-5-sonnet-20241022",
                   0.7, None, 2),
    GeneratorSpec("teacher_concise_high_t",
                   "anthropic:claude-3-5-sonnet-20241022",
                   1.0, "concise", 2),
]


def fan_out(prompt: str, mix: list[GeneratorSpec], clients: dict) -> list[dict]:
    rows: list[dict] = []
    for gen in mix:
        client = clients[gen.model_id]
        for i in range(gen.candidates_per_prompt):
            system = _system_for_variant(gen.system_prompt_variant)
            resp = client.generate(prompt=prompt, system=system,
                                     temperature=gen.temperature,
                                     max_tokens=2048)
            rows.append({
                "prompt": prompt,
                "generator_id": gen.id,
                "model_id": gen.model_id,
                "temperature": gen.temperature,
                "system_prompt_variant": gen.system_prompt_variant,
                "candidate_index": i,
                "response": resp,
            })
    return rows


def _system_for_variant(variant: str | None) -> str | None:
    if variant is None:
        return None
    if variant == "concise":
        return "Answer concisely, in two sentences or fewer."
    if variant == "verbose":
        return "Explain in detail, showing your reasoning."
    raise ValueError(f"unknown variant: {variant}")
```

Design notes defensible at review:

- **Policy-anchored candidate included.** Chapter 06 §"Diversity
  mechanisms" names this as the strongest diversity mechanism: the
  student model's candidate anchors the pair space to the policy's
  current distribution. Dropping it is a common bug that makes the
  preference signal less effective for iterative DPO.
- **Multi-temperature sampling per teacher is cheap diversity.**
  Two teacher candidates at `T=0.7` and two at `T=1.0` with a
  variant system prompt is the paper-typical setup.
- **System prompt variants are enumerated, not free-text.** Treats
  the variants as a schema; drift between runs is visible.

### 2.3 Candidate-level filters

Before scoring, drop clearly-unsuitable candidates:

```python
# code/candidate_filters.py
from __future__ import annotations

import re
from rouge_score import rouge_scorer

_SCORER = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=False)

REFUSAL_PREFIXES = (
    "i cannot", "i can't", "i'm sorry", "i am sorry",
    "as an ai language model", "as a language model",
)
PROMPT_LEAK_MIN_LEN = 20   # candidates shorter than this echo the prompt


def drop_candidate(candidate: dict, prompt: str, siblings: list[dict],
                    sibling_dedup_rouge_l: float = 0.9,
                    drop_refusals: bool = True) -> str | None:
    text = candidate["response"] or ""
    if not text.strip():
        return "empty"
    if candidate.get("truncated_by_max_tokens"):
        return "truncated"
    if drop_refusals and any(text.lower().startswith(p) for p in REFUSAL_PREFIXES):
        return "refusal"
    if len(text) < PROMPT_LEAK_MIN_LEN and prompt.lower()[:20] in text.lower():
        return "prompt_leak"
    for sib in siblings:
        if sib is candidate:
            continue
        if _SCORER.score(text, sib["response"] or "")["rougeL"].fmeasure > sibling_dedup_rouge_l:
            return "sibling_dup"
    return None
```

### 2.4 `code/judge.py` — pairwise judge with position-swap

```python
# code/judge.py
# Chapter 06 §"Pairwise comparison judge". Two calls per pair, with
# positions swapped. Only pairs where both calls agree are retained.
from __future__ import annotations

from dataclasses import dataclass


@dataclass(slots=True)
class JudgeVerdict:
    chosen_id: str | None     # None if disagreement
    agreement: bool
    forward_winner: str
    reverse_winner: str


_JUDGE_PROMPT = """You are judging two candidate responses to a user prompt.
Pick the better response. Reply with exactly 'A' or 'B'.

Prompt:
{prompt}

Response A:
{a}

Response B:
{b}

Which is better? Reply 'A' or 'B'."""


def judge_pair(prompt: str, a: dict, b: dict, judge_client) -> JudgeVerdict:
    forward = _ask(prompt, a["response"], b["response"], judge_client)
    reverse = _ask(prompt, b["response"], a["response"], judge_client)
    # Normalise reverse: if judge said 'A' on reverse call, that means
    # b (shown as A on reverse) won.
    reverse_winner = b["generator_id"] if reverse == "A" else a["generator_id"]
    forward_winner = a["generator_id"] if forward == "A" else b["generator_id"]
    agree = forward_winner == reverse_winner
    return JudgeVerdict(
        chosen_id=forward_winner if agree else None,
        agreement=agree,
        forward_winner=forward_winner,
        reverse_winner=reverse_winner,
    )


def _ask(prompt: str, a_text: str, b_text: str, client) -> str:
    resp = client.generate(prompt=_JUDGE_PROMPT.format(prompt=prompt, a=a_text,
                                                         b=b_text),
                            temperature=0.0, max_tokens=4)
    for c in resp.upper():
        if c in ("A", "B"):
            return c
    return "A"   # pick deterministically on malformed response; the pair
                  # will fail position-swap anyway.
```

Design notes worth defending at review:

- **Two calls, both directions.** The position-swap pattern halves
  the surviving pair count but every surviving pair carries
  position-bias-controlled signal. Chapter 06 §"Position-bias and
  length-bias controls" is the anchor.
- **Judge temperature is 0.** The judge is a classifier, not a
  generator — reproducibility is more valuable than diversity at
  this step.
- **Disagreement is recorded, not discarded silently.** The
  per-pair `agreement=False` rows land in `judge_scores.parquet`
  so the disagreement rate shows up in the balance report.

### 2.5 `code/pair_policies.py` — pair construction

```python
# code/pair_policies.py
# Policies: best_vs_worst, best_vs_median, adjacent_rank, threshold.
from __future__ import annotations

from typing import Callable


def best_vs_worst(candidates: list[dict]) -> list[tuple[dict, dict]]:
    """One pair per prompt: argmax vs argmin on judge score."""
    if len(candidates) < 2:
        return []
    ranked = sorted(candidates, key=lambda c: c["judge_score"])
    return [(ranked[-1], ranked[0])]


def best_vs_median(candidates: list[dict]) -> list[tuple[dict, dict]]:
    if len(candidates) < 2:
        return []
    ranked = sorted(candidates, key=lambda c: c["judge_score"])
    median = ranked[len(ranked) // 2]
    return [(ranked[-1], median)]


def adjacent_rank(candidates: list[dict]) -> list[tuple[dict, dict]]:
    ranked = sorted(candidates, key=lambda c: c["judge_score"])
    return [(ranked[i + 1], ranked[i]) for i in range(len(ranked) - 1)]


def threshold(candidates: list[dict], tau: float = 0.7) -> list[tuple[dict, dict]]:
    chosen = [c for c in candidates if c["judge_score"] >= tau]
    rejected = [c for c in candidates if c["judge_score"] < tau]
    return [(c, r) for c in chosen for r in rejected]


POLICIES: dict[str, Callable[[list[dict]], list[tuple[dict, dict]]]] = {
    "best_vs_worst": best_vs_worst,
    "best_vs_median": best_vs_median,
    "adjacent_rank": adjacent_rank,
    "threshold": threshold,
}
```

### 2.6 `code/buckets.py` — difficulty stratification

```python
# code/buckets.py
# A pair's "difficulty" is the score gap between chosen and rejected.
# Bucket by score-gap; small gap = hard pair; large gap = easy pair.
from __future__ import annotations

DEFAULT_BUCKETS = [
    (0.0, 0.2),   # hard
    (0.2, 0.4),
    (0.4, 0.6),
    (0.6, 0.8),
    (0.8, 1.0),   # easy
]


def bucket_for(chosen_score: float, rejected_score: float,
                buckets=DEFAULT_BUCKETS) -> str:
    gap = abs(chosen_score - rejected_score)
    for lo, hi in buckets:
        if lo <= gap < hi:
            return f"gap_{lo:.1f}_to_{hi:.1f}"
    return f"gap_{buckets[-1][0]:.1f}_to_{buckets[-1][1]:.1f}"
```

### 2.7 `code/balance_report.py` — the hand-off artifact

```python
# code/balance_report.py
# Emits filters/pair_balance_report.yaml. Chapter 06 §"Pair balance"
# names four axes: judge-score distribution, prompt-source balance,
# task-type balance, positive-vs-negative balance (for KTO). This
# exercise ships DPO pairs, so KTO's balance axis is a future-work
# field.
from __future__ import annotations

from collections import Counter
from pathlib import Path
from statistics import mean, stdev

import yaml


def build_report(pairs: list[dict], calibration_summary: dict,
                  output: Path) -> None:
    gap = [abs(p["judge_score_chosen"] - p["judge_score_rejected"])
           for p in pairs]
    length_ratio = [len(p["chosen"]) / max(len(p["rejected"]), 1)
                    for p in pairs]
    report = {
        "pair_count": len(pairs),
        "difficulty_buckets": dict(Counter(p["difficulty_bucket"] for p in pairs)),
        "judge_score_chosen": {
            "mean": mean(p["judge_score_chosen"] for p in pairs),
            "stdev": stdev(p["judge_score_chosen"] for p in pairs) if len(pairs) > 1 else 0,
        },
        "judge_score_rejected": {
            "mean": mean(p["judge_score_rejected"] for p in pairs),
            "stdev": stdev(p["judge_score_rejected"] for p in pairs) if len(pairs) > 1 else 0,
        },
        "score_gap": {
            "mean": mean(gap) if gap else 0,
            "stdev": stdev(gap) if len(gap) > 1 else 0,
        },
        "length_ratio_chosen_over_rejected": {
            "mean": mean(length_ratio) if length_ratio else 0,
            "p95": _percentile(length_ratio, 0.95),
        },
        "prompt_source_counts":
            dict(Counter(p["pair_source"] for p in pairs)),
        "task_type_counts":
            dict(Counter(p.get("task_type", "unknown") for p in pairs)),
        "judge_vs_human_calibration": calibration_summary,
        "position_swap_disagreement_rate":
            sum(1 for p in pairs if p.get("judge_disagreement")) / len(pairs)
            if pairs else 0,
    }
    output.write_text(yaml.safe_dump(report, sort_keys=False))


def _percentile(xs: list[float], q: float) -> float:
    if not xs:
        return 0.0
    s = sorted(xs)
    i = int(q * (len(s) - 1))
    return s[i]
```

### 2.8 `filters/judge_vs_human_calibration.md`

The judge is upstream of the policy; its calibration is not
optional. Chapter 06 §"Position-bias and length-bias controls" is
explicit: below ~70% judge-human agreement, the judge is not
usable; swap it.

Recipe:

1. Uniform-sample ≥ 100 pairs that survived the judge filter.
2. Ask ≥ 2 human reviewers to independently verdict each pair as
   "chosen is better", "rejected is better", or "tie".
3. Compute judge-human agreement (judge chose = human majority).
4. Report also judge-human agreement conditioned on length-ratio
   bucket (there's often a big drop in the bucket where chosen is
   much longer than rejected).
5. If judge-human agreement is below 70%, swap the judge; if there
   is a length-bucket collapse, apply length-matched candidate
   generation or an explicit length-controlled judge prompt.

Format:

```markdown
# Judge-vs-human calibration

Sample size: 120 pairs
Reviewers: @reviewer-a, @reviewer-b (both ai-data-engineer)
Inter-reviewer agreement: 0.86

Judge-human agreement (overall): 0.78
By length-ratio bucket:
  chosen/rejected in [0.5, 1.5]: 0.84
  chosen/rejected in [1.5, 2.0]: 0.72
  chosen/rejected > 2.0:         0.58   # length bias; see mitigation below

Mitigation: candidate generation capped at max_tokens=2048 per
generator; judge prompt explicitly instructs "ignore length when
judging quality". Judge-human agreement on pairs with length ratio
> 2.0 is still below the 70% threshold — these pairs are flagged
for review and excluded from the training hand-off.
```

### 2.9 `recipes/preference_pair_config.yaml`

```yaml
# recipes/preference_pair_config.yaml
prompt_pool:
  path: inputs/prompt_pool.parquet
  sha256: <64-hex-digest>
  item_count: 10000

candidate_generation:
  generators:
    - id: student_policy_checkpoint_42
      model_id: internal:student-v0
      temperature: 0.8
      candidates_per_prompt: 1
    - id: teacher_default
      model_id: anthropic:claude-3-5-sonnet-20241022
      temperature: 0.7
      candidates_per_prompt: 2
    - id: teacher_concise_high_t
      model_id: anthropic:claude-3-5-sonnet-20241022
      temperature: 1.0
      system_prompt_variant: concise
      candidates_per_prompt: 2
  candidate_level_filters:
    drop_refusals: true
    sibling_dedup_rouge_l: 0.9
    max_tokens: 2048

judge:
  shape: pairwise
  judge_model: anthropic:claude-3-5-sonnet-20241022
  judge_temperature: 0.0
  position_swap: true
  length_match_tolerance: 1.5
  reference_calibration_path: filters/judge_vs_human_calibration.md
  independence_warning: true    # overlaps with teacher generators

pair_construction:
  policy: best_vs_median
  pairs_per_prompt: 2
  difficulty_buckets:
    - [0.0, 0.2]
    - [0.2, 0.4]
    - [0.4, 0.6]
    - [0.6, 0.8]
    - [0.8, 1.0]

hand_off:
  format: dpo
  train_split: 0.95
  eval_split: 0.05
  recipient: fine-tuning-engineer

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

## 3. Validation steps

1. **Prompt pool hash matches** the pin in the manifest.
2. **Every candidate carries its generator identity.**
   `filters/candidates.parquet` has `generator_id`, `model_id`,
   `temperature`, `system_prompt_variant`, `candidate_index`.
3. **Position-swap control is enforced.** For every judged pair,
   the judge was called twice and agreement is recorded. Pairs
   with `agreement=False` do not appear in the final pairs parquet.
4. **No sibling duplicates.** For every prompt, pairwise ROUGE-L
   across sibling candidates is below the dedup threshold.
5. **Length-ratio controls.** The pair-balance report shows
   mean(chosen/rejected length) and p95; p95 > 2.0 is a yellow
   flag that must be acknowledged in the report.
6. **Judge-human calibration present.** `judge_vs_human_
   calibration.md` reports sample size ≥ 100 and an overall
   agreement number. Agreement < 0.70 blocks hand-off.
7. **Difficulty buckets are non-degenerate.** No single bucket
   contains > 80% of pairs. A degenerate distribution is a bucket-
   collapse signal (Chapter 06 §"Difficulty bucket collapse").
8. **Judge-generator overlap flagged.** When the judge and one of
   the generators share the model ID, `independence_warning: true`
   is pinned in the manifest and the calibration report
   acknowledges the overlap.
9. **`chosen` and `rejected` are from different generators or at
   least different sampling knobs.** A pair where chosen and
   rejected both came from the same generator at the same
   temperature is usually a sibling-dup escape.
10. **Hand-off split is reproducible.** `train_split` and
    `eval_split` use a hashed-prompt-id partition, not a random
    shuffle; reproducible across runs.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Multi-generator candidate mix with ≥ 3 distinct generator-+-temperature combinations                                 | Yes       |
| 2  | Policy-anchored candidate (student model) included in the mix                                                       | Yes       |
| 3  | Candidate-level filters (refusal, empty, truncated, prompt-leak, sibling-dedup) implemented                         | Yes       |
| 4  | Pairwise judge runs with position swap; disagreement rate is recorded                                                | Yes       |
| 5  | Judge temperature is 0 for reproducibility                                                                          | No        |
| 6  | Pair-construction policy is pinned in the manifest and matches the code path                                        | Yes       |
| 7  | Difficulty-stratification buckets are computed from the score gap and land as a parquet column                       | Yes       |
| 8  | Pair-balance report covers judge-score distribution, length ratio, bucket counts, prompt-source counts               | Yes       |
| 9  | Judge-vs-human calibration report exists with sample size ≥ 100 and reports agreement                                | Yes       |
| 10 | Judge-generator overlap is flagged in the manifest when present                                                     | Yes       |
| 11 | Train/eval split is hash-stable across runs                                                                         | Yes       |
| 12 | Hand-off format matches the recipient's trainer (`dpo` default, `orpo`/`kto`/`rlhf` supported by conversion)         | Yes       |
| 13 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0018.000/T0020, NIST AI RMF MEASURE 2.3 / MEASURE 2.11 / GOVERN 1.2) cited | No |
| 14 | Unverifiable claims marked `<!-- pending-verify: ... -->`                                                           | Yes       |
| 15 | Hand-off is logged as a mod-109 lineage event (ledger pointer in the manifest)                                      | Yes       |

## 5. Common mistakes

- **Single-generator candidate mix.** N draws at one temperature
  from one model collapses the judgeable surface. Chapter 06
  §"Candidate diversity" is explicit — diversity must be
  engineered, not assumed.
- **Skipping position swap.** Judges are position-biased. Running
  a single-direction judge call lets the surface artifact through.
  Position-swap halves the pair count; the retained pairs are
  higher signal.
- **Length-bias unmeasured.** A length ratio of 2× chosen-over-
  rejected teaches "longer is better". Report the ratio in the
  balance report; a p95 > 2.0 is a flag.
- **Same model as teacher and judge, no calibration.** Permitted
  but must be flagged and calibrated; `independence_warning: true`
  is the explicit marker.
- **Ignoring difficulty-bucket collapse.** If 80% of pairs fall in
  the easy bucket, the trainer cannot stratify. The balance
  report's `difficulty_buckets` dict is the diagnostic.
- **Random train/eval split.** A random shuffle gives different
  splits across runs. Use a hashed-prompt-id partition so the
  split is reproducible.
- **Dropping the policy-anchored candidate.** Iterative DPO needs
  the student's own candidate as one of the rejected slot
  sources; without it, the signal does not teach the student to
  move away from its own distribution.
- **Shipping pairs without a judge-human calibration run.** The
  calibration is a release gate in Chapter 06; shipping a judge
  with unknown fidelity is a rubric failure.
- **Not pinning the judge model version.** The judge is upstream
  of the training signal. A judge upgrade mid-run changes the
  distribution; pin the dated version and log the upgrade as a
  run boundary.
- **No refusal exclusion documentation.** If refusals are kept as
  valid chosen responses (e.g., when teaching refusal behaviour),
  the manifest must document the exception.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchor for judge-reward hacking entering the
  policy via DPO.
- **MITRE ATLAS — AML.T0018.000 Manipulate AI Model (Modify AI
  Model); AML.T0020 Poison Training Data:** <https://atlas.mitre.org/>.
  The adversary-facing anchors for the DPO training step and for
  biased preference pairs feeding it.
- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE 2.11,
  GOVERN 1.2:** <https://www.nist.gov/itl/ai-risk-management-framework>.
  MEASURE 2.3 anchors judge-fidelity-as-measurement; MEASURE 2.11
  anchors length-and-position bias measurement; GOVERN 1.2 anchors
  the hand-off accountability to `fine-tuning-engineer`.
- **DPO (Rafailov et al. 2023, NeurIPS 2023), arXiv 2305.18290:**
  <https://arxiv.org/abs/2305.18290>. The primary source for the
  paired preference format consumed by the hand-off.
- **ORPO (Hong et al. 2024, EMNLP 2024), arXiv 2403.07691:**
  <https://arxiv.org/abs/2403.07691>. Same pair shape, different
  loss; the hand-off format converts.
- **KTO (Ethayarajh et al. 2024), arXiv 2402.01306:**
  <https://arxiv.org/abs/2402.01306>. The unpaired binary-labelled
  variant; the exercise emits DPO pairs, KTO is a conversion.
- **InstructGPT (Ouyang et al. 2022), arXiv 2203.02155:**
  <https://arxiv.org/abs/2203.02155>. The classical RLHF reference
  for the scalar-reward judge variant.
- **Prometheus / Prometheus 2 (Kim et al. 2024):**
  <https://arxiv.org/abs/2405.01535>. Open-weight rubric judges
  referenced by Chapter 06 §"Known reference judges" as the
  independent-judge option.
- **Local anchors:** `mod-108` Chapter 06 (the full recipe and the
  three operational knobs); `mod-108` exercise-01 or -02 (the
  prompt pool input); `mod-107` Chapter 08 (the human preference-
  labelling protocol ported in `judge_vs_human_calibration.md`);
  `mod-109` (hand-off lineage logging).
