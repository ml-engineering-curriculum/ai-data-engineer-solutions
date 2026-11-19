# Exercise 04 — Active Learning Budget-vs-Quality Run (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner runs a controlled
experiment that compares **uncertainty-based active learning** against
**uniform random sampling** at matched labeling budgets, producing a
reproducible budget-vs-quality curve. The output is not a one-off
accuracy number — it is a **decision artifact**: the labeling
program uses the curve to choose how many items to send to annotators
in each round, and to detect when the active-learning policy stops
paying for itself.

Deliverables:

- A seed pool + unlabeled pool loaded from a labeled corpus (the
  "labels" are held back and used as an oracle at selection time;
  this simulates annotation).
- At least **two selection strategies**: `random` (baseline) and an
  `uncertainty` strategy (e.g., least-confidence or margin for
  classification). A third strategy (`entropy`) is highly
  recommended for defensible curves.
- A **round-based driver** that, at each round, trains a model on
  the current labeled set, scores the unlabeled pool, selects the
  next `k` items, "labels" them from the oracle, retrains, and
  evaluates on a held-out test set.
- A `runs/run_metrics.parquet` + `runs/budget_vs_quality.svg` curve.
- A `report.md` stating the **crossover budget** (the budget at
  which active learning beats random by a reviewer-accepted
  margin), the **saturation point** (where the curve flattens),
  and a **recommended labeling-round size** justified by the
  curve.
- A pytest suite that pins the metric outputs on a seeded fixture
  so the whole run is byte-reproducible.

Governance framing. Active learning concentrates labeling budget on
hard-to-label items — which is exactly where biased labels have the
largest downstream model impact. The control anchor is **NIST AI RMF
MEASURE 2.3** (measured performance) combined with **MEASURE 2.11**
(fairness-related metrics) because an uncertainty policy can
systematically under-sample a class the model is already confident
about and over-sample a borderline class — a representation drift
that must be *measured*, not assumed. On the adversary side, an
uncertainty-based oracle query policy is itself a pattern an
attacker can exploit (query-driven label flipping maps to **MITRE
ATLAS AML.T0020 Poison Training Data** when the pool has
adversarial content); the experiment design includes a canary
check for this.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── Makefile
├── config.yaml                     # budget schedule, strategies, seed, model
├── code/
│   ├── data.py                     # load and split into labeled/pool/test
│   ├── strategies.py               # random, uncertainty, margin, entropy
│   ├── model.py                    # thin logistic-regression wrapper
│   ├── run_al.py                   # the round-based driver (one run)
│   ├── run_matrix.py               # orchestrate N strategies × M seeds
│   ├── aggregate.py                # mean / CI over seeds → run_metrics.parquet
│   ├── plot_curve.py               # budget_vs_quality.svg
│   └── report.py                   # emit report.md with crossover + saturation
├── tests/
│   ├── test_strategies.py
│   └── test_driver_reproducibility.py
└── runs/
    ├── raw/                        # per-seed per-strategy jsonl traces
    ├── run_metrics.parquet
    ├── budget_vs_quality.svg
    └── report.md
```

### 2.2 `config.yaml`

```yaml
dataset:
  name: ag_news_small            # any labeled text classifier dataset works;
                                 # the learner swaps in the real pool in prod.
  test_size: 2000
  labeled_seed_size: 200         # starting labeled set
  pool_size: 10000               # unlabeled candidate pool

budget:
  rounds: 10
  per_round: 100                 # k items labeled per round
  # Total budget = labeled_seed_size + rounds*per_round = 1200

strategies:
  - random
  - uncertainty
  - margin
  - entropy

seeds: [20261118, 20261119, 20261120, 20261121, 20261122]

model:
  type: sklearn_logistic_regression
  C: 1.0
  max_iter: 2000
  class_weight: balanced

evaluation:
  metric: macro_f1               # not raw accuracy — catches class drift
  secondary_metric: balanced_accuracy
  decimal_places: 4

crossover_margin: 0.01           # AL must beat random by ≥ 0.01 macro-F1 at
                                 # the reported crossover budget.
```

Design notes:

- **Macro-F1 as primary.** Raw accuracy inflates with class
  imbalance and is exactly the number uncertainty sampling can
  silently degrade on; macro-F1 catches that.
- **Multiple seeds.** A single seed cannot distinguish "AL beats
  random" from "AL was lucky this seed". 5 seeds is the minimum
  defensible; more is better where compute allows.
- **Budget stated as `(seed + rounds*per_round)`.** The curve's
  x-axis is total labeled items, not round index, so the
  comparison is budget-matched.

### 2.3 `code/data.py`

```python
# code/data.py
# Split a labeled dataset into (labeled_seed, unlabeled_pool, test)
# deterministically. The "unlabeled pool" is labeled in reality; the
# driver pretends the labels are hidden and queries the oracle at
# selection time.
from __future__ import annotations

import hashlib
from dataclasses import dataclass

import numpy as np


@dataclass(frozen=True)
class Split:
    x_labeled: np.ndarray
    y_labeled: np.ndarray
    x_pool: np.ndarray
    y_pool: np.ndarray        # the hidden oracle labels
    x_test: np.ndarray
    y_test: np.ndarray


def deterministic_split(x: np.ndarray, y: np.ndarray, *,
                        seed: int, labeled_seed_size: int,
                        pool_size: int, test_size: int) -> Split:
    rng = np.random.default_rng(seed)
    perm = rng.permutation(len(x))
    i_test = perm[:test_size]
    i_seed = perm[test_size:test_size + labeled_seed_size]
    i_pool = perm[test_size + labeled_seed_size
                   : test_size + labeled_seed_size + pool_size]
    return Split(
        x_labeled=x[i_seed], y_labeled=y[i_seed],
        x_pool=x[i_pool], y_pool=y[i_pool],
        x_test=x[i_test], y_test=y[i_test],
    )


def fingerprint(split: Split) -> str:
    """Hash the split so reports can prove two runs used the same split."""
    h = hashlib.sha256()
    for arr in (split.x_labeled, split.y_labeled,
                split.x_pool, split.y_pool,
                split.x_test, split.y_test):
        h.update(np.ascontiguousarray(arr).tobytes())
    return h.hexdigest()
```

### 2.4 `code/strategies.py`

```python
# code/strategies.py
from __future__ import annotations

import numpy as np

StrategyName = str   # "random" | "uncertainty" | "margin" | "entropy"


def select(strategy: StrategyName, *, probas: np.ndarray | None,
           k: int, rng: np.random.Generator) -> np.ndarray:
    """Return pool indices (dense) to label this round.

    `probas` is shape (n_pool, n_classes); `None` only for `random`.
    """
    if strategy == "random":
        n = probas.shape[0] if probas is not None else 0
        return rng.choice(n, size=k, replace=False)

    if probas is None:
        raise ValueError(f"strategy '{strategy}' requires probabilities")

    if strategy == "uncertainty":
        # least-confidence: 1 - max_class_prob, descending
        score = 1.0 - probas.max(axis=1)
    elif strategy == "margin":
        # smallest margin between top-2 classes
        part = np.partition(probas, -2, axis=1)[:, -2:]
        score = -(part[:, 1] - part[:, 0])
    elif strategy == "entropy":
        p = np.clip(probas, 1e-12, 1.0)
        score = -(p * np.log(p)).sum(axis=1)
    else:
        raise ValueError(f"unknown strategy: {strategy}")

    # argpartition is fine here; break ties by index for determinism
    order = np.lexsort((np.arange(len(score)), -score))
    return order[:k]
```

Determinism note. The `lexsort((index, -score))` tie-break makes the
selection deterministic when many pool items share the same score,
which does happen on small datasets. Random `argpartition` tie-breaks
make the run drift seed-over-seed and corrupt the reproducibility
check in §3.

### 2.5 `code/model.py`

```python
# code/model.py
from __future__ import annotations

from dataclasses import dataclass

import numpy as np
from sklearn.linear_model import LogisticRegression


@dataclass
class ModelCfg:
    C: float = 1.0
    max_iter: int = 2000
    class_weight: str | None = "balanced"
    random_state: int = 0


class Classifier:
    def __init__(self, cfg: ModelCfg) -> None:
        self.cfg = cfg
        self.clf: LogisticRegression | None = None

    def fit(self, x: np.ndarray, y: np.ndarray) -> None:
        self.clf = LogisticRegression(
            C=self.cfg.C, max_iter=self.cfg.max_iter,
            class_weight=self.cfg.class_weight,
            random_state=self.cfg.random_state,
            solver="lbfgs", n_jobs=1,
        )
        self.clf.fit(x, y)

    def predict_proba(self, x: np.ndarray) -> np.ndarray:
        assert self.clf is not None
        return self.clf.predict_proba(x)

    def predict(self, x: np.ndarray) -> np.ndarray:
        assert self.clf is not None
        return self.clf.predict(x)
```

The exercise uses logistic regression intentionally: it is cheap
enough to re-fit ten times per seed per strategy (4 strategies × 5
seeds × 10 rounds = 200 fits), it exposes well-calibrated
probabilities by default, and its determinism is reliable (`n_jobs=1`
pinned). A larger model is appropriate for production runs but not
required for the illustrative curve.

### 2.6 `code/run_al.py`

```python
# code/run_al.py
# A single (strategy, seed) run.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
from sklearn.metrics import f1_score, balanced_accuracy_score

from data import Split, deterministic_split, fingerprint
from model import Classifier, ModelCfg
from strategies import select


def run(split: Split, *, strategy: str, rounds: int, per_round: int,
        model_cfg: ModelCfg, seed: int) -> list[dict]:
    """Returns a list of per-round metric rows."""
    rng = np.random.default_rng(seed)
    x_lab, y_lab = split.x_labeled.copy(), split.y_labeled.copy()
    # Mutable pool-index set so remove is O(1)
    pool_idx = np.arange(len(split.x_pool))
    trace: list[dict] = []
    for round_i in range(rounds + 1):  # round 0 = seed-only
        clf = Classifier(model_cfg)
        clf.fit(x_lab, y_lab)
        y_test_hat = clf.predict(split.x_test)
        trace.append({
            "round": round_i,
            "labeled_count": int(len(x_lab)),
            "strategy": strategy,
            "seed": seed,
            "macro_f1": round(float(f1_score(split.y_test, y_test_hat,
                                             average="macro",
                                             zero_division=0)), 6),
            "balanced_accuracy": round(float(balanced_accuracy_score(
                split.y_test, y_test_hat)), 6),
        })
        if round_i == rounds:
            break
        probas = (clf.predict_proba(split.x_pool[pool_idx])
                  if strategy != "random" else None)
        picks_local = select(strategy, probas=probas,
                              k=per_round, rng=rng)
        global_picks = pool_idx[picks_local]
        x_lab = np.concatenate([x_lab, split.x_pool[global_picks]])
        y_lab = np.concatenate([y_lab, split.y_pool[global_picks]])  # oracle
        pool_idx = np.delete(pool_idx, picks_local)
    return trace


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--seed", type=int, required=True)
    ap.add_argument("--strategy", required=True)
    ap.add_argument("--rounds", type=int, required=True)
    ap.add_argument("--per-round", type=int, required=True)
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()
    # ... load x,y from a reproducible source; the fixture tests pin this.
    x, y = _load_dataset()
    split = deterministic_split(x, y, seed=args.seed,
                                 labeled_seed_size=200,
                                 pool_size=10_000, test_size=2_000)
    trace = run(split, strategy=args.strategy,
                 rounds=args.rounds, per_round=args.per_round,
                 model_cfg=ModelCfg(random_state=args.seed),
                 seed=args.seed)
    args.out.write_text("\n".join(json.dumps(r) for r in trace))
    print(json.dumps({"split_fingerprint": fingerprint(split)}))


def _load_dataset(): ...
```

### 2.7 `code/run_matrix.py`, `code/aggregate.py`, `code/plot_curve.py`

- `run_matrix.py` iterates `strategies × seeds`, invoking `run_al.run`
  for each combination and writing `runs/raw/<strategy>_<seed>.jsonl`.
  It parallelises at process level with `multiprocessing.Pool` with
  deterministic seeding (no shared RNG state).
- `aggregate.py` loads all raw traces, groups by `(strategy,
  labeled_count)`, and emits `run_metrics.parquet` with columns
  `strategy, labeled_count, macro_f1_mean, macro_f1_lo, macro_f1_hi,
  balanced_accuracy_mean, n_seeds`. CIs are 2.5/97.5 percentiles
  across seeds — bootstrapping over 5 seeds would under-represent
  variance so the direct percentile is the honest estimate here.
- `plot_curve.py` renders the curve with Matplotlib. One line per
  strategy, x = `labeled_count`, y = `macro_f1_mean`, shaded
  interval = `[lo, hi]`. Saves as SVG so the report is text-diffable.

### 2.8 `code/report.py`

```python
# code/report.py
# Emit report.md with the decision artefacts the program consumes.
from __future__ import annotations

import argparse
from pathlib import Path

import pandas as pd


def crossover_budget(df: pd.DataFrame, strategy_a: str, strategy_b: str,
                     margin: float) -> int | None:
    """Smallest labeled_count where mean metric of `a` exceeds `b` by `margin`.
    Returns None if the condition never holds across the schedule.
    """
    a = df[df.strategy == strategy_a].set_index("labeled_count")["macro_f1_mean"]
    b = df[df.strategy == strategy_b].set_index("labeled_count")["macro_f1_mean"]
    common = a.index.intersection(b.index).sort_values()
    for c in common:
        if a.loc[c] - b.loc[c] >= margin:
            return int(c)
    return None


def saturation_point(df: pd.DataFrame, strategy: str,
                     threshold_delta: float = 0.005) -> int | None:
    """The labeled_count after which successive gains fall below `threshold_delta`."""
    s = df[df.strategy == strategy].sort_values("labeled_count")
    deltas = s["macro_f1_mean"].diff()
    for c, d in zip(s["labeled_count"], deltas):
        if pd.isna(d):
            continue
        if d < threshold_delta:
            return int(c)
    return None


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--metrics", required=True, type=Path)
    ap.add_argument("--margin", type=float, required=True)
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()
    df = pd.read_parquet(args.metrics)
    cross_u = crossover_budget(df, "uncertainty", "random", args.margin)
    cross_e = crossover_budget(df, "entropy", "random", args.margin)
    sat_u = saturation_point(df, "uncertainty")
    md = [
        "# Active-learning budget-vs-quality report\n",
        "## Decision summary\n",
        f"- Crossover budget (`uncertainty` vs `random`, margin={args.margin}): "
        f"**{cross_u if cross_u is not None else 'not reached within schedule'}**",
        f"- Crossover budget (`entropy` vs `random`, margin={args.margin}): "
        f"**{cross_e if cross_e is not None else 'not reached within schedule'}**",
        f"- Saturation point (`uncertainty`): **{sat_u if sat_u else 'not reached'}**",
        "\n## Interpretation\n",
        "- Set the per-round budget below the saturation point; past it, "
        "spend goes to diminishing returns.",
        "- If the crossover budget is not reached within the labeling-round "
        "schedule, the program should default to `random` for this task.",
    ]
    args.out.write_text("\n".join(md) + "\n")


if __name__ == "__main__":
    main()
```

### 2.9 Tests

```python
# tests/test_strategies.py
import numpy as np
from code.strategies import select


def test_random_is_seed_deterministic():
    p = np.random.default_rng(0).random((100, 3))
    p /= p.sum(axis=1, keepdims=True)
    a = select("random", probas=p, k=10, rng=np.random.default_rng(42))
    b = select("random", probas=p, k=10, rng=np.random.default_rng(42))
    assert np.array_equal(a, b)


def test_uncertainty_picks_closest_to_uniform():
    p = np.array([[0.9, 0.05, 0.05], [0.4, 0.3, 0.3], [0.1, 0.1, 0.8]])
    idx = select("uncertainty", probas=p, k=1, rng=np.random.default_rng(0))
    assert idx[0] == 1   # row 1 has the lowest max-class confidence


def test_strategies_disagree():
    rng = np.random.default_rng(1)
    p = rng.dirichlet(alpha=np.ones(5), size=200)
    picks = {
        s: set(select(s, probas=p, k=20, rng=np.random.default_rng(0)))
        for s in ("uncertainty", "margin", "entropy")
    }
    # The three strategies should not pick identical sets on random probabilities;
    # if they do, something is wrong in the scoring code.
    assert picks["uncertainty"] != picks["margin"] or \
           picks["margin"] != picks["entropy"]
```

```python
# tests/test_driver_reproducibility.py
from code.run_al import run
from code.data import deterministic_split
from code.model import ModelCfg
import numpy as np


def test_same_seed_same_trace():
    rng = np.random.default_rng(0)
    x = rng.random((15_000, 20))
    y = rng.integers(0, 4, size=15_000)
    split = deterministic_split(x, y, seed=1,
                                 labeled_seed_size=200,
                                 pool_size=10_000, test_size=2_000)
    t1 = run(split, strategy="uncertainty", rounds=3, per_round=50,
             model_cfg=ModelCfg(random_state=1), seed=1)
    t2 = run(split, strategy="uncertainty", rounds=3, per_round=50,
             model_cfg=ModelCfg(random_state=1), seed=1)
    assert t1 == t2
```

### 2.10 `Makefile`

```makefile
.PHONY: all run aggregate plot report test clean

all: run aggregate plot report

run:
	python code/run_matrix.py --config config.yaml --out-dir runs/raw

aggregate:
	python code/aggregate.py --raw-dir runs/raw --out runs/run_metrics.parquet

plot:
	python code/plot_curve.py --metrics runs/run_metrics.parquet \
		--out runs/budget_vs_quality.svg

report:
	python code/report.py --metrics runs/run_metrics.parquet \
		--margin 0.01 --out runs/report.md

test:
	pytest -q

clean:
	rm -rf runs/raw/*.jsonl runs/run_metrics.parquet \
		runs/budget_vs_quality.svg runs/report.md
```

## 3. Validation steps

1. **Deterministic re-run.** `make clean && make run aggregate` twice
   on the same host; `sha256sum runs/run_metrics.parquet` matches
   bit-for-bit.
2. **Cross-platform run_metrics.** On a second runner (different
   CPU, same Python + `requirements.lock`), `run_metrics.parquet`
   matches to the configured decimal places.
3. **Strategies produce different traces.** For the same seed,
   `uncertainty`, `margin`, `entropy`, and `random` traces differ
   starting from round 1. Identical traces mean the probabilities
   collapsed (model not fitted) or the tie-break is buggy.
4. **Macro-F1 is used, not accuracy.** `pandas.read_parquet` on
   `run_metrics.parquet` shows `macro_f1_mean` as the primary
   column; accuracy is secondary. A report that quotes accuracy
   as the headline number fails review.
5. **Crossover decision is honest.** When the crossover budget
   does not exist within the schedule, `report.md` says "not
   reached" — it does not stretch the schedule to manufacture
   one. Rerun with the margin set to 0 and confirm the crossover
   moves to a smaller budget (sanity).
6. **Class representation is tracked.** A side report (produced
   by aggregation) shows the labeled-class distribution per
   strategy at each round. `uncertainty` should not drive any
   class to zero representation; if it does, that's a finding,
   not a feature.
7. **Canary pool item.** Inject 20 adversarial pool items that
   look maximally uncertain (near-uniform `predict_proba`) but
   carry an incorrect oracle label. Run `uncertainty` and
   confirm the canary-poisoning signal appears as a macro-F1 dip
   at the round the canaries were selected. The purpose of this
   check is to make sure the pipeline surfaces the dip, not to
   claim AL is safe against poisoning.
8. **No shared RNG state across workers.** `test_driver_reproducibility`
   passes with and without multiprocessing enabled; multi-process
   runs produce the same aggregated metrics.
9. **SVG curve has strategy legend, budget on x-axis, macro-F1 on
   y-axis, CI shading, and saturation + crossover markers.**
10. **Report cites the crossover and saturation points.** Not an
    essay; concrete numbers, with "not reached" as a valid value.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 20/28 with zero zeros on
critical rows.

| #  | Dimension                                                                                                              | Critical? |
|----|------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | ≥ 2 strategies (`random` baseline + an uncertainty variant); `entropy` recommended                                     | Yes       |
| 2  | Budget-matched comparison: x-axis is total labeled items, not round index                                              | Yes       |
| 3  | Multiple seeds (≥ 5); CIs reported on the curve                                                                        | Yes       |
| 4  | Primary metric is macro-F1 or balanced accuracy (not raw accuracy)                                                      | Yes       |
| 5  | Deterministic driver: same seed + same strategy → byte-identical trace                                                  | Yes       |
| 6  | Tie-break in strategy selection is deterministic (lexicographic by index)                                               | Yes       |
| 7  | Oracle labels are held back from the model; AL "queries" them only when the pool item is picked                         | Yes       |
| 8  | `report.md` declares crossover budget and saturation point; "not reached" is a valid value                              | Yes       |
| 9  | Class-representation audit per strategy per round is produced                                                          | Yes       |
| 10 | Canary-pool experiment demonstrates AL's poisoning sensitivity (surfacing, not claiming immunity)                       | No        |
| 11 | Governance anchors (NIST AI RMF MEASURE 2.3 / 2.11, MITRE ATLAS AML.T0020, OWASP ML02) cited                             | No        |
| 12 | `run_metrics.parquet` reproduces cross-platform within configured precision                                             | Yes       |
| 13 | Tests pin reproducibility; `pytest -q` passes                                                                          | Yes       |
| 14 | Curve plot includes legend, CI shading, and crossover/saturation markers                                               | No        |

## 5. Common mistakes

- **Reporting raw accuracy.** On an imbalanced dataset, uncertainty
  sampling can hollow out a class and raise accuracy while the
  model gets worse at that class. Macro-F1 or balanced accuracy
  catches it.
- **Running a single seed.** A single seed cannot distinguish
  "uncertainty beat random" from "uncertainty was lucky". CIs
  across seeds are the answer.
- **Comparing at round indices, not budgets.** If `random` picks
  120 items per round and `uncertainty` picks 100, round 5 is not
  a fair comparison. The x-axis must be `labeled_count`.
- **Non-deterministic tie-break in selection.** Many pool items
  share identical uncertainty scores on small datasets; randomised
  `argpartition` ordering drifts the trace. Lexicographic
  tie-break by index fixes it.
- **Hidden data leak via vectorisation choices.** Fit the vectoriser
  on `x_labeled` or on `x_labeled ∪ x_pool` *without the test set*.
  Fitting on the test set leaks structure and inflates the curve.
- **Not auditing class representation.** An AL run that drives a
  class to < 1% of the labeled set is a fairness regression even if
  macro-F1 stays flat. Track and report per-class counts.
- **"We beat random by 0.2 F1 at round 3" without CIs.** 0.2 F1
  across seeds has a large interval on a 1200-item schedule; the
  "win" may be within noise. The curve with CIs is the honest
  artefact.
- **Hardcoding dataset path.** Make the dataset loader respect a
  fingerprint check so the report proves which dataset version
  was used.
- **Letting the model's own `random_state` drift seed-to-seed.**
  `random_state=args.seed` ties the model fit to the experiment
  seed; mixing a global RNG in kills reproducibility.
- **Reporting the crossover budget as absolute with no margin.**
  Any strategy beats any other by 0.001 at some round if you look
  hard enough. The margin (`crossover_margin: 0.01`) is the
  honest control.

## 6. References

- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE 2.11:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 anchors the budget-matched metric comparison; MEASURE 2.11
  anchors the class-representation audit that an AL policy can
  degrade.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  AL policies that always-pick-maximally-uncertain items are
  attractive targets for pool poisoning; the canary experiment in
  §3.7 surfaces the signature.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing anchor.
- **Settles, B. (2009) "Active Learning Literature Survey."**
  University of Wisconsin–Madison Technical Report 1648. The
  canonical taxonomy of AL strategies (least-confidence, margin,
  entropy, query-by-committee) the implementation in §2.4 maps to.
- **Lewis, D. & Gale, W. (1994) "A Sequential Algorithm for
  Training Text Classifiers."** SIGIR 1994. The least-confidence
  (uncertainty) baseline the exercise implements.
- **Scheffer, T., Decomain, C., & Wrobel, S. (2001) "Active Hidden
  Markov Models for Information Extraction."** Introduces margin
  sampling.
- **scikit-learn `LogisticRegression`:**
  <https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html>.
  The model used for the illustrative experiment; probabilities are
  well-calibrated out of the box, which is what the AL scorers need.
- **Local anchors:** `mod-107` ex-01 (the labeling policy whose
  per-round budget this exercise informs); `mod-107` ex-03 (the
  IAA tool that gates each round's labels before they enter the
  labeled set); `mod-111` (quality-regression monitor that
  consumes the curve metadata to detect AL-driven drift).
