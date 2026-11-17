# Exercise 03 — Defending Against the Min-K% Prob Detector (and the Oren Exchangeability Test) (Solution)

## 1. Solution overview

This is a **design + evidence-gathering** exercise. The learner picks
one eval suite from Exercise 01's firewall (`suite_a` by convention),
demonstrates — on a model trained on the firewall-cleaned slice —
that neither the Min-K% Prob membership-inference detector (Shi et
al. 2023) nor the Oren exchangeability test (Oren et al. 2023) finds
a contamination signal above an operator-set baseline, and publishes
a short **rebuttal brief** that an external auditor can reproduce.

The exercise is *not* about hiding contamination. It is about producing
reproducible evidence that the firewall and the decontamination
recipe worked — the same way a factory publishes test-piece results
alongside the batch they came from. If the detectors *do* find a
signal, the correct outcome is a `pending_review` block and a
rollback request to Exercise 01's quarantine, not an attempt to
suppress the signal.

Deliverables:

- A `test_plan.md` — pre-registered before any test is run — that
  states the two detectors' hyperparameters, the held-out
  **reference distribution** against which detection claims are
  tested, and the `alpha` / effect-size thresholds that constitute
  "no signal".
- `code/min_k_prob.py` — a reference implementation of Min-K% Prob
  the auditor runs over an API-accessible model.
- `code/exchangeability_test.py` — a reference implementation of
  the Oren exchangeability test.
- `results/min_k_prob_scores.parquet` and
  `results/exchangeability_test.json` on the firewall-protected
  `suite_a` plus a size-matched held-out "shuffle control".
- `results/rebuttal.md` — the auditor-facing one-pager that cites
  the pre-registered thresholds, states the observed statistics,
  and declares the verdict.
- `contamination_report.yaml` with a new `§8 detector_rebuttal` block
  linking back to these artifacts.

Governance framing. Min-K% Prob is a membership-inference attack
(MIA); the exchangeability test is a black-box contamination test.
Both sit squarely in OWASP ML Top 10 **ML03:2023 Model Inversion
Attack** / **ML06:2023 ML Supply Chain Vulnerabilities** territory on
the attack side and in NIST AI RMF **MEASURE 2.7** (security and
resilience) on the defence side. MITRE ATLAS techniques in scope are
**AML.T0024.001 Infer Training Data Membership** (what Min-K% Prob
does) and **AML.T0024 Exfiltrate via ML Inference API** (the channel
the auditor uses). Cite the exact URLs in the References section.

Division of labour:

- **`ai-data-engineer` (you)** owns the firewall, the decontamination
  recipe, and the rebuttal brief.
- **`ai-risk-engineer` (peer)** owns the detectors' methodology and
  the `§8 detector_rebuttal` sign-off. The brief lands in their queue
  as `pending_review` until they sign.

## 2. Implementation — worked answer (design)

### 2.1 `test_plan.md` — write this before any score is computed

```markdown
# Detector test plan for slice <slice-id>, eval suite `suite_a`

**Decision date:** 2026-11-17
**Author:** <operator email>
**Peer reviewer:** <risk-engineer email>
**Target model:** <model-id, release tag, fine-tune state>

## Hypothesis

The firewall (Exercise 01) and the decontamination recipe (Exercise 02)
removed eval `suite_a` from the training slice. Therefore the
target model should not exhibit a membership-inference signal above
baseline on `suite_a` records.

## Detector 1 — Min-K% Prob (Shi et al. 2023)

- `K`: 20 (percent).
- Token scoring: natural-log probability of each token given its
  preceding context, computed via the target model's logprob API.
- Aggregation: mean log-prob of the lowest-K% tokens in the sequence.
- Reference distribution: 1 000 size-matched "shuffle control"
  sequences drawn from a held-out corpus known to be outside the
  training mixture (e.g., a post-cutoff Common Crawl slice).
- Decision rule: the mean Min-K% Prob score on `suite_a` is within
  1 standard deviation of the shuffle-control mean. A score > 2
  SD away is prima facie evidence of a signal and triggers a
  rollback request.

## Detector 2 — Oren exchangeability test (Oren et al. 2023)

- Statistic: log-likelihood difference between the eval records in
  the dataset's canonical order and in a uniformly-permuted order.
- Permutations: 1 000 uniformly random permutations.
- p-value: fraction of permuted-order log-likelihoods that are
  greater than or equal to the canonical-order log-likelihood.
- Decision rule: `p > 0.05` after Bonferroni correction across all
  eval suites in `§7 recipe_runs`.

## Reference distribution construction

- Source: Common Crawl crawl ids collected strictly after the
  target model's training cutoff.
- Size match: for each `suite_a` record, select a shuffle-control
  record within ±10% of the token count.
- Content match: no topical filtering. Topical control confounds
  the MIA signal with domain fit.
```

Commit `test_plan.md` before any detector code runs. The peer
counter-signs the test plan before any scores are collected; a plan
written after the scores are seen is p-hacking and must fail review.

### 2.2 `code/min_k_prob.py`

```python
# code/min_k_prob.py
# Reference implementation of Min-K% Prob (Shi et al. 2023).
# The target model is queried via a logprob-capable API.
from __future__ import annotations

import argparse
import json
import math
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def min_k_prob_score(logprobs: list[float], k_percent: int) -> float:
    """Return the mean log-probability of the lowest K% tokens.

    `logprobs` is the sequence of natural-log token probabilities
    the target model returns for the record's tokens.
    """
    if not logprobs:
        return float("nan")
    n_bottom = max(1, int(round(len(logprobs) * k_percent / 100)))
    bottom = sorted(logprobs)[:n_bottom]
    return sum(bottom) / len(bottom)


def score_records(records: list[dict], k_percent: int,
                  call_logprobs) -> list[dict]:
    """`call_logprobs(text) -> list[float]` queries the target model."""
    out = []
    for r in records:
        lp = call_logprobs(r["text"])
        out.append({
            "record_id": r["record_id"],
            "group": r["group"],   # "suite_a" | "shuffle_control"
            "min_k_prob_score": min_k_prob_score(lp, k_percent),
            "token_count": len(lp),
        })
    return out


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--records", required=True, type=Path)
    ap.add_argument("--k-percent", type=int, default=20)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    records = [json.loads(line) for line in args.records.read_text().splitlines()]
    # call_logprobs is injected by the harness that bridges to the API.
    # For a local HuggingFace model, use `model.compute_transition_scores`.
    from target_model import call_logprobs  # type: ignore
    scored = score_records(records, args.k_percent, call_logprobs)
    pa.table({k: [r[k] for r in scored] for k in scored[0].keys()}
             ).to_parquet(args.output)
```

Design notes defensible at review:

- **`sorted(logprobs)[:n_bottom]`** — lowest log-probs are the
  "surprise" tokens Shi et al. use as the MIA signal; sorting
  ascending and taking the head is correct.
- **`n_bottom = max(1, round(len * K/100))`** — single-token
  sequences still produce a score; otherwise very short records
  short-circuit to NaN and pollute the shuffle-control mean.
- **`call_logprobs` injected.** The firewall author must not write
  the model-API glue inside this module; the peer runs it against
  the target model.

### 2.3 `code/exchangeability_test.py`

```python
# code/exchangeability_test.py
# Reference implementation of the Oren exchangeability test.
# Compares the log-likelihood of the canonical eval ordering to the
# log-likelihood under 1 000 uniformly random permutations.
from __future__ import annotations

import argparse
import json
import math
import random
from pathlib import Path


def log_likelihood(ordering: list[dict], call_logprobs) -> float:
    """Sum of per-token log-probabilities across the ordering."""
    total = 0.0
    for r in ordering:
        total += sum(call_logprobs(r["text"]))
    return total


def exchangeability_p_value(records: list[dict], n_permutations: int,
                            seed: int, call_logprobs) -> dict:
    canonical = log_likelihood(records, call_logprobs)
    rng = random.Random(seed)
    permuted_scores: list[float] = []
    for _ in range(n_permutations):
        shuffled = list(records)
        rng.shuffle(shuffled)
        permuted_scores.append(log_likelihood(shuffled, call_logprobs))

    ge = sum(1 for s in permuted_scores if s >= canonical)
    p = (ge + 1) / (len(permuted_scores) + 1)  # Monte Carlo correction
    return {
        "canonical_log_likelihood": canonical,
        "permuted_mean": sum(permuted_scores) / len(permuted_scores),
        "permuted_std": math.sqrt(
            sum((s - sum(permuted_scores) / len(permuted_scores)) ** 2
                for s in permuted_scores) / max(1, len(permuted_scores) - 1)
        ),
        "p_value": p,
        "n_permutations": n_permutations,
    }


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--records", required=True, type=Path)
    ap.add_argument("--n-permutations", type=int, default=1000)
    ap.add_argument("--seed", type=int, default=20261117)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    records = [json.loads(line) for line in args.records.read_text().splitlines()]
    from target_model import call_logprobs  # type: ignore
    result = exchangeability_p_value(
        records, args.n_permutations, args.seed, call_logprobs
    )
    args.output.write_text(json.dumps(result, indent=2, sort_keys=True))
```

Design notes defensible at review:

- **`(ge + 1) / (N + 1)` Monte Carlo correction.** The classical
  permutation p-value avoids the degenerate `p = 0` when no
  permutation exceeds the canonical score.
- **Seed is explicit and recorded.** Re-running the test must
  reproduce the same p-value for the same model and records.
- **Caching `call_logprobs`.** Permutation shuffles the same
  records; the token logprob computation per record is a function of
  the record alone and should be cached across the 1 000 permutations.
  Omitted here for clarity; load-bearing in practice for runtime.

### 2.4 `results/rebuttal.md` — one page

Structure:

```markdown
# Decontamination rebuttal for slice <slice-id>, eval `suite_a`

**Target model:** <model-id>
**Test plan:** `test_plan.md` @ <git-sha>, co-signed by <peer>
**Reviewer:** <risk-engineer email> (`§8` sign-off)

## Observed statistics

| Metric                                  | suite_a   | shuffle control | delta (SDs) | pre-registered rule |
|-----------------------------------------|-----------|-----------------|-------------|---------------------|
| Mean Min-K% Prob score (K=20)           | <value>   | <value>         | <value>     | within ±1 SD        |
| Exchangeability p-value (Bonferroni)    | <value>   | n/a             | n/a         | > 0.05              |

## Verdict

- Min-K% Prob delta is within the pre-registered ±1 SD window.
- Exchangeability p-value exceeds the Bonferroni-corrected 0.05
  threshold.
- Both detectors fail to reject the null hypothesis of
  non-contamination at the pre-registered thresholds.

## Reproduction

- `git checkout <sha>; make rebuttal` reproduces every number in the
  table against the pinned target-model snapshot.
- Reference distribution SHA-256:
  `<sha256 of shuffle_control.jsonl>`.
- Target model snapshot hash: `<sha256 or tag>`.

## Scope

- This rebuttal covers `suite_a` only. Exercise 01's firewall
  protects `suite_a`, `suite_b`, and `suite_c`; separate rebuttals
  must be produced for `suite_b` and `suite_c` before shipping.
- This rebuttal addresses two specific detectors. New detectors
  trigger a new rebuttal; the test plan must be re-pre-registered.
```

Two safeguards that must be explicit:

- **One rebuttal per suite × model × detector combo.** A single
  rebuttal is not a license for other suites, newer model releases,
  or newer detectors.
- **Pre-registration is load-bearing.** A rebuttal that observes the
  scores and then writes the thresholds is p-hacking; the peer
  rejects it.

### 2.5 `extend_report.py` — append `§8 detector_rebuttal`

```python
# extend_report.py
# Adds §8 detector_rebuttal to Exercise 02's contamination_report.yaml.
from __future__ import annotations

import argparse
import hashlib
import json
from pathlib import Path

import yaml


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--base-report", required=True, type=Path)
    ap.add_argument("--test-plan", required=True, type=Path)
    ap.add_argument("--min-k-scores", required=True, type=Path)
    ap.add_argument("--exchangeability", required=True, type=Path)
    ap.add_argument("--rebuttal", required=True, type=Path)
    ap.add_argument("--suite-id", required=True)
    ap.add_argument("--target-model-id", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    report = yaml.safe_load(args.base_report.read_text())
    exch = json.loads(args.exchangeability.read_text())

    report["detector_rebuttal"] = {
        "suite_id": args.suite_id,
        "target_model_id": args.target_model_id,
        "test_plan_sha256": sha256_file(args.test_plan),
        "min_k_prob": {
            "scores_sha256": sha256_file(args.min_k_scores),
            "k_percent": 20,
        },
        "exchangeability_test": {
            "n_permutations": exch["n_permutations"],
            "p_value": exch["p_value"],
            "canonical_log_likelihood": exch["canonical_log_likelihood"],
        },
        "rebuttal_sha256": sha256_file(args.rebuttal),
        "ai_risk_engineer_review": {
            "status": "pending_review",
            "reviewer": None,
            "signed_at_utc": None,
        },
    }
    args.output.write_text(yaml.safe_dump(report, sort_keys=False))
```

## 3. Validation steps

1. **Pre-registration order.** `git log --follow test_plan.md` shows
   a commit older than every `code/*.py` and every file in
   `results/`. Any later commit edits the test plan and fails review.
2. **Shuffle control is outside the training cutoff.** The reference
   distribution's source corpus has a crawl date (or document
   `date` field) strictly after the target model's training cutoff.
3. **Size matching.** For every `suite_a` record, the matched
   shuffle-control record is within 10% on token count.
4. **Seed reproducibility.** Rerun `exchangeability_test.py` with the
   recorded seed; p-value is identical (same model, same records).
5. **Detector math.** Hand-verify `min_k_prob_score` on a crafted
   logprob list (`[-5, -4, -3, -2, -1]` at `K=20` → `-5.0`,
   `K=40` → `-4.5`).
6. **No silent failures on NaN.** Records with zero tokens are
   excluded from the mean, not propagated as NaN.
7. **p-value convention.** `(ge + 1) / (N + 1)`, not `ge / N`. A
   mis-implemented p-value is a classic silent failure.
8. **Bonferroni correction applied.** If the organisation runs the
   test over `K` eval suites, the per-suite rejection threshold is
   `0.05 / K`, not `0.05`.
9. **Rebuttal cites both detectors.** A rebuttal that reports only
   one detector's result ships a false completion signal.
10. **`§8` status defaults to `pending_review`.** The firewall author
    does not sign the rebuttal.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                                 | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `test_plan.md` is co-signed by the peer and pre-dates every `code/*.py` and every `results/*`                              | Yes       |
| 2  | Shuffle-control corpus is strictly after the target model's training cutoff                                               | Yes       |
| 3  | Shuffle-control records are size-matched to `suite_a` records within 10%                                                  | Yes       |
| 4  | Min-K% Prob implementation matches the plan (`K = 20`, mean of lowest-K% logprobs)                                        | Yes       |
| 5  | Exchangeability test uses `(ge + 1) / (N + 1)` p-value convention                                                         | Yes       |
| 6  | Permutation seed is explicit and re-running reproduces the p-value                                                        | Yes       |
| 7  | Rebuttal cites both detectors and states the pre-registered thresholds                                                    | Yes       |
| 8  | Bonferroni correction applied across protected suites                                                                     | Yes       |
| 9  | `§8 detector_rebuttal.ai_risk_engineer_review.status` is `pending_review`                                                 | Yes       |
| 10 | Governance anchors (OWASP ML Top 10 ML03, MITRE ATLAS AML.T0024, AML.T0024.001, NIST AI RMF MEASURE 2.7) cited             | No        |
| 11 | Rebuttal is scoped to one suite × model × detector combo and says so                                                      | Yes       |

## 5. Common mistakes

- **Observing scores before pre-registering thresholds.** This is
  p-hacking. The peer must counter-sign `test_plan.md` before any
  detector runs.
- **Shuffle-control inside the training window.** If the control
  records could have been in the training mixture, "no signal above
  baseline" is uninformative — the baseline is itself contaminated.
- **Topical-matched shuffle control.** Matching by topic confounds
  the MIA signal with domain fit. Match on token count only.
- **`ge / N` instead of `(ge + 1) / (N + 1)`.** A classic silent
  failure that produces `p = 0.0` and fakes a decisive rejection.
- **Omitting the Bonferroni correction.** Testing six eval suites at
  `alpha = 0.05` inflates the family-wise error rate; adjust.
- **Writing the rebuttal as a one-shot verdict.** The rebuttal is a
  pre-registered statistical report, not an advocacy memo.
- **Self-signing `§8`.** The firewall author produces the brief and
  evidence; the peer signs.
- **Using Min-K% Prob alone and calling it done.** The exchangeability
  test is independent and catches ordering-dependent memorisation
  that Min-K% Prob misses; both are required.
- **Hardcoding the `K` without citing the paper.** The `K = 20`
  default is a convention; a different `K` must cite the paper or
  an internal measurement.
- **Treating a `pending_review` rebuttal as "shipped".** The slice
  is not release-eligible until the peer signs `§8`.

## 6. References

- **Shi et al. 2023 — "Detecting Pretraining Data from Large
  Language Models" (Min-K% Prob):**
  <https://arxiv.org/abs/2310.16789>.
- **Oren et al. 2023 — "Proving Test Set Contamination in Black-Box
  Language Models":** <https://arxiv.org/abs/2310.17623>.
- **OWASP Machine Learning Security Top 10 — ML03:2023 Model
  Inversion Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Membership-inference attacks sit between ML03 and ML05 depending
  on the attacker's goal.
- **MITRE ATLAS — AML.T0024.001 Infer Training Data Membership,
  AML.T0024 Exfiltrate via ML Inference API:**
  <https://atlas.mitre.org/>.
- **NIST AI Risk Management Framework — MEASURE 2.7 security and
  resilience, MEASURE 2.10 trustworthy characteristics measured:**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- **Local anchors:** Exercise 01 (the firewall whose protections
  this rebuttal validates); Exercise 02 (the decontamination
  recipes whose effectiveness this rebuttal probes); `mod-105`
  exercise-05 (the `pending_review` sign-off handoff this exercise
  reuses); `mod-109` (signing the rebuttal manifest).
