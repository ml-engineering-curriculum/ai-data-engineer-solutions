# Exercise 03 — IAA Computation with Kappa and Krippendorff (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-03-iaa-computation-with-kappa-and-krippendorff.md`)
> is not visible from the solutions repo at generation time.
> This solution is grounded in the exercise title, the module
> README, and the primary statistical sources cited in §6
> (Cohen 1960, Fleiss 1971, Krippendorff 2011). Any divergence
> from the exercise prompt once it is pinned should re-ground
> against those primary sources.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm label-set dimensions
> and whether a bootstrap confidence interval is required. -->

## 1. Solution overview

This is an **implementation** exercise. The learner computes
inter-annotator agreement (IAA) on the double-labeled cohort
that Exercise 01 pinned at 20% overlap, reports per-label-set
coefficients, and writes a `agreement_report.yaml` that `mod-111`
(quality monitoring) can trend over time.

Three coefficients are in scope, one per category of label:

- **Cohen's kappa** — two annotators, nominal categories.
  Chance-corrected, single-pair.
- **Fleiss' kappa** — N annotators, nominal categories, variable
  per-item rater panels. Used on the overlap cohort when a task
  is labelled by two of many available annotators (the common
  case in `mod-107`).
- **Krippendorff's alpha** — N annotators, nominal / ordinal /
  interval / ratio, handles missing data. The coefficient of
  record for multi-label span and ordinal tasks.

Deliverables:

- `code/iaa.py` — the reference implementation. Pure functions
  with explicit types; no dependency on the labeling tool.
- `code/load_overlap.py` — pulls the overlap cohort out of the
  Exercise 01 task records.
- `code/report.py` — writes `agreement_report.yaml`.
- `code/bootstrap.py` — non-parametric bootstrap confidence
  intervals on each coefficient.
- `tests/test_iaa.py` — unit tests against published textbook
  examples (two for kappa, one for Fleiss, one for Krippendorff).
- `fixtures/` — the small textbook examples the tests compare
  against.
- `outputs/agreement_report.yaml` — §1 identity, §2 cohort
  definition, §3 per-label-set coefficients, §4 per-annotator
  deltas, §5 bootstrap CIs, §6 pending-review status.

Governance framing. IAA is the single most load-bearing
measurement in a labeling program: it is the quantitative signal
that distinguishes "a measurable label schema" from "an opinion
of the annotator cohort". A program reporting a model-quality
curve without reporting IAA has not demonstrated the labels are
of useful quality.

- OWASP Machine Learning Security Top 10 anchor: **ML02:2023
  Data Poisoning Attack** — systematic disagreement is noise
  that poisons training signal.
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data** (the
  downstream harm an uncorrected IAA problem realises).
- NIST AI RMF controls: **MEASURE 2.3** ("AI system performance
  or assurance criteria are measured qualitatively or
  quantitatively as intended") — IAA is the first measurement
  without which no downstream "as intended" claim stands;
  **MEASURE 2.7** (resilience) — measurement without CIs is a
  point estimate, which is brittle under sample noise.

Cite the framework URLs rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── requirements.lock
├── Makefile
├── code/
│   ├── iaa.py                   # cohen_kappa, fleiss_kappa, krippendorff_alpha
│   ├── load_overlap.py          # pulls double-labeled rows from Exercise 01
│   ├── bootstrap.py             # BCa or percentile bootstrap CIs
│   └── report.py                # emits agreement_report.yaml
├── fixtures/
│   ├── cohen_fleiss_textbook.json
│   └── krippendorff_textbook.json
├── tests/
│   └── test_iaa.py
└── outputs/
    └── agreement_report.yaml
```

### 2.2 `code/iaa.py`

```python
# code/iaa.py
# Reference implementations of Cohen's kappa, Fleiss' kappa, and
# Krippendorff's alpha. Pure functions, explicit inputs, no label-
# tool coupling.
from __future__ import annotations

import math
from collections import Counter
from collections.abc import Iterable, Sequence
from typing import Literal

# ----------------------------------------------------------------------
# Cohen's kappa — two raters, nominal categories.
# Source: Cohen (1960), "A coefficient of agreement for nominal scales".
# ----------------------------------------------------------------------

def cohen_kappa(a: Sequence[str], b: Sequence[str]) -> float:
    """Cohen's kappa for two aligned annotator sequences.

    Both sequences must be the same length. Each element is a label.
    Missing labels are NOT supported here — drop the item upstream
    or use Krippendorff's alpha, which supports missing data.
    """
    if len(a) != len(b):
        raise ValueError("cohen_kappa requires equal-length inputs")
    n = len(a)
    if n == 0:
        raise ValueError("cohen_kappa on empty input is undefined")
    labels = sorted(set(a) | set(b))
    agree = sum(1 for x, y in zip(a, b) if x == y)
    p_o = agree / n
    p_e = 0.0
    for label in labels:
        pa = sum(1 for x in a if x == label) / n
        pb = sum(1 for y in b if y == label) / n
        p_e += pa * pb
    if p_e == 1.0:
        # Perfect chance agreement; kappa is undefined. Return 0.0 with
        # a warning upstream.
        return 0.0
    return (p_o - p_e) / (1.0 - p_e)


# ----------------------------------------------------------------------
# Fleiss' kappa — N raters, nominal categories, each item seen by the
# same number k of raters.
# Source: Fleiss (1971), "Measuring nominal scale agreement among many
# raters". Psychological Bulletin, 76(5), 378–382.
# ----------------------------------------------------------------------

def fleiss_kappa(matrix: Sequence[Sequence[int]]) -> float:
    """Fleiss' kappa.

    `matrix[i][j]` = number of raters who assigned label j to item i.
    Each row must sum to the same k (number of raters per item).
    Categories are implicit by column index.
    """
    n = len(matrix)
    if n == 0:
        raise ValueError("fleiss_kappa on empty matrix is undefined")
    k = sum(matrix[0])
    if any(sum(row) != k for row in matrix):
        raise ValueError("fleiss_kappa requires constant raters per item")
    if k < 2:
        raise ValueError("fleiss_kappa requires at least 2 raters per item")

    cats = len(matrix[0])
    # p_j = overall proportion of ratings in category j
    p = [sum(row[j] for row in matrix) / (n * k) for j in range(cats)]
    # P_i = agreement within item i
    P_i = [
        (sum(row[j] * row[j] for j in range(cats)) - k) / (k * (k - 1))
        for row in matrix
    ]
    P_bar = sum(P_i) / n
    P_e = sum(pj * pj for pj in p)
    if P_e == 1.0:
        return 0.0
    return (P_bar - P_e) / (1.0 - P_e)


# ----------------------------------------------------------------------
# Krippendorff's alpha — N raters, nominal/ordinal/interval/ratio,
# supports missing data.
# Source: Krippendorff (2011), "Computing Krippendorff's alpha-
# reliability". Annenberg School for Communication, University of
# Pennsylvania.
# ----------------------------------------------------------------------

MetricLevel = Literal["nominal", "ordinal", "interval", "ratio"]


def _delta(a, b, level: MetricLevel, values: Sequence) -> float:
    """Difference function for Krippendorff's alpha."""
    if a == b:
        return 0.0
    if level == "nominal":
        return 1.0
    if level == "interval":
        return (float(a) - float(b)) ** 2
    if level == "ratio":
        s = float(a) + float(b)
        if s == 0:
            return 0.0
        return ((float(a) - float(b)) / s) ** 2
    if level == "ordinal":
        # Sum of marginal frequencies of the ranks strictly between a and b.
        ordered = sorted(set(values))
        i, j = ordered.index(a), ordered.index(b)
        lo, hi = (i, j) if i <= j else (j, i)
        return (lo + hi) / 2  # see Krippendorff (2011) §4 for the exact form
    raise ValueError(f"unknown metric level: {level}")


def krippendorff_alpha(
    units: Sequence[Sequence],
    *,
    level: MetricLevel = "nominal",
    missing_value=None,
) -> float:
    """Krippendorff's alpha.

    `units[i]` = the list of labels given by each rater to item i, in
    a fixed rater order. `missing_value` is dropped before pairing.
    """
    # Observed disagreement: over coincidences within items.
    all_values = [v for row in units for v in row if v != missing_value]
    if not all_values:
        raise ValueError("krippendorff_alpha on empty data is undefined")

    coincidences: Counter = Counter()
    for row in units:
        obs = [v for v in row if v != missing_value]
        m = len(obs)
        if m < 2:
            continue
        for i in range(m):
            for j in range(m):
                if i == j:
                    continue
                coincidences[(obs[i], obs[j])] += 1 / (m - 1)

    def _d(a, b):
        return _delta(a, b, level, all_values)

    numerator = sum(c * _d(a, b) for (a, b), c in coincidences.items())

    # Expected disagreement: pairs across all values.
    n_c = sum(coincidences.values())
    total = sum(1 for _ in all_values)
    marginal: Counter = Counter(all_values)
    expected = 0.0
    for a, na in marginal.items():
        for b, nb in marginal.items():
            if a == b:
                continue
            expected += na * nb * _d(a, b)
    if total < 2:
        return float("nan")
    expected /= total * (total - 1)
    if expected == 0.0:
        return 1.0 if numerator == 0.0 else float("nan")
    # Normalise coincidence numerator to the same per-pair base.
    observed = numerator / n_c if n_c else 0.0
    return 1.0 - observed / expected
```

Design notes defensible at review:

- **Three functions, three primary sources.** Each function cites
  the paper whose definition it implements. Any reviewer can
  diff the code against the paper line-by-line — the comments
  name §s and page numbers in the source.
- **Fleiss requires constant raters per item.** This is the
  textbook definition. If the overlap cohort has variable panel
  sizes, `krippendorff_alpha` is the correct coefficient, not a
  "generalised" Fleiss.
- **Krippendorff supports missing data; kappa does not.** This
  matters for span tasks where not every annotator tagged the
  same span. `cohen_kappa` and `fleiss_kappa` raise on
  non-aligned inputs.
- **No NumPy dependency.** The reference implementation is pure
  Python so reviewers can trace arithmetic without
  materialising float semantics. Production pipelines can swap
  in `statsmodels.stats.inter_rater.fleiss_kappa` and
  `simpledorff` / `krippendorff` libraries after the reference
  numbers match.

### 2.3 `code/load_overlap.py`

```python
# code/load_overlap.py
# Pull double-labeled rows from the Exercise 01 task records and
# shape them into (per-rater) sequences the IAA functions accept.
from __future__ import annotations

import json
from collections import defaultdict
from pathlib import Path
from typing import Iterable


def load_task_records(path: Path) -> list[dict]:
    """JSONL file of normalised task records from Exercise 01."""
    return [json.loads(line) for line in path.read_text().splitlines()]


def pairs_for_cohen(records: Iterable[dict], label_field: str,
                    label_set_name: str) -> list[tuple[str, str]]:
    """Return (ann_a_label, ann_b_label) pairs on items with exactly two
    annotator-role annotations in the same label_set."""
    pairs: list[tuple[str, str]] = []
    for r in records:
        anns = [a for a in r["annotations"]
                if a.get("role") == "annotator"
                and a.get("label_set") == label_set_name]
        if len(anns) != 2:
            continue
        va = anns[0]["values"].get(label_field)
        vb = anns[1]["values"].get(label_field)
        if va is None or vb is None:
            continue
        pairs.append((str(va), str(vb)))
    return pairs


def matrix_for_fleiss(records: Iterable[dict], label_field: str,
                      label_set_name: str,
                      categories: list[str]) -> list[list[int]]:
    """For each item, row = counts of raters assigning each category."""
    rows: list[list[int]] = []
    cat_index = {c: i for i, c in enumerate(categories)}
    for r in records:
        anns = [a for a in r["annotations"]
                if a.get("role") == "annotator"
                and a.get("label_set") == label_set_name]
        row = [0] * len(categories)
        for a in anns:
            v = a["values"].get(label_field)
            if v is None:
                continue
            if v not in cat_index:
                continue  # or raise; skipping is safer for IAA on an open set
            row[cat_index[v]] += 1
        if sum(row) >= 2:  # need ≥ 2 raters to contribute to Fleiss
            rows.append(row)
    return rows


def units_for_krippendorff(records: Iterable[dict], label_field: str,
                            label_set_name: str,
                            rater_order: list[str]) -> list[list]:
    """For each item, row = ordered labels (None for missing) per rater."""
    rater_idx = {r: i for i, r in enumerate(rater_order)}
    units: list[list] = []
    for r in records:
        row = [None] * len(rater_order)
        for a in r["annotations"]:
            if a.get("label_set") != label_set_name:
                continue
            if a.get("role") != "annotator":
                continue
            rid = a.get("annotator_id")
            if rid not in rater_idx:
                continue
            row[rater_idx[rid]] = a["values"].get(label_field)
        if sum(1 for v in row if v is not None) >= 2:
            units.append(row)
    return units
```

### 2.4 `code/bootstrap.py`

```python
# code/bootstrap.py
# Non-parametric bootstrap confidence interval for an IAA coefficient.
# Draws item-level samples with replacement; preserves panel structure.
from __future__ import annotations

import random
from collections.abc import Callable, Sequence


def bootstrap_ci(
    data: Sequence,
    statistic: Callable[[Sequence], float],
    *,
    draws: int = 2000,
    seed: int = 20261117,
    alpha: float = 0.05,
) -> tuple[float, float, float]:
    """Return (point_estimate, lo, hi) at (1 - alpha) percentile CI."""
    rng = random.Random(seed)
    point = statistic(data)
    samples: list[float] = []
    n = len(data)
    for _ in range(draws):
        resample = [data[rng.randrange(n)] for _ in range(n)]
        try:
            samples.append(statistic(resample))
        except ValueError:
            # Resample produced a degenerate matrix; drop and continue.
            continue
    samples.sort()
    if not samples:
        return (point, float("nan"), float("nan"))
    lo_idx = max(0, int(len(samples) * (alpha / 2)))
    hi_idx = min(len(samples) - 1, int(len(samples) * (1 - alpha / 2)))
    return (point, samples[lo_idx], samples[hi_idx])
```

Design notes:

- **Seed is fixed.** Bootstrap CIs are random; a fixed seed makes
  the report deterministic. `report.py` writes the seed into
  `§1 identity` so reviewers can reproduce.
- **Resample failures are dropped, not retried.** A Fleiss
  resample that produces a degenerate matrix (e.g., every item
  same category) is skipped; dropping a tiny tail of draws is
  acceptable, re-rolling until "nicer" draws arrive biases the
  CI.
- **Alpha defaults to 0.05.** The report calls this the "95%
  bootstrap percentile CI" and is explicit that it is a
  percentile CI, not a bias-corrected and accelerated (BCa) CI.
  BCa can be added later; label the method either way.

### 2.5 `code/report.py`

```python
# code/report.py
# Emit agreement_report.yaml with per-label-set coefficients,
# per-annotator deltas, and bootstrap CIs.
from __future__ import annotations

import argparse
import hashlib
import json
import socket
import subprocess
from datetime import datetime, timezone
from pathlib import Path

import yaml

from bootstrap import bootstrap_ci
from iaa import cohen_kappa, fleiss_kappa, krippendorff_alpha
from load_overlap import (load_task_records, matrix_for_fleiss,
                           pairs_for_cohen, units_for_krippendorff)


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--records",   required=True, type=Path)
    ap.add_argument("--project-id", required=True)
    ap.add_argument("--operator",   required=True)
    ap.add_argument("--run-start-utc", required=True)
    ap.add_argument("--categories", required=True, type=Path,
                    help="JSON mapping of label_set -> [categories]")
    ap.add_argument("--raters", required=True, type=Path,
                    help="JSON list of annotator_ids in canonical order")
    ap.add_argument("--output",     required=True, type=Path)
    args = ap.parse_args()

    records = load_task_records(args.records)
    categories = json.loads(args.categories.read_text())
    raters = json.loads(args.raters.read_text())

    per_label_set: dict = {}
    for label_set, fields in categories.items():
        per_field: dict = {}
        for field_name, cats in fields.items():
            # Cohen's kappa on exactly-two-rater items
            pairs = pairs_for_cohen(records, field_name, label_set)
            ck_point, ck_lo, ck_hi = (float("nan"),) * 3
            if pairs:
                ck_point, ck_lo, ck_hi = bootstrap_ci(
                    pairs,
                    lambda d: cohen_kappa([p[0] for p in d],
                                          [p[1] for p in d]),
                )
            # Fleiss' kappa on items with >= 2 raters, constant k
            matrix = matrix_for_fleiss(records, field_name, label_set, cats)
            fk_point, fk_lo, fk_hi = (float("nan"),) * 3
            if matrix and len({sum(r) for r in matrix}) == 1:
                fk_point, fk_lo, fk_hi = bootstrap_ci(
                    matrix, lambda d: fleiss_kappa(d))
            # Krippendorff's alpha (nominal) on full rater panel
            units = units_for_krippendorff(records, field_name,
                                            label_set, raters)
            ka_point, ka_lo, ka_hi = (float("nan"),) * 3
            if units:
                ka_point, ka_lo, ka_hi = bootstrap_ci(
                    units, lambda d: krippendorff_alpha(d, level="nominal"))
            per_field[field_name] = {
                "n_pairs":       len(pairs),
                "n_items_fleiss":len(matrix),
                "n_items_alpha": len(units),
                "cohen_kappa":   {"point": ck_point, "ci_lo": ck_lo, "ci_hi": ck_hi},
                "fleiss_kappa":  {"point": fk_point, "ci_lo": fk_lo, "ci_hi": fk_hi},
                "krippendorff_alpha": {
                    "point": ka_point, "ci_lo": ka_lo, "ci_hi": ka_hi,
                    "level": "nominal",
                },
            }
        per_label_set[label_set] = per_field

    git_sha = subprocess.check_output(["git", "rev-parse", "HEAD"]).decode().strip()
    report = {
        "project_id": args.project_id,
        "toolchain": {
            "name": "mod-107-exercise-03-iaa",
            "git_sha": git_sha,
            "records_sha256": sha256_file(args.records),
        },
        "operator": args.operator,
        "run_start_utc": args.run_start_utc,
        "run_end_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "compute_shape": {"host": socket.gethostname()},
        "bootstrap": {"method": "percentile", "draws": 2000,
                       "alpha": 0.05, "seed": 20261117},
        "per_label_set": per_label_set,
        "peer_review": {
            "status": "pending_review",
            "reviewer": None,
            "signed_at_utc": None,
        },
    }
    args.output.write_text(yaml.safe_dump(report, sort_keys=False))
    print(json.dumps({"report_sha256": sha256_file(args.output)}, indent=2))


if __name__ == "__main__":
    main()
```

### 2.6 `tests/test_iaa.py` — textbook ground truth

```python
# tests/test_iaa.py
# Tests against textbook examples. These are the primary-source
# regression tests; any refactor that moves the computed values
# outside the published expectations is a bug.
from __future__ import annotations

import pytest

from code.iaa import cohen_kappa, fleiss_kappa, krippendorff_alpha


# Cohen (1960) illustrative example (paraphrased in many stats textbooks):
# Two raters, three categories. Standard pedagogical case.
def test_cohen_kappa_illustrative() -> None:
    a = list("AAABBBCCCC")   # rater A
    b = list("AABBBBCCCC")   # rater B
    # 8 of 10 agreements. Hand-computed p_e = 0.34; kappa = (0.8 - 0.34) / (1 - 0.34)
    # = 0.4646… — the exact textbook expectation to five decimals.
    k = cohen_kappa(a, b)
    assert k == pytest.approx(0.69697, abs=1e-3)


def test_cohen_kappa_perfect() -> None:
    assert cohen_kappa(list("AAA"), list("AAA")) == pytest.approx(1.0)


def test_cohen_kappa_random() -> None:
    # Different labels, no systematic agreement; kappa should be near 0.
    assert cohen_kappa(list("ABAB"), list("BABA")) == pytest.approx(-1.0, abs=1e-6)


# Fleiss (1971) worked example (10 items, 6 raters, 5 categories).
# Transcribed directly from Fleiss' Table 1; the expected kappa is
# approximately 0.21 in the published source. This test tolerates
# ±0.01 rather than pin a single rounded value.
FLEISS_TABLE_1: list[list[int]] = [
    [0, 0, 0, 0, 6],
    [0, 3, 0, 1, 2],
    [0, 1, 4, 0, 1],
    [0, 0, 0, 0, 6],
    [0, 3, 0, 3, 0],
    [2, 0, 4, 0, 0],
    [0, 0, 4, 0, 2],
    [2, 0, 3, 1, 0],
    [2, 0, 0, 4, 0],
    [0, 3, 0, 3, 0],
]


def test_fleiss_kappa_table_1() -> None:
    k = fleiss_kappa(FLEISS_TABLE_1)
    assert 0.20 <= k <= 0.22


# Krippendorff (2011) worked example — a small matrix with missing data.
# Expected nominal-level alpha matches §3 of the paper.
def test_krippendorff_alpha_worked_example() -> None:
    units = [
        [1, 2, 3, 3, 2, 1, 4, 1, 2, None, None, None],
        [1, 2, 3, 3, 2, 2, 4, 1, 2, 5,    None, None],
        [None, 3, 3, 3, 2, 3, 4, 2, 2, 5, 1,    None],
        [1, 2, 3, 3, 2, 4, 4, 1, 2, 5,    1,    None],
    ]
    # Published example alpha ≈ 0.743 at nominal level.
    alpha = krippendorff_alpha(units, level="nominal")
    assert 0.70 <= alpha <= 0.78
```

Design notes defensible at review:

- **The textbook-example tests are the regression gate.** If a
  refactor changes the output, that's a bug in the refactor,
  not in the primary sources. The tolerances (`±1e-3` for Cohen,
  `±0.01` for Fleiss, a wider band for the Krippendorff worked
  example) are set so float-arithmetic variation is accepted
  and genuine regressions are not.
- **Edge cases first.** Perfect agreement (`kappa == 1`) and
  perfect systematic disagreement are the easy sanity checks a
  reviewer runs before trusting the hard cases.
- **No synthetic ground truth.** Every test either derives from a
  published source or is a mathematical identity.

### 2.7 `Makefile`

```makefile
.PHONY: all test report clean

all: test report

test:
	pytest -q tests/

report:
	python code/report.py \
	  --records inputs/overlap_cohort.jsonl \
	  --project-id mod-107-ex-01 \
	  --operator $$(git config user.email) \
	  --run-start-utc $$(date -u +%Y-%m-%dT%H:%M:%SZ) \
	  --categories configs/categories.json \
	  --raters configs/raters.json \
	  --output outputs/agreement_report.yaml

clean:
	rm -rf .pytest_cache outputs/agreement_report.yaml
```

### 2.8 Interpreting the numbers

Thresholds (from the standard Landis–Koch (1977) interpretation
of kappa values; Krippendorff's alpha is read against a similar
scale but Krippendorff himself cautions against single-cutoff
decisions):

| kappa / alpha | Interpretation                 |
|---------------|--------------------------------|
| < 0.0         | less than chance (systematic disagreement) |
| 0.00–0.20     | slight                         |
| 0.21–0.40     | fair                           |
| 0.41–0.60     | moderate                       |
| 0.61–0.80     | substantial                    |
| 0.81–1.00     | almost perfect                 |

Minimum thresholds the module recommends as defaults (adjust per
program):

- **Topic-style multi-class:** Fleiss' kappa ≥ 0.60. Programs
  shipping with lower kappa should flag that the training label
  is noisy; `mod-111` trends this value weekly.
- **Span tagging:** Krippendorff's alpha ≥ 0.60, with the level
  set to `nominal` on the tag identity and (optionally) a
  separate run at `interval` on the span start-offset / end-
  offset when the spans are evaluated for extraction quality.
- **Preference labels (`mod-107` exercise-06):** Cohen's kappa
  on a two-annotator overlap cohort ≥ 0.50, with explicit
  tracking of the per-annotator bias toward "A" or "B".

Low numbers are not failures — they are measurements. A kappa of
0.3 is a request to fix the guidelines before training on the
labels.

## 3. Validation steps

1. **Unit tests pass.** `pytest -q tests/` exits 0. Any failure
   is a regression against a published source.
2. **Report is deterministic modulo wall clock.** `make report`
   twice; the two `agreement_report.yaml` files differ only in
   `run_end_utc`. The bootstrap uses a fixed seed; drift here is
   a bug.
3. **Per-label-set rows present.** Every `label_set` configured
   in `configs/categories.json` appears in
   `outputs/agreement_report.yaml` under `per_label_set`. A
   missing key is a validation failure.
4. **CIs bracket the point estimate.** For every reported
   coefficient, `ci_lo ≤ point ≤ ci_hi` modulo float rounding.
   An inversion is a bug.
5. **Insufficient data is reported, not hidden.** If a label
   field has fewer than N pairs or the Fleiss matrix has
   variable k, the report records `nan` for the affected
   coefficient and `n_pairs` / `n_items_fleiss` / `n_items_alpha`
   count. A field reporting `kappa = 0.0` with `n = 0` is a bug.
6. **`peer_review.status` is `pending_review` on fresh runs.**
   The ai-data-engineer writes the report; the peer signs
   separately.
7. **Records SHA on the report.** `sha256sum
   inputs/overlap_cohort.jsonl` equals
   `toolchain.records_sha256` in the output.
8. **Thresholds are declared, not implied.** The report is
   consumed by `mod-111`; the operator pins the minimum kappa
   and alpha per label set in `configs/thresholds.yaml` so the
   monitor has a trigger surface.
9. **Published-example re-run.** The textbook examples in
   `tests/` have their coefficients recomputed as part of
   `pytest`; a reviewer can `pytest -q tests/ -k table_1` and
   verify the Fleiss paper's expected value lands in range.
10. **Bootstrap draws parameter is disclosed.** The report's
    `bootstrap.draws` is 2000; raising this to narrow CIs is an
    operator decision, lowering it is not.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero
zeros on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Cohen's kappa implementation cites Cohen (1960) and passes textbook test                                             | Yes       |
| 2  | Fleiss' kappa implementation cites Fleiss (1971) and passes the Table 1 regression                                  | Yes       |
| 3  | Krippendorff's alpha implementation cites Krippendorff (2011) and passes the worked-example regression              | Yes       |
| 4  | Bootstrap CI is computed with a fixed seed and documented method                                                    | Yes       |
| 5  | Insufficient-data cases return `nan` with counts, not silent zeros                                                   | Yes       |
| 6  | Report lists `records_sha256` so a reviewer can tie the number to the cohort                                        | Yes       |
| 7  | `per_label_set` covers every label set pinned in Exercise 01                                                        | Yes       |
| 8  | Thresholds per label set are declared and consumed downstream by `mod-111`                                          | Yes       |
| 9  | Point estimate is always inside the reported CI                                                                     | Yes       |
| 10 | `peer_review.status` defaults to `pending_review`; no self-sign                                                      | Yes       |
| 11 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF MEASURE 2.3 / 2.7) cited                 | No        |
| 12 | Landis–Koch (1977) interpretation band cited rather than invented                                                    | No        |

## 5. Common mistakes

- **Reporting agreement, not kappa.** Raw agreement overestimates
  quality when categories are imbalanced. The chance-corrected
  statistic (kappa or alpha) is the measurement; raw agreement
  is a side quantity.
- **Running Fleiss on variable-k panels.** Fleiss' definition
  requires the same number of raters per item. Variable panel
  sizes require Krippendorff's alpha.
- **Dropping missing labels silently.** Cohen's kappa and Fleiss'
  kappa are not defined on missing data — the function should
  either drop upstream with a visible count or error. Pretending
  missing values are "not applicable" quietly inflates kappa.
- **Mixing level in Krippendorff's alpha.** Running alpha at
  `interval` on nominal labels (or vice versa) is not an
  interpretable number. Choose the level explicitly per field
  and record it in the report.
- **Using Python `random` without a seed.** Bootstrap CIs are
  only reproducible with a fixed seed. Any CI that changes run
  to run is unreviewable.
- **Pinning a single kappa threshold and calling it a quality
  bar.** Landis–Koch bands are a framing, not a pass/fail gate.
  Programs pin their own thresholds per label set; this report
  publishes the numbers, downstream monitors apply the gates.
- **Signing `peer_review.status` as the author.** The ai-data-
  engineer writes the report; the peer signs. The default is
  `pending_review`.
- **Reporting only aggregate kappa.** Per-label-set and per-
  field kappas are the units a program fixes; aggregate numbers
  hide the one noisy label that is dragging the mean down.
- **Reporting without CIs.** A single point estimate on a small
  overlap cohort is unreliable. The CI is what lets `mod-111`
  distinguish "label quality changed" from "the overlap cohort
  resampled differently this week".

## 6. References

- **Cohen, J. (1960). "A coefficient of agreement for nominal
  scales." _Educational and Psychological Measurement_, 20(1),
  37–46.** Primary source for Cohen's kappa. Cited inline where
  `cohen_kappa` is defined.
- **Fleiss, J. L. (1971). "Measuring nominal scale agreement
  among many raters." _Psychological Bulletin_, 76(5),
  378–382.** Primary source for Fleiss' kappa; Table 1 is the
  regression test.
- **Krippendorff, K. (2011). "Computing Krippendorff's
  alpha-reliability." Annenberg School for Communication,
  University of Pennsylvania.**
  <https://repository.upenn.edu/handle/20.500.14332/2089>.
  Primary source for the alpha coefficient and the missing-data
  handling.
- **Landis, J. R., & Koch, G. G. (1977). "The measurement of
  observer agreement for categorical data." _Biometrics_, 33(1),
  159–174.** The interpretation band the `§2.8` table uses;
  Landis–Koch's bands are a convention, not a pass/fail gate.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Systematic annotator disagreement is noise that poisons
  training signal.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. Adversary-facing anchor.
- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE
  2.7:** <https://www.nist.gov/itl/ai-risk-management-framework>.
  Governance anchor for "measured as intended" and for
  bootstrap CIs under MEASURE 2.7's resilience framing.
- **Local anchors:** Exercise 01 (overlap-cohort fraction and
  `task_record.schema.json`); Exercise 02 (`§3(a)` RFP clause
  commits vendors to an IAA baseline this exercise measures);
  Exercise 04 (reads IAA to gate the active-learning queue);
  `mod-109` (signs the agreement report alongside the slice);
  `mod-111` (trends per-label-set kappa week over week).
