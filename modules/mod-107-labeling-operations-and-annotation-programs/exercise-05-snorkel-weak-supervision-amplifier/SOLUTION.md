# Exercise 05 — Snorkel Weak-Supervision Amplifier (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-05-snorkel-weak-supervision-amplifier.md`) is not
> visible from the solutions repo at generation time. This
> solution is grounded in the exercise title, the module README,
> and the official sources cited in §6 (Ratner et al. "Snorkel"
> 2017 / 2020 and the Snorkel project documentation). Any
> divergence from the exercise prompt once it is pinned should
> re-ground against those sources.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm label-schema choices
> and the hand-off format for the amplified labels. -->

## 1. Solution overview

This is an **implementation** exercise. The learner builds a
**Snorkel weak-supervision pipeline** that amplifies a small
human-labeled seed into a large probabilistically-labeled
training set. The deliverable is a label matrix, a trained
`LabelModel`, probabilistic labels over the unlabeled pool, and
an `amplification_report.yaml` that records the LF-analysis
numbers (coverage, overlap, conflict, polarity, empirical
accuracy) that make weak-supervision outputs reviewable.

Snorkel's contract (per Ratner et al.):

- A set of **labeling functions (LFs)**, each noisy and limited-
  coverage, votes on each data point.
- A **label model** learns per-LF accuracies and correlations
  from the **pattern of agreements and disagreements** — not
  from the gold labels. A small gold development set (dev) is
  used only to evaluate.
- The label model outputs **probabilistic labels** over the
  unlabeled pool. A downstream classifier ("end model") is
  trained on these soft labels.

Deliverables:

- `code/lfs.py` — the labeling functions. One function per LF,
  each cites the heuristic (regex, keyword list, model
  prediction) and declares an explicit abstain contract.
- `code/label_matrix.py` — applies LFs to the unlabeled pool;
  writes `L_train.npy`.
- `code/label_model.py` — fits Snorkel's `LabelModel` on
  `L_train` and emits `prob_labels.parquet`.
- `code/analysis.py` — LF analysis (coverage, overlaps,
  conflicts, polarity); against a small `dev` set, per-LF
  empirical accuracy.
- `code/end_model.py` — a cheap classifier trained on soft
  labels; reports held-out accuracy and macro-F1 as the
  downstream check.
- `code/report.py` — emits `amplification_report.yaml`.
- `tests/test_lfs.py` — contract tests on every LF (returns one
  of `{-1, 0, 1, ...}`, no side effects, deterministic).
- `outputs/` — `L_train.npy`, `prob_labels.parquet`,
  `lf_analysis.csv`, `amplification_report.yaml`, end-model
  metrics.

Governance framing. Weak supervision amplifies a small signal
into a large signal; failure modes include systematic bias from
a popular LF, correlated LFs that collectively mislead the
label model, and the illusion of coverage when most LFs abstain.

- OWASP Machine Learning Security Top 10 anchor: **ML02:2023
  Data Poisoning Attack** — a rogue LF (or a cohort of
  correlated LFs reflecting the same bias) poisons training.
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data**
  (downstream harm), **AML.T0043 Craft Adversarial Data**
  (attacker crafts items that satisfy a biased LF to pull
  training signal).
- NIST AI RMF controls: **MEASURE 2.3** (performance measured
  as intended) — LF-analysis numbers are the measurement
  surface; **MEASURE 2.11** (fairness and bias) — per-slice
  LF accuracy is what distinguishes amplification from
  bias-magnification.

Cite the framework URLs rather than paraphrasing the control
text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── code/
│   ├── lfs.py
│   ├── label_matrix.py
│   ├── label_model.py
│   ├── analysis.py
│   ├── end_model.py
│   └── report.py
├── configs/
│   └── experiment.yaml
├── tests/
│   └── test_lfs.py
└── outputs/
    ├── L_train.npy
    ├── prob_labels.parquet
    ├── lf_analysis.csv
    ├── end_model_metrics.json
    └── amplification_report.yaml
```

### 2.2 `configs/experiment.yaml`

```yaml
experiment_id: mod-107-ex-05-run-1
seed: 20261117
task:
  name: topic_classification
  classes: ["policy", "technical", "product", "other"]
  class_index:
    policy: 0
    technical: 1
    product: 2
    other: 3
  abstain_value: -1
dataset:
  unlabeled_pool_path: data/pool.parquet
  dev_path:            data/dev.parquet     # small gold dev set
  test_path:           data/test.parquet    # held-out evaluation
  text_field: text
  label_field: topic
label_model:
  cardinality: 4
  epochs: 500
  lr: 0.01
  l2: 0.0
end_model:
  kind: logistic_regression_bow
  max_features: 20000
  C: 1.0
```

### 2.3 `code/lfs.py`

Each LF is intentionally narrow and returns one of
`{abstain, class_index}`. The point of writing many narrow LFs
is to let the label model see the pattern of agreements across
them. A single wide LF is a hand-written classifier; it is not
a weak-supervision workflow.

```python
# code/lfs.py
# Labeling functions for topic classification.
# Each LF takes one record and returns either an integer class
# index or `ABSTAIN` (-1). LFs are pure, deterministic, and side-
# effect-free; the test suite proves it.
from __future__ import annotations

import re
from typing import Callable

ABSTAIN = -1
POLICY, TECHNICAL, PRODUCT, OTHER = 0, 1, 2, 3


# ---- Regex / keyword LFs --------------------------------------

_POLICY_RE = re.compile(r"\b(policy|regulation|compliance|gdpr|statute)\b",
                        re.IGNORECASE)
_TECH_RE = re.compile(r"\b(kubernetes|python|latency|throughput|p99|p95)\b",
                      re.IGNORECASE)
_PRODUCT_RE = re.compile(r"\b(feature|roadmap|customer|ux|user[\s-]facing)\b",
                         re.IGNORECASE)


def lf_policy_regex(x: dict) -> int:
    return POLICY if _POLICY_RE.search(x["text"]) else ABSTAIN


def lf_technical_regex(x: dict) -> int:
    return TECHNICAL if _TECH_RE.search(x["text"]) else ABSTAIN


def lf_product_regex(x: dict) -> int:
    return PRODUCT if _PRODUCT_RE.search(x["text"]) else ABSTAIN


# ---- Short-text and long-text heuristics ----------------------

def lf_short_boilerplate_other(x: dict) -> int:
    return OTHER if len(x["text"].split()) < 15 else ABSTAIN


def lf_contains_code_block(x: dict) -> int:
    # A document with a triple-backtick code block is probably technical.
    return TECHNICAL if "```" in x["text"] else ABSTAIN


# ---- Weak-model LF --------------------------------------------

def make_lf_from_prelabel(predicted_class: Callable[[str], int],
                           confidence_threshold: float,
                           confidence_scorer: Callable[[str], float]) -> Callable:
    """Wrap an external classifier as an LF. If the classifier's
    confidence is above `confidence_threshold`, return its class;
    otherwise abstain. Separate `predicted_class` and
    `confidence_scorer` keep the wrapper framework-agnostic.
    """
    def lf(x: dict) -> int:
        score = confidence_scorer(x["text"])
        if score < confidence_threshold:
            return ABSTAIN
        return predicted_class(x["text"])
    lf.__name__ = "lf_prelabel_wrapper"
    return lf


# ---- Canonical LF list ----------------------------------------

LFS: list[Callable[[dict], int]] = [
    lf_policy_regex,
    lf_technical_regex,
    lf_product_regex,
    lf_short_boilerplate_other,
    lf_contains_code_block,
]
```

Design notes defensible at review:

- **Each LF is pure and deterministic.** No `requests.get()`, no
  random, no stateful counters. If an LF needs an external model,
  it must be deterministic under its inputs; the wrapper in
  `make_lf_from_prelabel` requires a passed-in callable so the
  LF code itself stays pure.
- **`ABSTAIN` is `-1`, matching Snorkel's convention.** A
  reviewer should not have to think about label-value
  collisions with real class indices; `-1` is the Snorkel
  convention across the library.
- **Keyword regexes are narrow.** A regex that matches "user"
  anywhere is unreviewable. The regex vocabulary is a committed
  artifact; adding a term is a diff-reviewable change.
- **The weak-model LF wrapper separates class from confidence.**
  A single-model LF can miscalibrate badly on new slices; the
  confidence gate lets reviewers tune it without rewriting the
  LF.

### 2.4 `code/label_matrix.py`

```python
# code/label_matrix.py
# Apply LFs to the unlabeled pool; emit L_train as a numpy array.
from __future__ import annotations

import argparse
from pathlib import Path

import numpy as np
import pandas as pd

from lfs import LFS


def apply_lfs(records: list[dict]) -> np.ndarray:
    mat = np.full((len(records), len(LFS)), fill_value=-1, dtype=np.int8)
    for i, rec in enumerate(records):
        for j, lf in enumerate(LFS):
            mat[i, j] = lf(rec)
    return mat


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pool", required=True, type=Path)
    ap.add_argument("--out",  required=True, type=Path)
    args = ap.parse_args()
    df = pd.read_parquet(args.pool)
    records = df.to_dict(orient="records")
    L = apply_lfs(records)
    np.save(args.out, L)
```

### 2.5 `code/analysis.py`

LF analysis is the review surface for weak supervision. Coverage
and conflict numbers that look reasonable in aggregate can hide
a single LF that dominates a slice.

```python
# code/analysis.py
# Coverage, overlap, conflict, polarity per LF. Against a dev set,
# empirical accuracy per LF.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd

from label_matrix import apply_lfs
from lfs import LFS


def lf_analysis(L: np.ndarray) -> pd.DataFrame:
    n, m = L.shape
    rows = []
    for j in range(m):
        votes = L[:, j]
        active = votes != -1
        coverage = float(active.mean())
        polarity = sorted(set(int(v) for v in votes[active])) if active.any() else []
        # Overlap: active AND any other LF active on same item.
        other_active = (L != -1)
        other_active[:, j] = False
        overlap = float((active & other_active.any(axis=1)).mean())
        # Conflict: active AND any other LF disagrees on same item.
        if active.any():
            conflict_mask = np.zeros(n, dtype=bool)
            for k in range(m):
                if k == j:
                    continue
                other = L[:, k]
                conflict_mask |= active & (other != -1) & (other != votes)
            conflict = float(conflict_mask.mean())
        else:
            conflict = 0.0
        rows.append({
            "lf": LFS[j].__name__,
            "coverage": coverage,
            "overlap":  overlap,
            "conflict": conflict,
            "polarity": polarity,
        })
    return pd.DataFrame(rows)


def empirical_accuracy(L_dev: np.ndarray, y_dev: np.ndarray) -> pd.DataFrame:
    rows = []
    for j in range(L_dev.shape[1]):
        votes = L_dev[:, j]
        active = votes != -1
        if not active.any():
            rows.append({"lf": LFS[j].__name__, "empirical_accuracy": float("nan"),
                          "support": 0})
            continue
        acc = float((votes[active] == y_dev[active]).mean())
        rows.append({"lf": LFS[j].__name__, "empirical_accuracy": acc,
                      "support": int(active.sum())})
    return pd.DataFrame(rows)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--L-train", required=True, type=Path)
    ap.add_argument("--dev",      required=True, type=Path,
                    help="Parquet with text + label (gold) columns")
    ap.add_argument("--class-index-json", required=True, type=Path)
    ap.add_argument("--out",      required=True, type=Path)
    args = ap.parse_args()
    L = np.load(args.L_train)
    stats = lf_analysis(L)
    dev = pd.read_parquet(args.dev)
    class_index = json.loads(args.class_index_json.read_text())
    L_dev = apply_lfs(dev.to_dict(orient="records"))
    y_dev = dev["label"].map(class_index).astype(np.int64).to_numpy()
    acc = empirical_accuracy(L_dev, y_dev)
    merged = stats.merge(acc, on="lf", how="left")
    merged.to_csv(args.out, index=False)
```

### 2.6 `code/label_model.py`

```python
# code/label_model.py
# Train Snorkel's LabelModel on L_train. Emit probabilistic
# labels over the pool.
from __future__ import annotations

import argparse
from pathlib import Path

import numpy as np
import pandas as pd
from snorkel.labeling.model import LabelModel  # official Snorkel SDK


def train_label_model(L_train: np.ndarray, *, cardinality: int,
                       epochs: int, lr: float, l2: float,
                       seed: int) -> LabelModel:
    lm = LabelModel(cardinality=cardinality, verbose=False)
    lm.fit(L_train=L_train, n_epochs=epochs, lr=lr, l2=l2, seed=seed)
    return lm


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--L-train", required=True, type=Path)
    ap.add_argument("--pool",    required=True, type=Path)
    ap.add_argument("--cardinality", type=int, required=True)
    ap.add_argument("--epochs",  type=int, default=500)
    ap.add_argument("--lr",      type=float, default=0.01)
    ap.add_argument("--l2",      type=float, default=0.0)
    ap.add_argument("--seed",    type=int, default=20261117)
    ap.add_argument("--out",     required=True, type=Path)
    args = ap.parse_args()
    L = np.load(args.L_train)
    lm = train_label_model(L, cardinality=args.cardinality,
                            epochs=args.epochs, lr=args.lr, l2=args.l2,
                            seed=args.seed)
    probs = lm.predict_proba(L=L)
    pool = pd.read_parquet(args.pool)
    out = pool.copy()
    for k in range(probs.shape[1]):
        out[f"p_class_{k}"] = probs[:, k]
    out.to_parquet(args.out, index=False)
```

Design notes:

- **Label model trains on `L_train` only.** It does not see gold
  labels during fit; the development set is for evaluation and
  for `LabelModel.score()` after training. A label model that
  was fit against gold is a supervised classifier, not a weak-
  supervision model.
- **Seed is fixed.** Snorkel's `LabelModel` exposes a `seed` to
  `fit()`; pin it so the amplification is reproducible.
- **Cardinality must match the task.** A mismatch silently
  corrupts the probability distribution; `cardinality` in the
  config is the authoritative source and the end model reads
  the same value.

### 2.7 `code/end_model.py`

```python
# code/end_model.py
# A cheap downstream classifier trained on Snorkel's probabilistic
# labels. Reports held-out accuracy and macro-F1 against the test set.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline


def _hard_labels(probs: np.ndarray) -> np.ndarray:
    return probs.argmax(axis=1)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--prob-labels", required=True, type=Path)
    ap.add_argument("--test",        required=True, type=Path)
    ap.add_argument("--class-index-json", required=True, type=Path)
    ap.add_argument("--max-features", type=int, default=20000)
    ap.add_argument("--C",            type=float, default=1.0)
    ap.add_argument("--seed",         type=int, default=20261117)
    ap.add_argument("--filter-abstain-threshold", type=float, default=None,
                    help="If set, drop rows whose max class prob is below this.")
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()

    train_df = pd.read_parquet(args.prob_labels)
    prob_cols = [c for c in train_df.columns if c.startswith("p_class_")]
    probs = train_df[prob_cols].to_numpy()
    if args.filter_abstain_threshold is not None:
        keep = probs.max(axis=1) >= args.filter_abstain_threshold
        train_df = train_df.loc[keep].reset_index(drop=True)
        probs = probs[keep]
    y_train = _hard_labels(probs)

    pipe = Pipeline([
        ("tfidf", TfidfVectorizer(max_features=args.max_features,
                                   ngram_range=(1, 2),
                                   sublinear_tf=True)),
        ("lr", LogisticRegression(C=args.C, max_iter=500,
                                   random_state=args.seed, n_jobs=1)),
    ])
    pipe.fit(train_df["text"].tolist(), y_train)

    test_df = pd.read_parquet(args.test)
    class_index = json.loads(args.class_index_json.read_text())
    y_test = test_df["label"].map(class_index).astype(np.int64).to_numpy()
    pred = pipe.predict(test_df["text"].tolist())
    acc = float((pred == y_test).mean())
    # Macro-F1
    labs = sorted(set(y_test.tolist()) | set(pred.tolist()))
    f1s = []
    for c in labs:
        tp = int(((pred == c) & (y_test == c)).sum())
        fp = int(((pred == c) & (y_test != c)).sum())
        fn = int(((pred != c) & (y_test == c)).sum())
        if tp == 0:
            f1s.append(0.0)
            continue
        p = tp / (tp + fp)
        r = tp / (tp + fn)
        f1s.append(2 * p * r / (p + r))
    macro_f1 = float(np.mean(f1s)) if f1s else 0.0
    args.out.write_text(json.dumps({"accuracy": acc, "macro_f1": macro_f1},
                                     indent=2, sort_keys=True))
```

### 2.8 `code/report.py`

```python
# code/report.py
# Emit amplification_report.yaml.
from __future__ import annotations

import argparse
import hashlib
import json
import socket
import subprocess
from datetime import datetime, timezone
from pathlib import Path

import pandas as pd
import yaml


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--experiment-id", required=True)
    ap.add_argument("--operator", required=True)
    ap.add_argument("--config", required=True, type=Path)
    ap.add_argument("--lf-analysis", required=True, type=Path)
    ap.add_argument("--prob-labels", required=True, type=Path)
    ap.add_argument("--end-metrics", required=True, type=Path)
    ap.add_argument("--run-start-utc", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    analysis = pd.read_csv(args.lf_analysis).to_dict(orient="records")
    prob = pd.read_parquet(args.prob_labels)
    prob_cols = [c for c in prob.columns if c.startswith("p_class_")]

    import numpy as np
    maxp = prob[prob_cols].to_numpy().max(axis=1)

    git_sha = subprocess.check_output(["git", "rev-parse", "HEAD"]).decode().strip()
    report = {
        "experiment_id": args.experiment_id,
        "toolchain": {
            "name": "mod-107-exercise-05-snorkel",
            "git_sha": git_sha,
            "config_sha256": sha256_file(args.config),
            "prob_labels_sha256": sha256_file(args.prob_labels),
            "lf_analysis_sha256": sha256_file(args.lf_analysis),
        },
        "operator": args.operator,
        "run_start_utc": args.run_start_utc,
        "run_end_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "compute_shape": {"host": socket.gethostname()},
        "lf_analysis": analysis,
        "confidence_distribution": {
            "p_ge_0.5":  float((maxp >= 0.5).mean()),
            "p_ge_0.7":  float((maxp >= 0.7).mean()),
            "p_ge_0.9":  float((maxp >= 0.9).mean()),
        },
        "end_model_metrics": json.loads(args.end_metrics.read_text()),
        "peer_review": {
            "status": "pending_review",
            "reviewer": None,
            "signed_at_utc": None,
        },
    }
    args.output.write_text(yaml.safe_dump(report, sort_keys=False))
    print(json.dumps({"report_sha256": sha256_file(args.output)}, indent=2))
```

### 2.9 `tests/test_lfs.py`

```python
# tests/test_lfs.py
# Contract tests for every LF: pure, deterministic, returns either
# an integer class index in [0, cardinality) or the abstain sentinel.
from __future__ import annotations

import pytest

from code.lfs import ABSTAIN, LFS


@pytest.mark.parametrize("lf", LFS)
def test_lf_is_pure(lf) -> None:
    # Two calls on the same input return the same value.
    rec = {"text": "GDPR compliance review of the roadmap feature"}
    assert lf(rec) == lf(rec)


@pytest.mark.parametrize("lf", LFS)
def test_lf_returns_valid_label(lf) -> None:
    rec = {"text": "some sample text"}
    v = lf(rec)
    assert v == ABSTAIN or (isinstance(v, int) and v >= 0)
```

### 2.10 `Makefile`

```makefile
.PHONY: all test matrix analysis label_model end_model report clean

all: test matrix analysis label_model end_model report

test:
	pytest -q tests/

matrix:
	python code/label_matrix.py \
	  --pool data/pool.parquet --out outputs/L_train.npy

analysis:
	python code/analysis.py \
	  --L-train outputs/L_train.npy \
	  --dev data/dev.parquet \
	  --class-index-json configs/class_index.json \
	  --out outputs/lf_analysis.csv

label_model:
	python code/label_model.py \
	  --L-train outputs/L_train.npy \
	  --pool data/pool.parquet \
	  --cardinality 4 --epochs 500 --lr 0.01 --l2 0.0 \
	  --out outputs/prob_labels.parquet

end_model:
	python code/end_model.py \
	  --prob-labels outputs/prob_labels.parquet \
	  --test data/test.parquet \
	  --class-index-json configs/class_index.json \
	  --out outputs/end_model_metrics.json

report:
	python code/report.py \
	  --experiment-id mod-107-ex-05-run-1 \
	  --operator $$(git config user.email) \
	  --config configs/experiment.yaml \
	  --lf-analysis outputs/lf_analysis.csv \
	  --prob-labels outputs/prob_labels.parquet \
	  --end-metrics outputs/end_model_metrics.json \
	  --run-start-utc $$(date -u +%Y-%m-%dT%H:%M:%SZ) \
	  --output outputs/amplification_report.yaml

clean:
	rm -rf outputs/
```

### 2.11 Interpreting the report

Four signals a reviewer reads before signing off:

1. **Coverage sum vs effective coverage.** Summed coverage can
   exceed 1.0 (LFs overlap); effective coverage is the fraction
   of pool items where at least one LF is active. A pipeline
   with effective coverage below ~50% is not amplifying much;
   either add LFs or accept a smaller training set.
2. **Per-LF empirical accuracy vs support.** An LF with 0.95
   accuracy on 10 dev items is noise; the same accuracy on 1000
   items is signal. The analysis table reports both.
3. **Conflict rate.** High conflict (>30%) with low empirical
   accuracy on the top conflicting LF is a sign that the LF is
   systematically biased. The label model can correct for
   accuracy, but it can't correct for a confidently wrong LF
   whose errors are correlated with content.
4. **End-model gain vs no-LF baseline.** The end-model accuracy
   should exceed the accuracy of a model trained on the dev
   gold labels alone. If it doesn't, weak supervision is not
   paying for itself on this task.

## 3. Validation steps

1. **LF contract tests pass.** `pytest -q tests/` exits 0.
   Any failure is a regression in the LF contract.
2. **`L_train` is deterministic.** `python code/label_matrix.py`
   twice on the same pool → byte-identical `.npy`.
3. **Label model is seeded.** `grep "seed=" code/label_model.py`
   returns the config's seed; two runs produce identical
   `prob_labels.parquet`.
4. **LF analysis has one row per LF.** `wc -l
   outputs/lf_analysis.csv` equals `len(LFS) + 1` (header).
5. **No LF operates on non-pool inputs during analysis.**
   `analysis.py` reads `pool` and `dev` only.
6. **Confidence distribution reported.** The report's
   `confidence_distribution` keys cover 0.5 / 0.7 / 0.9 floors.
   A pipeline whose p ≥ 0.9 bucket is empty is not confident
   anywhere; this must be visible, not hidden.
7. **End-model gain measured.** `end_model.py` reports both
   accuracy and macro-F1 on the test set; compare against a
   dev-only baseline (train the end model on dev labels alone
   for the comparison).
8. **Hand-off to downstream training uses `prob_labels.parquet`
   OR hard argmax.** The pipeline documents which was chosen;
   using argmax is defensible but discards calibration signal.
9. **`peer_review.status` defaults to `pending_review`.**
10. **No gold labels in `L_train`.** The gold `dev` set is never
    concatenated with the pool; a leak here invalidates both
    the LabelModel fit and the end-model evaluation.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero
zeros on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Every LF is pure, deterministic, and uses `-1` as the abstain sentinel                                              | Yes       |
| 2  | At least three LFs; at least one keyword-based, at least one length/structure-based, and one model-wrapper          | Yes       |
| 3  | Label model trained on `L_train` only (no gold labels in fit)                                                       | Yes       |
| 4  | LF analysis emits coverage, overlap, conflict, polarity, empirical accuracy, support per LF                         | Yes       |
| 5  | Confidence distribution over pool items reported in the report                                                      | Yes       |
| 6  | End-model gain over dev-only baseline reported                                                                       | Yes       |
| 7  | Seeded Snorkel `LabelModel.fit` with a documented seed                                                               | Yes       |
| 8  | `cardinality` sourced from a single config, not duplicated across files                                             | Yes       |
| 9  | Dev / pool / test splits do not overlap                                                                             | Yes       |
| 10 | `prob_labels.parquet` keeps per-class probabilities, not just argmax                                                 | No        |
| 11 | Hand-off format to `mod-108` / downstream training documented                                                        | No        |
| 12 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020 / T0043, NIST AI RMF MEASURE 2.3 / 2.11) cited        | No        |
| 13 | `peer_review.status` defaults to `pending_review`                                                                    | Yes       |
| 14 | LF analysis is signed by a peer (`mod-105` ex-05 convention) before the amplified set enters training                | Yes       |

## 5. Common mistakes

- **Writing one wide LF.** A single LF that classifies every
  item without abstaining is a hand-coded classifier; the
  label model has nothing to learn from.
- **Fitting the LabelModel on gold labels.** That is supervised
  learning. Snorkel's contract is that the fit sees only the
  LFs' votes.
- **Not reporting empirical accuracy with support.** 95%
  accuracy on 10 items is marketing; 85% on 1000 is a working
  LF.
- **Hiding abstain rate.** If half the pool has every LF
  abstaining, the amplified set is half the pool; silently
  dropping those items without reporting the rate is a bug.
- **Shipping probabilistic labels as argmax without a floor.**
  Low-confidence items (max p < 0.5) often add noise to the end
  model; filter them or weight by confidence.
- **Not evaluating the end model on a held-out test.** The whole
  point of the amplification is a better downstream model;
  measure it against a test set the LabelModel never saw.
- **Correlated LFs treated as independent signal.** Two
  near-identical LFs (same keyword list, different regex)
  double-count that signal. Check conflict / overlap, and
  consider using Snorkel's dependency-aware label model if
  correlations are known.
- **Missing a seed on `LabelModel.fit`.** Snorkel fits a
  logistic model under the hood; without a seed the probs
  shift run-to-run and the amplified set is non-deterministic.
- **Skipping the dev-only baseline.** Without a baseline you
  cannot show weak supervision is paying for itself.

## 6. References

- **Ratner, A. et al. (2017). "Snorkel: Rapid Training Data
  Creation with Weak Supervision." _VLDB 2017_.**
  <https://www.vldb.org/pvldb/vol11/p269-ratner.pdf>. Primary
  source for the labeling-function / label-model /
  end-model pipeline and the agreement/disagreement signal the
  label model fits.
- **Snorkel project documentation:**
  <https://www.snorkel.org/> and
  <https://snorkel.readthedocs.io/>. Primary source for the
  `LabelModel` API, LF analysis helpers, and the `-1` abstain
  convention.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Governance anchor — a biased LF is a poisoning channel.
- **MITRE ATLAS — AML.T0020 Poison Training Data,
  AML.T0043 Craft Adversarial Data:**
  <https://atlas.mitre.org/>. Adversary-facing anchors.
- **NIST AI Risk Management Framework — MEASURE 2.3 (performance
  as intended), MEASURE 2.11 (fairness and bias):**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
  Governance anchors for the LF-analysis measurement surface
  and per-slice accuracy.
- **scikit-learn user guide — `LogisticRegression`,
  `TfidfVectorizer`, `Pipeline`:**
  <https://scikit-learn.org/stable/user_guide.html>. Primary
  source for the end-model choice.
- **Local anchors:** Exercise 01 (`task_record.schema.json`
  carries the LF output and gold labels through the same
  pipeline); Exercise 03 (IAA on the gold dev set); Exercise
  04 (active learning as the alternative labeling-cost
  strategy); `mod-108` (consumes the probabilistic labels as
  synthetic-labeled training data); `mod-111` (trends LF
  analysis numbers across weekly reruns).
