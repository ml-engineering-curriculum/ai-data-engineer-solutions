# Exercise 05 — Snorkel Weak-Supervision Amplifier (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds a
Snorkel-style weak-supervision pipeline that **amplifies** a small
human-labeled slice into a large probabilistically-labeled training
set, then measures whether the amplifier helps. The deliverable is
not a label matrix: it is an auditable pipeline whose outputs the
rest of the data engineering stack can sign (dataset card, lineage
manifest) and whose quality can be defended end-to-end.

Core concepts:

- **Labeling functions (LFs)** are small Python functions that
  return one of `{POSITIVE, NEGATIVE, ABSTAIN}` on an example.
  Each LF is a weak signal — heuristics, keyword rules, regexes,
  lookup against a gazetteer, or an external classifier.
- The **label model** reads the LF matrix and estimates each
  example's class probability by learning LF accuracies and
  pairwise correlations from the agreement/disagreement pattern
  — *without* using ground-truth labels at fit time.
- The resulting **probabilistic labels** become the training set
  for an **end-model** (the "noise-aware" classifier). The
  end-model is evaluated against a held-out human-labeled dev
  set that was *not* used to write the LFs.

Deliverables:

- An **LF policy note** pinned before any LF is written —
  taxonomy, LF naming convention, abstain contract, how LFs are
  reviewed, how LF drift is detected.
- A versioned **`labeling_functions/` directory** with ≥ 5
  independent LFs and per-LF unit tests.
- A **label-model fit script** that produces a reproducible
  `lf_summary.parquet` (per-LF coverage, overlaps, conflicts,
  learned accuracy) and `probabilistic_labels.parquet` for the
  unlabeled pool.
- An **end-model benchmark** that compares three training
  regimes on the same architecture and dev set:
  1. `human_only` — train on the small human slice;
  2. `ws_only` — train on Snorkel probabilistic labels only;
  3. `ws_then_human` — train on probabilistic labels, fine-tune
     on the human slice.
- A `report.md` that states whether weak supervision helps for
  this task at this budget, and if so by how much, with
  confidence intervals over seeds.

Governance framing. Weak supervision introduces **synthetic labels**
into the training set — the control anchor is **NIST AI RMF MAP 2.3**
(scientific integrity of training data; a weak-supervision derived
label must be documented as such) combined with **MEASURE 2.3**
(measured as intended; the dev set must stay untouched). The
adversary-facing anchor is **OWASP ML Top 10 ML02:2023 Data
Poisoning** — a compromised LF is a bulk-poisoning vector because
one bad heuristic can relabel millions of rows — and **MITRE ATLAS
AML.T0020 Poison Training Data**. The pipeline must log every LF's
provenance so a poisoned LF can be identified and reversed.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── policy_note.md                  # pinned before any LF is written
├── labeling_functions/
│   ├── __init__.py
│   ├── lf_keyword_profanity.py
│   ├── lf_regex_url.py
│   ├── lf_length_short.py
│   ├── lf_gazetteer_brand.py
│   ├── lf_distilbert_pretrained.py # external classifier LF
│   └── tests/
│       ├── test_lf_keyword_profanity.py
│       └── ...
├── code/
│   ├── apply_lfs.py                # builds LF matrix
│   ├── fit_label_model.py          # Snorkel label model → probabilistic labels
│   ├── train_end_model.py          # end-model training; three regimes
│   ├── evaluate.py                 # dev-set evaluation with seed CIs
│   ├── emit_dataset_card.py        # dataset card for probabilistic labels
│   └── emit_report.py
├── data/
│   ├── human_slice.jsonl           # ≤ 500 human-labeled items
│   ├── unlabeled_pool.jsonl        # ≥ 20,000 items
│   ├── dev.jsonl                   # held-out human-labeled, LF-blinded
│   └── test.jsonl                  # held-out human-labeled, for final number
└── results/
    ├── lf_matrix.parquet
    ├── lf_summary.parquet
    ├── probabilistic_labels.parquet
    ├── metrics.parquet
    ├── dataset_card.md
    └── report.md
```

### 2.3 `policy_note.md` — pin this before LF development

```markdown
# Weak-supervision policy — program <program-id>

**Decision date:** 2026-11-18
**Author:** <operator email>

## Task and taxonomy
- Binary classification (POSITIVE / NEGATIVE) on <task-domain>.
  See taxonomy from mod-107 ex-01.
- LFs return `{POSITIVE, NEGATIVE, ABSTAIN}`; multi-class tasks
  decompose into per-class binary LF stacks.

## LF authoring rules
- Each LF lives in its own `.py` file named `lf_<reason>.py`.
- Each LF has at least one unit test asserting its behaviour on a
  worked example per output class.
- LF coverage, individual accuracy on a held-out dev-LF slice (separate
  from `dev.jsonl`), and conflict rate with every other LF are logged
  at `lf_summary.parquet` and reviewed before fitting the label model.
- **Dev/test blindness:** No LF may be tuned using `dev.jsonl` or
  `test.jsonl`. LF development uses `data/human_slice.jsonl` only.

## Review and provenance
- Every LF commit names an author and an intent; LFs that are later
  removed must be removed with a commit that says *why*.
- Pretrained-classifier LFs name the model, version, and source URL
  in the LF docstring. A model-weights change requires a new LF file
  with a bumped version suffix; the old file remains in git for
  traceability.
- Gazetteer LFs commit the gazetteer alongside the LF under
  `labeling_functions/gazetteers/`; gazetteer updates follow the same
  LF-provenance rules.

## Label-model decisions
- Expected class balance is set from the human slice's class rate and
  recorded in `policy_note.md`; changing it invalidates the fit.
- Probabilistic labels below a chosen confidence are either dropped
  or retained with a sample weight equal to `max(p) - 0.5` (chosen in
  the fit script; whichever is used is documented here).

## Adversarial controls
- Any LF whose learned accuracy is below 0.5 is **not** silently
  down-weighted; it is quarantined (moved to `labeling_functions/quarantined/`)
  and reviewed. A systematically-wrong LF is as informative as a
  noisy-correct one and may indicate corruption of its inputs.
- Gazetteer and keyword LFs include an SHA-256 of their inputs in
  the LF summary so a swap is detectable.
```

Commit `policy_note.md` in its own commit before any LF; `git log
--follow` trail is the audit evidence.

### 2.4 `labeling_functions/__init__.py` — registry

```python
# labeling_functions/__init__.py
# A small registry so LFs can be enumerated in a stable order.
from __future__ import annotations

from dataclasses import dataclass
from typing import Callable

ABSTAIN = -1
NEGATIVE = 0
POSITIVE = 1


@dataclass(frozen=True)
class LF:
    name: str
    version: str        # bump when semantics change
    func: Callable[[dict], int]
    source: str = ""    # e.g., git URL + commit SHA of a model
```

### 2.5 Example LFs

```python
# labeling_functions/lf_keyword_profanity.py
from __future__ import annotations

import hashlib
import json
from pathlib import Path

from . import LF, POSITIVE, ABSTAIN

GAZETTEER = Path(__file__).parent / "gazetteers" / "profanity_en.txt"
GAZETTEER_SHA = hashlib.sha256(GAZETTEER.read_bytes()).hexdigest()


_WORDS = {w.strip().lower() for w in GAZETTEER.read_text().splitlines() if w.strip()}


def _apply(x: dict) -> int:
    tokens = x["text"].lower().split()
    return POSITIVE if any(t in _WORDS for t in tokens) else ABSTAIN


lf_keyword_profanity = LF(
    name="lf_keyword_profanity", version="1.0",
    func=_apply,
    source=f"gazetteer_sha256={GAZETTEER_SHA}",
)
```

```python
# labeling_functions/lf_regex_url.py
from __future__ import annotations

import re

from . import LF, POSITIVE, ABSTAIN

_URL = re.compile(r"https?://\S{10,}", re.IGNORECASE)


def _apply(x: dict) -> int:
    # Heuristic: outbound links in short-form text correlate with POSITIVE
    # in this task; defensible only against a dev-LF slice (see policy).
    return POSITIVE if _URL.search(x["text"]) else ABSTAIN


lf_regex_url = LF(name="lf_regex_url", version="1.0", func=_apply)
```

```python
# labeling_functions/lf_length_short.py
from __future__ import annotations

from . import LF, NEGATIVE, ABSTAIN


def _apply(x: dict) -> int:
    # Very short texts tend NEGATIVE in this task. Reasonable upstream
    # bias prior; weaker signal than the keyword LF.
    return NEGATIVE if len(x["text"].split()) < 5 else ABSTAIN


lf_length_short = LF(name="lf_length_short", version="1.0", func=_apply)
```

```python
# labeling_functions/lf_distilbert_pretrained.py
from __future__ import annotations

from functools import lru_cache
from pathlib import Path

from transformers import pipeline

from . import LF, NEGATIVE, POSITIVE, ABSTAIN

MODEL_ID = "distilbert-base-uncased-finetuned-sst-2-english"
MODEL_REV = "714eb0f"  # pinned commit; bump requires a new LF version

@lru_cache(maxsize=1)
def _model():
    return pipeline("text-classification", model=MODEL_ID, revision=MODEL_REV)


def _apply(x: dict) -> int:
    out = _model()(x["text"][:512])[0]
    if out["score"] < 0.8:
        return ABSTAIN
    return POSITIVE if out["label"] == "POSITIVE" else NEGATIVE


lf_distilbert_pretrained = LF(
    name="lf_distilbert_pretrained", version="1.0",
    func=_apply,
    source=f"huggingface:{MODEL_ID}@{MODEL_REV}",
)
```

Design points defensible at review:

- **Each LF is its own file.** A monolithic `lfs.py` makes LF
  provenance (who added, when, why) harder to read in git; the
  one-file-per-LF rule enforces traceability.
- **LFs that use external artefacts (gazetteers, pretrained models)
  pin and hash those artefacts.** A gazetteer swap is a bulk
  poisoning path.
- **LFs abstain often; silence is cheaper than wrong.** The label
  model weights an LF by its learned accuracy and coverage;
  low-accuracy LFs that abstain less can be more harmful than
  low-accuracy LFs that abstain more.

### 2.6 `code/apply_lfs.py` — build the LF matrix

```python
# code/apply_lfs.py
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq

from labeling_functions import LF
from labeling_functions.lf_keyword_profanity import lf_keyword_profanity
from labeling_functions.lf_regex_url import lf_regex_url
from labeling_functions.lf_length_short import lf_length_short
from labeling_functions.lf_gazetteer_brand import lf_gazetteer_brand
from labeling_functions.lf_distilbert_pretrained import lf_distilbert_pretrained


LFS: list[LF] = [
    lf_keyword_profanity,
    lf_regex_url,
    lf_length_short,
    lf_gazetteer_brand,
    lf_distilbert_pretrained,
]


def apply_to_pool(pool_path: Path, out_path: Path) -> np.ndarray:
    rows = [json.loads(line) for line in pool_path.read_text().splitlines()]
    L = np.full((len(rows), len(LFS)), fill_value=-1, dtype=np.int8)
    for j, lf in enumerate(LFS):
        for i, rec in enumerate(rows):
            L[i, j] = int(lf.func(rec))
    tbl = pa.table(
        {"example_id": [r.get("id", str(i)) for i, r in enumerate(rows)]}
        | {lf.name: L[:, j] for j, lf in enumerate(LFS)}
    )
    pq.write_table(tbl, out_path)
    return L


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pool", required=True, type=Path)
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()
    L = apply_to_pool(args.pool, args.out)
    print(json.dumps({
        "n_examples": int(L.shape[0]),
        "n_lfs": int(L.shape[1]),
        "coverage_per_lf": {
            LFS[j].name: float((L[:, j] != -1).mean())
            for j in range(L.shape[1])
        },
    }, indent=2))


if __name__ == "__main__":
    main()
```

### 2.7 `code/fit_label_model.py`

```python
# code/fit_label_model.py
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
from snorkel.labeling.model import LabelModel
from snorkel.labeling import LFAnalysis

from labeling_functions import POSITIVE, NEGATIVE


def fit(matrix_path: Path, class_balance: list[float],
        out_labels: Path, out_summary: Path, seed: int) -> dict:
    tbl = pq.read_table(matrix_path)
    L = np.stack([tbl.column(name).to_numpy() for name in tbl.column_names
                   if name != "example_id"], axis=1)
    lf_names = [n for n in tbl.column_names if n != "example_id"]

    label_model = LabelModel(cardinality=2, verbose=False, device="cpu")
    label_model.fit(L_train=L, class_balance=class_balance,
                    n_epochs=500, seed=seed, log_freq=100, lr=0.01)

    probs = label_model.predict_proba(L)  # (N, 2)
    pq.write_table(pa.table({
        "example_id": tbl.column("example_id"),
        "prob_negative": probs[:, 0],
        "prob_positive": probs[:, 1],
    }), out_labels)

    analysis = LFAnalysis(L=L).lf_summary()
    analysis.reset_index().rename(columns={"index": "lf_name"}) \
            .to_parquet(out_summary, index=False)

    return {
        "lf_names": lf_names,
        "mean_probability_high": float((probs.max(axis=1) >= 0.7).mean()),
    }


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--matrix", required=True, type=Path)
    ap.add_argument("--class-balance", nargs=2, type=float, required=True,
                    help="Class balance for [NEGATIVE, POSITIVE], from policy_note.md.")
    ap.add_argument("--seed", type=int, required=True)
    ap.add_argument("--out-labels", required=True, type=Path)
    ap.add_argument("--out-summary", required=True, type=Path)
    args = ap.parse_args()
    info = fit(args.matrix, args.class_balance, args.out_labels,
                args.out_summary, args.seed)
    print(json.dumps(info, indent=2))


if __name__ == "__main__":
    main()
```

Design notes defensible at review:

- **`class_balance` comes from the human slice, pinned in
  `policy_note.md`.** The label model is *identifiable only up to
  class label order* without the balance; getting the balance wrong
  flips the output class interpretation on borderline examples.
- **`seed` is a required argument.** Snorkel's label-model fit is
  stochastic; fixing the seed is the reproducibility knob.
- **`LFAnalysis.lf_summary()` is persisted.** The coverage,
  overlap, conflict, and learned accuracy table is the artifact
  reviewers scrutinise before accepting the probabilistic labels.

### 2.8 `code/train_end_model.py`

Three regimes on the same architecture:

```python
# code/train_end_model.py
from __future__ import annotations

import argparse
import json
from pathlib import Path

import numpy as np
import pandas as pd
import pyarrow.parquet as pq
import torch
import torch.nn as nn


def train(regime: str, human_path: Path, probabilistic_path: Path,
          dev_path: Path, seed: int) -> dict:
    """regime ∈ {human_only, ws_only, ws_then_human}."""
    torch.manual_seed(seed)
    # ... load tf-idf features x and labels y for each slice;
    # for ws_only, use soft-label cross-entropy (BCEWithLogitsLoss with
    # target = prob_positive); for ws_then_human, pretrain on ws then
    # fine-tune on human with a lower learning rate.
    #
    # Return a dict with the dev metric and a confusion matrix.
    ...
```

### 2.9 `code/emit_dataset_card.py`

A Snorkel-produced dataset is a *synthetic* dataset in the mod-108
sense; it must ship with a dataset card. Fields to include:

- Task statement, class taxonomy, class balance (from the human slice).
- The list of LFs with version strings and provenance sources.
- Label-model `class_balance` input and fit seed.
- LF summary table excerpt (coverage, conflict, learned accuracy).
- Explicit statement: "Labels are probabilistic; a confidence
  column is provided. This dataset is not a human-labeled gold
  reference and must not be used as such for evaluation."

### 2.10 `Makefile`

```makefile
.PHONY: all apply fit train evaluate card report test clean

all: apply fit train evaluate card report

apply:
	python code/apply_lfs.py --pool data/unlabeled_pool.jsonl \
		--out results/lf_matrix.parquet

fit:
	python code/fit_label_model.py --matrix results/lf_matrix.parquet \
		--class-balance 0.6 0.4 --seed 20261118 \
		--out-labels results/probabilistic_labels.parquet \
		--out-summary results/lf_summary.parquet

train:
	python code/train_end_model.py \
		--regimes human_only ws_only ws_then_human \
		--seeds 20261118 20261119 20261120 \
		--out results/metrics.parquet

evaluate:
	python code/evaluate.py --metrics results/metrics.parquet \
		--out results/metrics_summary.parquet

card:
	python code/emit_dataset_card.py --lf-summary results/lf_summary.parquet \
		--labels results/probabilistic_labels.parquet \
		--policy policy_note.md \
		--out results/dataset_card.md

report:
	python code/emit_report.py --metrics results/metrics_summary.parquet \
		--out results/report.md

test:
	pytest -q

clean:
	rm -rf results/*.parquet results/*.md
```

## 3. Validation steps

1. **Policy predates LFs.** `git log --follow policy_note.md | tail -1`
   returns a commit older than any `labeling_functions/*.py`. A
   retro-fitted policy fails review.
2. **Dev / test blindness.** `git grep -n 'dev.jsonl' labeling_functions/`
   returns no hits (LF code never reads dev/test slices).
3. **Per-LF unit tests pass.** `pytest labeling_functions/tests -q`
   exits 0.
4. **LF summary is persisted.** `results/lf_summary.parquet` contains
   `coverage, overlaps, conflicts, learned_accuracy` for every LF;
   the tool reads it and prints a human-readable table for review.
5. **Label-model fit is deterministic.** `make clean && make fit`
   twice; `sha256sum results/probabilistic_labels.parquet` is stable.
6. **Class balance is pinned.** The `--class-balance` passed to the
   fit matches the value documented in `policy_note.md`; a mismatch
   is a CI failure.
7. **End-model benchmark is run across ≥ 3 seeds.** `metrics.parquet`
   has ≥ 3 rows per regime; the summary reports mean and CI.
8. **ws_only is not evaluated on probabilistic labels.** Dev-metric
   computation uses only human-labeled dev examples; probabilistic
   labels never enter the metric.
9. **Low-accuracy LFs are quarantined, not down-weighted silently.**
   An LF whose learned accuracy is < 0.5 triggers a job failure that
   points the operator at the quarantined directory.
10. **Dataset card ships with every run.** `results/dataset_card.md`
    exists and lists every LF's version and provenance; a missing
    card blocks downstream mod-109 lineage.
11. **Report is honest about outcome.** If `ws_then_human ≤
    human_only` within CI, the report says "weak supervision does
    not help at this budget", with the numbers to back it; it does
    not post a selective result.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/32 with zero zeros on
critical rows.

| #  | Dimension                                                                                                          | Critical? |
|----|--------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `policy_note.md` predates all LFs; versioned in git                                                                | Yes       |
| 2  | ≥ 5 LFs, each in its own file, each with a unit test                                                                | Yes       |
| 3  | LFs return `{POSITIVE, NEGATIVE, ABSTAIN}`; abstain is the explicit silent return                                   | Yes       |
| 4  | External artefact LFs pin version + content hash (gazetteers, pretrained models)                                   | Yes       |
| 5  | Dev / test slices are LF-blind (no LF reads them)                                                                   | Yes       |
| 6  | Label-model `class_balance` comes from the human slice and is pinned in the policy                                  | Yes       |
| 7  | Label-model fit is seeded and byte-reproducible                                                                     | Yes       |
| 8  | LF summary (coverage / overlap / conflict / accuracy) persisted and reviewed                                        | Yes       |
| 9  | End-model benchmark runs three regimes (human_only, ws_only, ws_then_human) across ≥ 3 seeds                        | Yes       |
| 10 | Dev metric is computed only on human-labeled items                                                                  | Yes       |
| 11 | Low-accuracy LF handling (< 0.5): quarantine + manual review, no silent down-weight                                 | Yes       |
| 12 | Dataset card emitted with LF provenance, label-model config, and the "probabilistic, not gold" statement             | Yes       |
| 13 | Report states outcome honestly, including "WS does not help" if the data says so                                    | Yes       |
| 14 | Governance anchors (NIST AI RMF MAP 2.3 / MEASURE 2.3, OWASP ML02, MITRE ATLAS AML.T0020) cited                      | No        |
| 15 | Deterministic re-run (`make clean && make all` twice) produces byte-identical results modulo wall-clock              | Yes       |
| 16 | Gazetteer / keyword LFs log input SHAs; swap is detectable                                                          | Yes       |

## 5. Common mistakes

- **Writing LFs against the dev or test slice.** This is the single
  biggest trap. Every "look at where the model fails" session on
  dev data turns into an LF that pattern-matches dev, inflating
  reported numbers. LFs are developed on the human *training*
  slice only.
- **Hand-weighting LFs.** The label model learns weights from the
  LF matrix; a hand-picked `weight=0.3` on an LF whose data says
  `0.7` is just noise. Trust the label model or write a better LF.
- **Treating ABSTAIN as "the LF didn't vote".** ABSTAIN is a
  first-class return and is modelled as such; the LF matrix
  encodes it explicitly. Replacing ABSTAIN with 0 or the majority
  class hard-biases the label-model fit.
- **Picking `class_balance` from a vibe.** Use the measured balance
  from the human slice; if that slice is small, say so and bound
  the class_balance by its CI.
- **Evaluating on probabilistic labels.** The dev metric uses
  human labels. If probabilistic labels appear in the dev
  denominator, the number is self-congratulation.
- **Skipping the end-model comparison.** A high LF accuracy does
  not entail that an end-model trained on probabilistic labels
  beats one trained on the small human slice. Measure it.
- **Not pinning pretrained-model LFs.** `distilbert-...-finetuned-sst-2-english`
  has multiple revisions; without a pinned commit, the LF drifts
  silently.
- **Monolithic LF module.** A single file with 20 LFs has worthless
  git blame; move to one file per LF.
- **Silently keeping an LF with learned accuracy < 0.5.** A
  consistently-wrong LF is *informative* only if its sign is
  flipped by you intentionally. Silent down-weighting hides the
  underlying bug.
- **Skipping the dataset card.** Downstream (mod-109) will refuse
  the artefact without a card. Treat the card as part of the
  pipeline, not documentation.

## 6. References

- **NIST AI Risk Management Framework — MAP 2.3, MEASURE 2.3:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MAP 2.3
  anchors scientific integrity (document probabilistic labels as
  probabilistic); MEASURE 2.3 anchors the dev-blind metric.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  A compromised LF is bulk poisoning; LF provenance is the
  detection control.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing anchor.
- **Ratner, A., Bach, S. H., Ehrenberg, H., Fries, J., Wu, S., &
  Ré, C. (2017) "Snorkel: Rapid Training Data Creation with Weak
  Supervision."** VLDB 2017. The label-model + end-model approach
  this exercise implements.
- **Fu, D. Y., Chen, M. F., Sala, F., Hooper, S. M., Fatahalian,
  K., & Ré, C. (2020) "Fast and Three-rious: Speeding Up Weak
  Supervision with Triplet Methods."** ICML 2020. Background on
  the label-model fit algorithm.
- **Snorkel documentation — LabelModel and LFAnalysis:**
  <https://www.snorkel.org/use-cases/>. The reference for
  `LabelModel.fit`, `predict_proba`, and `LFAnalysis.lf_summary`.
- **HuggingFace `transformers` pipeline:**
  <https://huggingface.co/docs/transformers/main_classes/pipelines>.
  The API the pretrained-classifier LF calls.
- **Local anchors:** `mod-107` ex-01 (the labeling policy whose
  taxonomy the LFs honour and whose human slice is the LF
  development set); `mod-107` ex-03 (IAA on the human slice —
  which gates whether the slice is clean enough to anchor a WS
  class-balance estimate); `mod-108` (synthetic-data generation,
  where probabilistic labels are documented as a related but
  distinct category); `mod-109` (lineage + dataset-card signing
  for the amplified dataset).
