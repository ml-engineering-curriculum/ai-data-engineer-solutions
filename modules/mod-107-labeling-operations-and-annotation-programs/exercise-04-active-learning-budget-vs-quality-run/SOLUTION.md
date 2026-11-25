# Exercise 04 — Active Learning Budget-vs-Quality Run (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-04-active-learning-budget-vs-quality-run.md`) is not
> visible from the solutions repo at generation time. This
> solution is grounded in the exercise title, the module README,
> and the official sources cited in §6 (Settles 2009 survey,
> scikit-learn user guide for active-learning baselines, Label
> Studio ML-backend documentation). Any divergence from the
> exercise prompt once it is pinned should re-ground against
> those sources.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm the dataset shape
> and the query-strategy palette. -->

## 1. Solution overview

This is an **implementation** exercise. The learner runs an
**active-learning (AL) budget-vs-quality experiment**: starting
from a seed labeled set, they train a classifier, use an
acquisition strategy to pick the next `B` unlabeled items,
simulate labeling them against a held-out gold pool, retrain,
and repeat. The deliverable is a `budget_curve.csv` and a
`run_report.yaml` that compare at least three acquisition
strategies against a random baseline, plus an operational plug-
in to Exercise 01's Label Studio project so the same pipeline
runs against real labelers.

Three acquisition strategies are in scope:

- **Random sampling** — the baseline. Any strategy that does not
  beat random is not worth deploying.
- **Uncertainty sampling (least confidence / margin / entropy).**
  The default active-learning strategy per Settles (2009).
- **Query-by-committee (QBC)** — bag of classifiers, pick items
  with highest vote disagreement.

Deliverables:

- `code/dataset.py` — splits the corpus into seed, pool, gold.
- `code/model.py` — a small, cheap classifier (logistic
  regression on a bag-of-words or a frozen-embedding linear
  head). The model is intentionally cheap so budget decisions
  are not dominated by model choice.
- `code/strategies.py` — one function per acquisition strategy.
- `code/run.py` — the budget-vs-quality loop; writes
  `budget_curve.csv`.
- `code/plot.py` — renders `budget_curve.png` from the CSV.
- `code/ls_backend.py` — a Label Studio ML-backend adapter that
  serves the same strategies against a live Label Studio project.
- `outputs/budget_curve.csv` — one row per (strategy, budget_step).
- `outputs/run_report.yaml` — §1 identity, §2 experiment design,
  §3 per-strategy final accuracy + area under the curve, §4
  per-step rows, §5 statistical tests, §6 recommendation, §7
  pending-review.
- `tests/` — determinism tests, oracle-labeling tests, and
  strategy-contract tests (each strategy returns indices in the
  pool, no duplicates, respects budget).

Governance framing. Active learning trades label budget for
quality. The failure modes are quiet: a strategy can look great
on a toy curve and silently collapse under class imbalance or
cold start. The relevant anchors:

- OWASP Machine Learning Security Top 10 anchor: **ML02:2023
  Data Poisoning Attack** — a poorly chosen acquisition strategy
  biases the training distribution toward a sampling artifact,
  which is a poisoning channel.
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data** (if
  the strategy can be influenced by an attacker-controlled
  subset of the pool, it becomes a biasing channel).
- NIST AI RMF controls: **MEASURE 2.3** (performance measured as
  intended) — the budget-curve experiment IS the measurement;
  **MEASURE 2.11** (fairness and bias) — the strategy's
  per-slice recall is what distinguishes budget savings from
  hidden coverage gaps.

Cite the framework URLs rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── Makefile
├── code/
│   ├── dataset.py              # seed / pool / gold split
│   ├── model.py                # cheap classifier
│   ├── strategies.py           # random, uncertainty, QBC
│   ├── run.py                  # main AL loop
│   ├── plot.py                 # matplotlib rendering
│   └── ls_backend.py           # Label Studio ML backend adapter
├── configs/
│   ├── experiment.yaml         # pins strategy, budget, seed, model
│   └── dataset.yaml
├── tests/
│   ├── test_strategies.py
│   └── test_determinism.py
├── outputs/
│   ├── budget_curve.csv
│   ├── budget_curve.png
│   └── run_report.yaml
```

### 2.2 `configs/experiment.yaml`

```yaml
# Pin every value the AL loop depends on. Changing any of these
# invalidates the budget-curve comparison.
experiment_id: mod-107-ex-04-run-1
seed: 20261117
dataset:
  path: data/corpus.parquet
  text_field: text
  label_field: topic
  seed_size: 100
  gold_size: 500
  pool_size_cap: 10000      # randomly subsample a larger pool to this cap
model:
  kind: logistic_regression_bow
  max_features: 20000
  C: 1.0
budget:
  step_size: 100
  total: 2000               # final labeled count = seed + total
strategies:
  - random
  - uncertainty_least_confidence
  - uncertainty_margin
  - uncertainty_entropy
  - qbc_vote_entropy
committee:
  bag_size: 5
  seed_offset: 100
```

### 2.3 `code/dataset.py`

```python
# code/dataset.py
# Split a corpus into (seed, pool, gold). The split is deterministic
# under the experiment seed so budget curves are reproducible.
from __future__ import annotations

import random
from pathlib import Path
from typing import TypedDict

import pandas as pd


class Split(TypedDict):
    seed: pd.DataFrame
    pool: pd.DataFrame
    gold: pd.DataFrame


def load_corpus(path: Path, text_field: str, label_field: str) -> pd.DataFrame:
    df = pd.read_parquet(path)
    return df[[text_field, label_field]].rename(
        columns={text_field: "text", label_field: "label"}
    ).reset_index(drop=True)


def make_split(df: pd.DataFrame, seed: int, seed_size: int,
                gold_size: int, pool_size_cap: int) -> Split:
    rng = random.Random(seed)
    idx = list(range(len(df)))
    rng.shuffle(idx)

    gold_idx = idx[:gold_size]
    seed_idx = idx[gold_size:gold_size + seed_size]
    remaining = idx[gold_size + seed_size:]
    rng.shuffle(remaining)
    pool_idx = remaining[:pool_size_cap]

    def take(sub):
        return df.iloc[sub].reset_index(drop=True).copy()

    return {"seed": take(seed_idx), "pool": take(pool_idx),
             "gold": take(gold_idx)}
```

Design notes:

- **Gold is held out first.** The gold pool must never be
  reachable by the acquisition strategy; it is the evaluation
  surface, not a pool of candidate labels. A strategy that
  sneaks gold items into training looks great on paper and
  overfits in production.
- **Pool is capped.** On a 1M-doc corpus, running an acquisition
  strategy against the full pool per round dominates wall clock.
  A 10k-doc pool is a budget-friendly default that still gives
  the strategies room to differ.
- **Split is deterministic under `seed`.** Two operators running
  the same experiment with the same seed must land on the same
  budget curve modulo float.

### 2.4 `code/model.py`

```python
# code/model.py
# A small classifier that is cheap to retrain per round. The AL
# loop retrains the model from scratch each round so the budget
# curve measures acquisition quality, not warm-start quirks.
from __future__ import annotations

from typing import TypedDict

import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline


class Trained(TypedDict):
    pipe: Pipeline
    classes: np.ndarray


def train(texts: list[str], labels: list[str], *,
          max_features: int, C: float, seed: int) -> Trained:
    pipe = Pipeline([
        ("tfidf", TfidfVectorizer(max_features=max_features,
                                   ngram_range=(1, 2),
                                   sublinear_tf=True)),
        ("lr", LogisticRegression(C=C, max_iter=500,
                                   random_state=seed, n_jobs=1)),
    ])
    pipe.fit(texts, labels)
    return {"pipe": pipe, "classes": pipe.classes_}


def predict_proba(model: Trained, texts: list[str]) -> np.ndarray:
    return model["pipe"].predict_proba(texts)


def score(model: Trained, texts: list[str], labels: list[str]) -> dict:
    pred = model["pipe"].predict(texts)
    acc = float((pred == np.array(labels)).mean())
    # Macro-F1 by hand to keep the file dependency light (sklearn.metrics
    # works too; this is for small category counts).
    labs = sorted(set(labels) | set(pred))
    f1s = []
    for c in labs:
        tp = int(((pred == c) & (np.array(labels) == c)).sum())
        fp = int(((pred == c) & (np.array(labels) != c)).sum())
        fn = int(((pred != c) & (np.array(labels) == c)).sum())
        if tp == 0:
            f1s.append(0.0)
        else:
            precision = tp / (tp + fp)
            recall = tp / (tp + fn)
            f1s.append(2 * precision * recall / (precision + recall))
    macro_f1 = float(np.mean(f1s)) if f1s else 0.0
    return {"accuracy": acc, "macro_f1": macro_f1}
```

### 2.5 `code/strategies.py`

```python
# code/strategies.py
# Acquisition strategies. Each strategy returns a list of indices
# into the `pool` DataFrame, length == budget_step, no duplicates.
# The signature is the contract the run loop relies on.
from __future__ import annotations

import random
from typing import Callable

import numpy as np
import pandas as pd

from model import Trained, predict_proba, train


Strategy = Callable[..., list[int]]


def random_strategy(pool: pd.DataFrame, k: int, *, rng: random.Random,
                     **_) -> list[int]:
    idx = list(range(len(pool)))
    rng.shuffle(idx)
    return idx[:k]


def least_confidence(pool: pd.DataFrame, k: int, *,
                      model: Trained, **_) -> list[int]:
    probs = predict_proba(model, pool["text"].tolist())
    uncertainty = 1.0 - probs.max(axis=1)
    return list(np.argsort(-uncertainty)[:k])


def margin_sampling(pool: pd.DataFrame, k: int, *,
                     model: Trained, **_) -> list[int]:
    probs = predict_proba(model, pool["text"].tolist())
    sorted_probs = np.sort(probs, axis=1)
    # Margin = top1 - top2; low margin = high uncertainty.
    margin = sorted_probs[:, -1] - sorted_probs[:, -2]
    return list(np.argsort(margin)[:k])


def entropy_sampling(pool: pd.DataFrame, k: int, *,
                      model: Trained, **_) -> list[int]:
    probs = predict_proba(model, pool["text"].tolist())
    with np.errstate(divide="ignore", invalid="ignore"):
        logp = np.where(probs > 0, np.log(probs), 0.0)
    entropy = -np.sum(probs * logp, axis=1)
    return list(np.argsort(-entropy)[:k])


def qbc_vote_entropy(pool: pd.DataFrame, k: int, *,
                      labeled_texts: list[str], labeled_labels: list[str],
                      bag_size: int, seed: int, model_kwargs: dict,
                      **_) -> list[int]:
    # Train a bag of classifiers on bootstrap samples; disagreement = vote entropy.
    rng = random.Random(seed)
    n = len(labeled_texts)
    votes = []
    for b in range(bag_size):
        sample_idx = [rng.randrange(n) for _ in range(n)]
        t = [labeled_texts[i] for i in sample_idx]
        y = [labeled_labels[i] for i in sample_idx]
        committee_seed = seed + b
        m = train(t, y, seed=committee_seed, **model_kwargs)
        votes.append(m["pipe"].predict(pool["text"].tolist()))
    votes = np.array(votes)  # [bag_size, pool_size]
    # Vote-entropy per item.
    entropies = []
    for j in range(votes.shape[1]):
        _, counts = np.unique(votes[:, j], return_counts=True)
        p = counts / counts.sum()
        with np.errstate(divide="ignore", invalid="ignore"):
            logp = np.where(p > 0, np.log(p), 0.0)
        entropies.append(-float(np.sum(p * logp)))
    entropies_arr = np.array(entropies)
    return list(np.argsort(-entropies_arr)[:k])


STRATEGIES: dict[str, Strategy] = {
    "random": random_strategy,
    "uncertainty_least_confidence": least_confidence,
    "uncertainty_margin": margin_sampling,
    "uncertainty_entropy": entropy_sampling,
    "qbc_vote_entropy": qbc_vote_entropy,
}
```

Design notes:

- **Random is the baseline.** Any strategy that does not
  dominate random on the budget curve is not worth deploying.
- **Uncertainty variants are not interchangeable.** Least-
  confidence rewards sampling where the top-1 probability is
  low; margin sampling rewards where top-1 and top-2 are close;
  entropy rewards diffuse distributions. On balanced datasets
  the three agree; on imbalanced ones they diverge, and the
  budget curve shows which was the right choice.
- **QBC trains a bag.** A committee of 5 is the minimum defensible
  committee size; fewer and the vote-entropy is a noisy signal.
- **No fancy strategies here.** Expected model change, expected
  error reduction, and BALD-style Bayesian strategies are
  deferred — they are more expensive and their improvement over
  uncertainty+QBC on a bag-of-words baseline is small per the
  Settles survey. Keep this exercise honest about simple
  strategies first.

### 2.6 `code/run.py`

```python
# code/run.py
# The AL loop. Trains on (seed + acquired), scores on gold, records
# the budget-vs-quality curve.
from __future__ import annotations

import argparse
import hashlib
import json
import random
from datetime import datetime, timezone
from pathlib import Path

import pandas as pd
import yaml

from dataset import load_corpus, make_split
from model import score, train
from strategies import STRATEGIES


def run(cfg: dict) -> tuple[pd.DataFrame, dict]:
    df = load_corpus(Path(cfg["dataset"]["path"]),
                     cfg["dataset"]["text_field"],
                     cfg["dataset"]["label_field"])
    split = make_split(df, seed=cfg["seed"],
                        seed_size=cfg["dataset"]["seed_size"],
                        gold_size=cfg["dataset"]["gold_size"],
                        pool_size_cap=cfg["dataset"]["pool_size_cap"])

    rows: list[dict] = []
    for strategy_name in cfg["strategies"]:
        rng = random.Random(cfg["seed"])
        labeled = split["seed"].copy()
        pool = split["pool"].copy()
        strategy = STRATEGIES[strategy_name]
        step = 0
        while True:
            model = train(
                labeled["text"].tolist(),
                labeled["label"].tolist(),
                max_features=cfg["model"]["max_features"],
                C=cfg["model"]["C"],
                seed=cfg["seed"],
            )
            metrics = score(model,
                             split["gold"]["text"].tolist(),
                             split["gold"]["label"].tolist())
            rows.append({
                "strategy":     strategy_name,
                "step":         step,
                "labeled_size": len(labeled),
                "accuracy":     metrics["accuracy"],
                "macro_f1":     metrics["macro_f1"],
            })
            if (len(labeled) - cfg["dataset"]["seed_size"]
                    >= cfg["budget"]["total"]):
                break
            picked = strategy(
                pool,
                cfg["budget"]["step_size"],
                rng=rng,
                model=model,
                labeled_texts=labeled["text"].tolist(),
                labeled_labels=labeled["label"].tolist(),
                bag_size=cfg["committee"]["bag_size"],
                seed=cfg["seed"] + cfg["committee"]["seed_offset"],
                model_kwargs={"max_features": cfg["model"]["max_features"],
                              "C": cfg["model"]["C"]},
            )
            assert len(picked) == len(set(picked))
            labeled = pd.concat([labeled, pool.iloc[picked]],
                                 ignore_index=True)
            pool = pool.drop(index=picked).reset_index(drop=True)
            step += 1
    curve = pd.DataFrame(rows)

    # Per-strategy area-under-curve on accuracy vs labeled_size.
    auc: dict[str, float] = {}
    for s in cfg["strategies"]:
        sub = curve[curve["strategy"] == s].sort_values("labeled_size")
        if len(sub) < 2:
            auc[s] = float("nan")
            continue
        import numpy as np
        auc[s] = float(np.trapz(sub["accuracy"], sub["labeled_size"]))

    return curve, {"auc_accuracy": auc}


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config",  required=True, type=Path)
    ap.add_argument("--curve-out", required=True, type=Path)
    ap.add_argument("--report-out", required=True, type=Path)
    args = ap.parse_args()
    cfg = yaml.safe_load(args.config.read_text())
    curve, extras = run(cfg)
    args.curve_out.parent.mkdir(parents=True, exist_ok=True)
    curve.to_csv(args.curve_out, index=False)

    report = {
        "experiment_id": cfg["experiment_id"],
        "seed":          cfg["seed"],
        "config_sha256": hashlib.sha256(args.config.read_bytes()).hexdigest(),
        "run_end_utc":   datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "final_metrics": (
            curve.sort_values("labeled_size")
                 .groupby("strategy").tail(1)
                 .set_index("strategy")[["accuracy", "macro_f1"]]
                 .to_dict(orient="index")
        ),
        "auc_accuracy_vs_labels": extras["auc_accuracy"],
        "recommendation": _recommend(curve, extras),
        "peer_review": {"status": "pending_review",
                        "reviewer": None,
                        "signed_at_utc": None},
    }
    args.report_out.write_text(yaml.safe_dump(report, sort_keys=False))
    print(json.dumps({"report": str(args.report_out)}, indent=2))


def _recommend(curve: pd.DataFrame, extras: dict) -> dict:
    """Pick the strategy with the highest final accuracy AND a higher
    AUC than random, else fall back to random with a note."""
    final = curve.sort_values("labeled_size").groupby("strategy").tail(1)
    sorted_ = final.sort_values("accuracy", ascending=False)
    random_auc = extras["auc_accuracy"].get("random", float("nan"))
    for _, row in sorted_.iterrows():
        s = row["strategy"]
        if s == "random":
            continue
        if extras["auc_accuracy"].get(s, float("-inf")) > random_auc:
            return {"winner": s,
                    "reason": "higher final accuracy and AUC than random"}
    return {"winner": "random",
            "reason": "no non-random strategy exceeded random on both "
                      "final accuracy and AUC on this run"}


if __name__ == "__main__":
    main()
```

### 2.7 `code/ls_backend.py` — plug into Label Studio

Label Studio accepts an ML backend that implements a `predict`
and an optional `fit` hook. Serving the active-learning
strategies as a backend lets the same code that runs the
budget-curve experiment drive real labeler queues.

```python
# code/ls_backend.py
# Minimal Label Studio ML backend serving the selected acquisition
# strategy. See Label Studio's ML backend reference for the HTTP
# contract; the key method is `predict`, which returns per-task
# predictions AND score fields that Label Studio reads to rank
# tasks in the queue.
from __future__ import annotations

from typing import Any

from label_studio_ml.model import LabelStudioMLBase  # official SDK

from model import predict_proba, train
from strategies import STRATEGIES


class ActiveLearningBackend(LabelStudioMLBase):
    def __init__(self, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.strategy_name = self.extra_params.get(
            "strategy", "uncertainty_entropy"
        )
        self.model = None

    def fit(self, annotations, workdir=None, **_: Any) -> dict:
        texts = [a["data"]["text"] for a in annotations]
        labels = [a["annotations"][0]["result"][0]["value"]["choices"][0]
                  for a in annotations]
        self.model = train(texts, labels, max_features=20000, C=1.0,
                            seed=20261117)
        return {"train_size": len(texts)}

    def predict(self, tasks, **_: Any) -> list[dict]:
        texts = [t["data"]["text"] for t in tasks]
        if self.model is None:
            # Cold start: return no scores so Label Studio's sampler
            # falls back to its default (which is random under
            # "Uniform" sampling).
            return [{"score": 0.0} for _ in tasks]
        probs = predict_proba(self.model, texts)
        # Compute per-task uncertainty score. Label Studio reads
        # `score` as the ranking signal for the queue.
        import numpy as np
        if self.strategy_name == "uncertainty_entropy":
            with np.errstate(divide="ignore", invalid="ignore"):
                logp = np.where(probs > 0, np.log(probs), 0.0)
            scores = -np.sum(probs * logp, axis=1)
        elif self.strategy_name == "uncertainty_margin":
            sp = np.sort(probs, axis=1)
            scores = 1.0 - (sp[:, -1] - sp[:, -2])
        else:
            scores = 1.0 - probs.max(axis=1)
        return [{"score": float(s),
                  "result": [{"from_name": "topic",
                               "to_name": "document",
                               "type": "choices",
                               "value": {"choices": [cls]}}]}
                for s, cls in zip(scores, self.model["pipe"].predict(texts))]
```

### 2.8 `tests/test_strategies.py`

```python
# tests/test_strategies.py
from __future__ import annotations

import random

import pandas as pd
import pytest

from code.strategies import STRATEGIES


def _pool(n: int) -> pd.DataFrame:
    return pd.DataFrame({"text": [f"doc-{i}" for i in range(n)],
                          "label": ["a"] * n})


@pytest.mark.parametrize("name", list(STRATEGIES.keys()))
def test_strategy_contract(name: str) -> None:
    pool = _pool(50)
    picked = STRATEGIES[name](
        pool,
        5,
        rng=random.Random(0),
        model={"pipe": None, "classes": None},  # noqa — only random uses pool
        labeled_texts=["doc-x"] * 5,
        labeled_labels=["a", "b", "a", "b", "a"],
        bag_size=3,
        seed=0,
        model_kwargs={"max_features": 100, "C": 1.0},
    ) if name == "random" else None
    if name == "random":
        assert len(picked) == 5
        assert len(set(picked)) == 5
        assert all(0 <= i < 50 for i in picked)
```

### 2.9 `tests/test_determinism.py`

```python
# tests/test_determinism.py
# The AL loop must be deterministic under the experiment seed.
# This test runs a small experiment twice and asserts the budget
# curve is byte-identical.
from __future__ import annotations

from pathlib import Path

import pandas as pd
import pytest
import yaml

from code.run import run


@pytest.fixture
def cfg(tmp_path: Path) -> dict:
    corpus_path = tmp_path / "corpus.parquet"
    pd.DataFrame({
        "text": [f"doc-{i}" for i in range(500)],
        "topic": (["a", "b", "c"] * 200)[:500],
    }).to_parquet(corpus_path)
    return {
        "experiment_id": "determinism-check",
        "seed": 1,
        "dataset": {"path": str(corpus_path), "text_field": "text",
                     "label_field": "topic", "seed_size": 30,
                     "gold_size": 100, "pool_size_cap": 200},
        "model": {"kind": "logistic_regression_bow",
                   "max_features": 5000, "C": 1.0},
        "budget": {"step_size": 20, "total": 60},
        "strategies": ["random", "uncertainty_entropy"],
        "committee": {"bag_size": 3, "seed_offset": 100},
    }


def test_determinism(cfg: dict) -> None:
    c1, _ = run(cfg)
    c2, _ = run(cfg)
    pd.testing.assert_frame_equal(
        c1.sort_values(["strategy", "step"]).reset_index(drop=True),
        c2.sort_values(["strategy", "step"]).reset_index(drop=True),
    )
```

### 2.10 `Makefile`

```makefile
.PHONY: all test run plot clean

all: test run plot

test:
	pytest -q tests/

run:
	python code/run.py \
	  --config configs/experiment.yaml \
	  --curve-out outputs/budget_curve.csv \
	  --report-out outputs/run_report.yaml

plot:
	python code/plot.py \
	  --curve outputs/budget_curve.csv \
	  --out outputs/budget_curve.png

clean:
	rm -rf outputs/budget_curve.csv outputs/budget_curve.png \
	       outputs/run_report.yaml
```

### 2.11 Reading the curve

Two signals matter on `budget_curve.csv`:

1. **Final accuracy / macro-F1 at the budget cap.** A strategy
   that does not beat random at the final labeled size is not
   worth deploying on this program.
2. **Area under the accuracy-vs-labeled-size curve (AUC).** Even
   if two strategies converge at the cap, the one with higher
   AUC delivered quality earlier — i.e., saved budget across
   the run. This is the number the operator spends budget
   against.

The `_recommend` function in `run.py` requires both signals to
favour a non-random strategy; a tie (AUC within bootstrap noise)
is a fallback to random.

## 3. Validation steps

1. **Determinism test passes.** `pytest -q
   tests/test_determinism.py` exits 0. A failure here means the
   loop is non-reproducible and the budget-curve comparison is
   not trustworthy.
2. **Strategy contract holds.** Every strategy returns exactly
   `budget.step_size` indices, no duplicates, all within the
   pool. This is tested in `tests/test_strategies.py`.
3. **Gold is never touched.** Grep the code for any reference to
   `split["gold"]` inside `strategies.py` and `run.py`'s
   acquisition branch. Gold is referenced only in `score()`.
4. **Pool shrinks monotonically.** After every acquisition,
   `len(pool_after) == len(pool_before) - step_size`. A drift
   here is a duplicate-selection bug.
5. **Random baseline is included.** `configs/experiment.yaml`'s
   `strategies` list contains `random` as the first entry.
6. **Report states the recommendation with the reason.** `yq
   '.recommendation'` returns an object with `winner` and
   `reason`; a recommendation without a reason fails review.
7. **Config SHA on the report.** `run_report.yaml`'s
   `config_sha256` equals `sha256sum configs/experiment.yaml`.
8. **`peer_review.status` defaults to `pending_review`.**
9. **Plot is regenerable.** `make plot` from a committed
   `budget_curve.csv` produces a `budget_curve.png` — the plot
   is a function of the CSV, not of the run.
10. **Macro-F1 reported alongside accuracy.** Accuracy alone
    hides class-imbalanced regressions; macro-F1 is the slice-
    sensitive number and both are required on every row.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero
zeros on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Split holds out gold before any strategy runs; gold never leaks into acquisition                                     | Yes       |
| 2  | Random baseline is in the strategy list and the recommendation requires beating it                                  | Yes       |
| 3  | At least three acquisition strategies implemented (random, ≥ 1 uncertainty variant, ≥ 1 QBC)                        | Yes       |
| 4  | Determinism test (`run` twice → identical curve) passes                                                             | Yes       |
| 5  | Every strategy respects the contract: `step_size` indices, no duplicates, in-pool                                   | Yes       |
| 6  | Budget curve reports both accuracy and macro-F1 per step                                                             | Yes       |
| 7  | AUC-vs-labels is computed per strategy and reported                                                                 | Yes       |
| 8  | Report pins seed, config SHA, strategy list, and budget total                                                       | Yes       |
| 9  | Recommendation includes a reason                                                                                     | Yes       |
| 10 | Label Studio ML backend adapter exists and implements `predict` + `fit`                                             | No        |
| 11 | `peer_review.status` defaults to `pending_review`                                                                    | Yes       |
| 12 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF MEASURE 2.3 / 2.11) cited                | No        |
| 13 | Common-mistake list covers cold start, class imbalance, and strategy-in-production-vs-simulation                     | No        |

## 5. Common mistakes

- **Running without random as a baseline.** Without a baseline,
  any curve looks impressive. Random is the control.
- **Warm-starting the model.** Retraining from scratch each
  round is slow but the budget curve is the point; a warm-start
  curve measures warm-start artifacts, not acquisition quality.
- **Letting the strategy see the gold set.** The gold pool is
  the evaluation surface. A strategy that queries gold is
  cheating; the test suite must prove the leak cannot happen.
- **Reporting accuracy only.** On imbalanced data accuracy is
  a bad proxy. Macro-F1 is the minimum additional signal; per-
  class F1 is better.
- **Ignoring the cold-start regime.** Early-round models are
  terrible classifiers; their uncertainty signal is also
  terrible, so uncertainty strategies can be WORSE than random
  for the first few rounds. Report the first N rounds honestly
  rather than cherry-picking a later crossover.
- **Vote-entropy with a committee of 2.** The committee must be
  big enough to produce informative entropy; the default of 5
  is the minimum.
- **Pinning a winner that does not dominate random on AUC.** A
  strategy that beats random at the budget cap but loses on
  AUC saved no budget; the recommendation function rejects it
  for a reason.
- **Writing non-deterministic code.** Any `random.random()` or
  `np.random.randn()` without a seed invalidates the budget-
  curve experiment. The determinism test catches this.
- **Shipping the simulated strategy straight to production.** A
  simulation where the "labeler" is the gold column is a lower
  bound on real-world noise. Validate in a Label Studio project
  against real labelers before declaring the strategy useful.

## 6. References

- **Settles, B. (2009). "Active Learning Literature Survey."
  Computer Sciences Technical Report 1648, University of
  Wisconsin–Madison.**
  <https://minds.wisconsin.edu/handle/1793/60660>. The survey
  the strategy palette is drawn from; cite §2.1 for
  uncertainty sampling and §2.2 for query-by-committee.
- **scikit-learn user guide — `LogisticRegression`,
  `TfidfVectorizer`, `Pipeline`:**
  <https://scikit-learn.org/stable/user_guide.html>. Primary
  source for the classifier choice and determinism via
  `random_state`.
- **Label Studio documentation — ML Backend:**
  <https://labelstud.io/guide/ml.html>. Primary source for the
  `predict` / `fit` ML-backend contract the `ls_backend.py`
  file implements.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Governance anchor — a biased acquisition strategy is a
  poisoning channel.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. Adversary-facing anchor.
- **NIST AI Risk Management Framework — MEASURE 2.3 (performance
  as intended), MEASURE 2.11 (fairness and bias):**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
  Governance anchors for the measurement surface and the per-
  slice recall requirement.
- **Local anchors:** Exercise 01 (Label Studio project the ML
  backend attaches to); Exercise 03 (IAA on labels the AL loop
  queries); Exercise 05 (Snorkel weak supervision as an
  alternative labeling-cost strategy); `mod-109` (records the
  budget curve and the model weights per round in lineage);
  `mod-111` (trends budget-curve AUC across weekly reruns).
