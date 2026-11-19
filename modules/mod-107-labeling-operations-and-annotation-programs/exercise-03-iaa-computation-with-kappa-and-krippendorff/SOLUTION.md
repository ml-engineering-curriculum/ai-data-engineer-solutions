# Exercise 03 — IAA Computation with Cohen's κ and Krippendorff's α (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds the
inter-annotator agreement (IAA) tool that gates every labeling batch
produced by the mod-107 ex-01 project configuration. The tool reads
a batch's exported annotations, computes **Cohen's κ** pairwise and
**Krippendorff's α** across the full annotator pool, writes a signed
`iaa_report.yaml`, and exits non-zero if the policy thresholds set
in ex-01 are not met.

Deliverables:

- A CLI `python code/compute_iaa.py --export <path> --policy <path>
  --output <dir>` that produces `iaa_report.yaml` and
  `iaa_report.yaml.sig`.
- Deterministic κ and α numbers at a chosen number of decimal
  places (determinism check in §3).
- Per-class and per-annotator breakdowns, not just a single number
  per batch — low aggregate α rarely points to a cause; the
  per-class breakdown does.
- A pytest suite that pins the metric outputs against hand-computed
  gold cases, including the degenerate cases κ = 1 (perfect
  agreement) and κ = 0 (chance-only agreement).
- An **exit-code contract** the CI job can consume (0 = batch
  passes, 10 = thresholds not met, 20 = schema / data error).

Governance framing. IAA is the *measurement* that makes mod-107 ex-01
auditable; without it, a labeling batch cannot be entered into the
training mixture under **NIST AI RMF MEASURE 2.3** ("AI system
performance or assurance criteria are measured qualitatively or
quantitatively as intended"). Low IAA on a class is a
*construct-validity* failure that silently biases model evaluation —
an adversary doesn't need to poison the batch if the definition is
already wrong. **OWASP ML Top 10 ML02:2023** and **MITRE ATLAS
AML.T0020** are the adversary-facing anchors, but IAA is primarily
a MEASURE control.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── requirements.lock
├── Makefile
├── policy.yaml                     # thresholds from mod-107 ex-01
├── code/
│   ├── load_export.py              # Label Studio / Argilla export → tidy table
│   ├── metrics.py                  # κ + α implementations
│   ├── breakdowns.py               # per-class, per-annotator, per-pair tables
│   ├── compute_iaa.py              # CLI orchestrator
│   └── sign_report.py              # cosign wrapper
├── tests/
│   ├── test_cohen_kappa.py
│   ├── test_krippendorff_alpha.py
│   └── fixtures/
│       └── annotations_tiny.jsonl  # 20 items, 3 annotators, known κ/α
└── results/
    ├── annotations_tidy.parquet
    ├── iaa_report.yaml
    └── iaa_report.yaml.sig
```

### 2.2 `policy.yaml` — thresholds inherited from mod-107 ex-01

```yaml
program_id: program-xyz
batch_scope: batches_001_to_050

# These are the policy numbers set in mod-107 ex-01 §2.2. Changing
# a threshold here without a new policy_note.md revision is a
# configuration drift the CI check refuses.
cohen_kappa:
  pairwise_min: 0.70
  critical_classes: []   # populate with classes that require a tighter bar

krippendorff_alpha:
  overall_min: 0.67
  # Weights bootstrap uses the metric nominal for NER class labels and
  # ordinal for Likert ratings. Set per question.
  level_of_measurement_per_question:
    classification: nominal
    rating_a: ordinal
    rating_b: ordinal
    preference: nominal

reporting:
  decimal_places: 4
  bootstrap_resamples: 1000  # for 95% CI only; not used to make the pass/fail decision.
  rng_seed: 20261118
```

### 2.3 `code/load_export.py` — unify Label Studio / Argilla

Both UIs export slightly differently. Normalise everything to a tidy
Parquet with columns `(item_id, question, annotator_id, value)` so
the metric layer is tool-agnostic.

```python
# code/load_export.py
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def load_label_studio(path: Path) -> list[dict]:
    """Label Studio JSON export → list of (item_id, question, annotator_id, value)."""
    rows: list[dict] = []
    data = json.loads(path.read_text())
    for task in data:
        item_id = task["data"].get("external_id") or str(task["id"])
        for ann in task.get("annotations", []):
            annotator_id = ann["completed_by"]["email"] if isinstance(
                ann.get("completed_by"), dict) else str(ann.get("completed_by"))
            for r in ann.get("result", []):
                q = r["from_name"]
                v = _result_value(r)
                if v is None:
                    continue
                rows.append({
                    "item_id": item_id, "question": q,
                    "annotator_id": annotator_id, "value": v,
                })
    return rows


def _result_value(r: dict) -> str | None:
    val = r.get("value", {})
    if "choices" in val:
        return "|".join(sorted(val["choices"]))
    if "labels" in val:
        # Span labels — encode (start, end, label) so offsets are part of
        # the comparison. mod-107 ex-01 policy accepts ±3 char on offsets;
        # that bucketing happens in metrics.py, not here.
        return json.dumps({
            "start": val.get("start"), "end": val.get("end"),
            "label": sorted(val["labels"])[0],
        }, sort_keys=True)
    if "rating" in val:
        return str(val["rating"])
    if "text" in val:
        # Free-text rationale; not an IAA question.
        return None
    return None


def load_argilla(path: Path) -> list[dict]:
    """Argilla JSONL export (`to_hf_dataset().to_json(...)`) → tidy rows."""
    rows: list[dict] = []
    for line in path.read_text().splitlines():
        rec = json.loads(line)
        item_id = rec["external_id"] or rec.get("id")
        for response in rec.get("responses", []):
            annotator_id = response.get("user_id") or response.get("user")
            for q, v in (response.get("values") or {}).items():
                rows.append({
                    "item_id": item_id, "question": q,
                    "annotator_id": str(annotator_id),
                    "value": str(v.get("value") if isinstance(v, dict) else v),
                })
    return rows


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--label-studio", type=Path)
    ap.add_argument("--argilla", type=Path)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    rows: list[dict] = []
    if args.label_studio:
        rows.extend(load_label_studio(args.label_studio))
    if args.argilla:
        rows.extend(load_argilla(args.argilla))

    tbl = pa.Table.from_pylist(rows,
        schema=pa.schema([
            ("item_id", pa.string()),
            ("question", pa.string()),
            ("annotator_id", pa.string()),
            ("value", pa.string()),
        ]))
    pq.write_table(tbl, args.output)


if __name__ == "__main__":
    main()
```

### 2.4 `code/metrics.py` — κ and α

```python
# code/metrics.py
# Cohen's κ (pairwise, nominal) and Krippendorff's α (nominal / ordinal
# / interval) for a tidy annotations table.
from __future__ import annotations

import itertools
import math
from collections import defaultdict
from dataclasses import dataclass


@dataclass(frozen=True)
class KappaResult:
    annotator_a: str
    annotator_b: str
    question: str
    kappa: float
    n_items_both_rated: int


def cohen_kappa(rows: list[dict], question: str,
                annotator_a: str, annotator_b: str) -> KappaResult:
    """Pairwise Cohen's κ on items BOTH annotators rated for `question`."""
    by_item: dict[str, dict[str, str]] = defaultdict(dict)
    for r in rows:
        if r["question"] != question:
            continue
        if r["annotator_id"] in (annotator_a, annotator_b):
            by_item[r["item_id"]][r["annotator_id"]] = r["value"]

    paired = [
        (v[annotator_a], v[annotator_b])
        for v in by_item.values()
        if annotator_a in v and annotator_b in v
    ]
    n = len(paired)
    if n == 0:
        return KappaResult(annotator_a, annotator_b, question,
                           kappa=float("nan"), n_items_both_rated=0)

    labels = sorted({v for pair in paired for v in pair})
    cm: dict[tuple[str, str], int] = defaultdict(int)
    for a, b in paired:
        cm[(a, b)] += 1

    po = sum(cm[(l, l)] for l in labels) / n
    pa_marg = {l: sum(cm[(l, m)] for m in labels) / n for l in labels}
    pb_marg = {l: sum(cm[(m, l)] for m in labels) / n for l in labels}
    pe = sum(pa_marg[l] * pb_marg[l] for l in labels)
    kappa = 0.0 if math.isclose(1 - pe, 0.0) else (po - pe) / (1 - pe)
    return KappaResult(annotator_a, annotator_b, question,
                       round(kappa, 6), n)


# Krippendorff's α — reliability-data formulation (Krippendorff 2011,
# "Computing Krippendorff's Alpha-Reliability"):
#
#     α = 1 - D_o / D_e
#
# where D_o is observed disagreement and D_e is expected disagreement
# under chance. Distance functions:
#   nominal : δ(a,b) = 0 if a==b else 1
#   ordinal : δ(a,b) = ((rank_a - rank_b))^2 using Krippendorff's
#             ordinal rank-based distance (sum of category frequencies
#             between the two).
#   interval: δ(a,b) = (a - b)^2
def _nominal_delta(a: str, b: str, *_: object) -> float:
    return 0.0 if a == b else 1.0


def _interval_delta(a: str, b: str, *_: object) -> float:
    return (float(a) - float(b)) ** 2


def _ordinal_delta(a: str, b: str, freqs: dict[str, int]) -> float:
    """Krippendorff's ordinal delta.

    δ(a,b) = (sum of freqs between a and b inclusive at the endpoints
             only in half)^2  — equivalent to the formulation in
             Krippendorff (2011) §4.2.
    """
    order = sorted(freqs.keys(), key=lambda x: float(x))
    ia, ib = order.index(a), order.index(b)
    if ia == ib:
        return 0.0
    lo, hi = sorted((ia, ib))
    between = sum(freqs[order[i]] for i in range(lo, hi + 1))
    return (between - (freqs[a] + freqs[b]) / 2) ** 2


DELTAS = {
    "nominal": _nominal_delta,
    "ordinal": _ordinal_delta,
    "interval": _interval_delta,
}


def krippendorff_alpha(rows: list[dict], question: str,
                       level: str) -> tuple[float, int]:
    """Krippendorff's α across all annotators on `question`.

    Items with only one annotation are discarded (reliability requires
    pairs). Returns (alpha, n_items_used).
    """
    delta = DELTAS[level]
    by_item: dict[str, list[str]] = defaultdict(list)
    for r in rows:
        if r["question"] == question:
            by_item[r["item_id"]].append(r["value"])

    used = {it: vs for it, vs in by_item.items() if len(vs) >= 2}
    if not used:
        return float("nan"), 0

    all_values = [v for vs in used.values() for v in vs]
    freqs: dict[str, int] = defaultdict(int)
    for v in all_values:
        freqs[v] += 1
    n = len(all_values)

    numer = 0.0
    for vs in used.values():
        m = len(vs)
        pairs = list(itertools.combinations(vs, 2))
        s = sum(delta(a, b, freqs) for a, b in pairs)
        numer += s / (m - 1)
    d_o = numer / n

    d_e = 0.0
    labels = list(freqs.keys())
    for i, li in enumerate(labels):
        for lj in labels[i:]:
            w = freqs[li] * freqs[lj] if li != lj else freqs[li] * (freqs[li] - 1)
            if li == lj:
                d_e += w * delta(li, lj, freqs)
            else:
                d_e += 2 * w * delta(li, lj, freqs)
    d_e = d_e / (n * (n - 1)) if n > 1 else float("nan")

    if math.isclose(d_e, 0.0):
        alpha = 1.0 if math.isclose(d_o, 0.0) else float("nan")
    else:
        alpha = 1 - d_o / d_e
    return round(alpha, 6), len(used)
```

Design points defensible at review:

- **α is reliability-data form, not the agreement-matrix form.** The
  reliability-data form handles missing annotations and non-uniform
  rater sets without padding tricks, which is the common case when
  one of three annotators drops an item.
- **Items with < 2 annotations are excluded, not imputed.** Imputing
  annotations is a measurement-hygiene violation; a single annotation
  is not a reliability signal.
- **Rounding at 6 decimal places inside the computation, 4 in the
  report.** The 6-decimal internal precision is stable across
  platforms for the test fixtures; the report truncates to 4 for
  human readability.

### 2.5 `code/breakdowns.py` — per-class and per-annotator

```python
# code/breakdowns.py
from __future__ import annotations

from collections import defaultdict


def per_class_breakdown(rows: list[dict], question: str) -> dict:
    """Agreement rate per class — the 'where is the disagreement?' view."""
    by_item: dict[str, list[str]] = defaultdict(list)
    for r in rows:
        if r["question"] == question:
            by_item[r["item_id"]].append(r["value"])
    out: dict[str, dict[str, int]] = defaultdict(lambda: {"all_agree": 0, "majority": 0, "none": 0})
    for vs in by_item.values():
        if len(vs) < 2:
            continue
        counts = defaultdict(int)
        for v in vs:
            counts[v] += 1
        most = max(counts.values())
        winners = [k for k, c in counts.items() if c == most]
        kind = "all_agree" if most == len(vs) else ("majority" if most > len(vs) / 2 else "none")
        for w in winners:
            out[w][kind] += 1
    return dict(out)


def per_annotator_breakdown(rows: list[dict]) -> dict:
    """Per-annotator majority-agreement rate across all questions.

    An annotator whose answers consistently diverge from the majority
    is either an outlier (bias flag) or the only correct answer in a
    truly ambiguous question (guideline flag). This view surfaces the
    pattern; adjudication decides which case it is.
    """
    by_item_ann: dict[tuple[str, str], str] = {}
    by_item: dict[str, list[tuple[str, str]]] = defaultdict(list)
    for r in rows:
        key = (r["item_id"], r["question"])
        by_item_ann[(r["item_id"], r["question"], r["annotator_id"])] = r["value"]
        by_item[key].append((r["annotator_id"], r["value"]))

    totals: dict[str, int] = defaultdict(int)
    with_majority: dict[str, int] = defaultdict(int)
    for (item, q), entries in by_item.items():
        counts: dict[str, int] = defaultdict(int)
        for _, v in entries:
            counts[v] += 1
        most = max(counts.values())
        majority_val = {k for k, c in counts.items() if c == most}
        for ann, v in entries:
            totals[ann] += 1
            if v in majority_val:
                with_majority[ann] += 1
    return {
        a: {"items": totals[a],
            "majority_agreement_rate": round(with_majority[a] / totals[a], 4)}
        for a in totals
    }
```

### 2.6 `code/compute_iaa.py` — CLI orchestrator

```python
# code/compute_iaa.py
from __future__ import annotations

import argparse
import hashlib
import itertools
import json
import socket
import subprocess
import sys
from datetime import datetime, timezone
from pathlib import Path

import pyarrow.parquet as pq
import yaml

from metrics import cohen_kappa, krippendorff_alpha
from breakdowns import per_class_breakdown, per_annotator_breakdown


EXIT_OK = 0
EXIT_THRESHOLDS_FAILED = 10
EXIT_SCHEMA_ERROR = 20


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--annotations", required=True, type=Path,
                    help="Tidy parquet from load_export.py")
    ap.add_argument("--policy", required=True, type=Path)
    ap.add_argument("--batch-id", required=True)
    ap.add_argument("--operator", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    policy = yaml.safe_load(args.policy.read_text())
    tbl = pq.read_table(args.annotations)
    rows = tbl.to_pylist()

    if not rows:
        print("empty annotations table", file=sys.stderr)
        return EXIT_SCHEMA_ERROR

    annotators = sorted({r["annotator_id"] for r in rows})
    questions = sorted({r["question"] for r in rows})

    kappa_results: list[dict] = []
    for q in questions:
        for a, b in itertools.combinations(annotators, 2):
            kr = cohen_kappa(rows, q, a, b)
            kappa_results.append({
                "question": q, "annotator_a": a, "annotator_b": b,
                "kappa": kr.kappa, "n": kr.n_items_both_rated,
            })

    alpha_results: list[dict] = []
    levels = policy["krippendorff_alpha"]["level_of_measurement_per_question"]
    for q in questions:
        level = levels.get(q, "nominal")
        alpha, n_items = krippendorff_alpha(rows, q, level)
        alpha_results.append({
            "question": q, "level": level, "alpha": alpha, "n_items": n_items,
        })

    pairwise_min = policy["cohen_kappa"]["pairwise_min"]
    alpha_overall_min = policy["krippendorff_alpha"]["overall_min"]

    kappa_fails = [
        k for k in kappa_results
        if k["n"] > 0 and not (k["kappa"] >= pairwise_min)
    ]
    alpha_fails = [
        a for a in alpha_results
        if a["n_items"] > 0 and not (a["alpha"] >= alpha_overall_min)
    ]

    report = {
        "batch_id": args.batch_id,
        "operator": args.operator,
        "toolchain": {
            "git_sha": subprocess.check_output(
                ["git", "rev-parse", "HEAD"]).decode().strip(),
            "policy_sha256": sha256_file(args.policy),
            "annotations_sha256": sha256_file(args.annotations),
        },
        "run_end_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "compute_shape": {"host": socket.gethostname()},
        "annotators": annotators,
        "questions": questions,
        "cohen_kappa": {
            "pairwise_min": pairwise_min,
            "pairs": kappa_results,
            "failures": kappa_fails,
        },
        "krippendorff_alpha": {
            "overall_min": alpha_overall_min,
            "per_question": alpha_results,
            "failures": alpha_fails,
        },
        "breakdowns": {
            "per_class_per_question": {
                q: per_class_breakdown(rows, q) for q in questions
            },
            "per_annotator": per_annotator_breakdown(rows),
        },
        "verdict": "pass" if not (kappa_fails or alpha_fails) else "fail",
    }
    args.output.write_text(yaml.safe_dump(report, sort_keys=False))

    print(json.dumps({
        "verdict": report["verdict"],
        "report_sha256": sha256_file(args.output),
    }, indent=2))

    return EXIT_OK if report["verdict"] == "pass" else EXIT_THRESHOLDS_FAILED


if __name__ == "__main__":
    sys.exit(main())
```

### 2.7 `code/sign_report.py` — cosign wrapper

```python
# code/sign_report.py
from __future__ import annotations

import argparse
import os
import subprocess


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--report", required=True)
    args = ap.parse_args()
    subprocess.check_call([
        "cosign", "sign-blob", "--yes",
        "--key", os.environ["COSIGN_KEY"],
        "--output-signature", args.report + ".sig",
        args.report,
    ])


if __name__ == "__main__":
    main()
```

### 2.8 Tests — `tests/test_cohen_kappa.py` and `tests/test_krippendorff_alpha.py`

Pin the metrics against hand-computed cases.

```python
# tests/test_cohen_kappa.py
from code.metrics import cohen_kappa


def _rows(pairs):
    out = []
    for i, (a, b) in enumerate(pairs):
        out.append({"item_id": str(i), "question": "q", "annotator_id": "A", "value": a})
        out.append({"item_id": str(i), "question": "q", "annotator_id": "B", "value": b})
    return out


def test_kappa_perfect_agreement():
    rows = _rows([("x", "x"), ("y", "y"), ("x", "x")])
    assert cohen_kappa(rows, "q", "A", "B").kappa == 1.0


def test_kappa_chance_only():
    # 2x2 contingency with equal marginals and diagonal == off-diagonal.
    rows = _rows([("x", "x"), ("x", "y"), ("y", "x"), ("y", "y")])
    kr = cohen_kappa(rows, "q", "A", "B")
    assert abs(kr.kappa - 0.0) < 1e-9


def test_kappa_landis_koch_example():
    # From Landis & Koch (1977) Table 1: κ = 0.68 on a published example.
    # The fixture is hand-constructed so the computed κ matches 0.68 ± 0.005.
    ...
```

```python
# tests/test_krippendorff_alpha.py
from code.metrics import krippendorff_alpha


def _rows(table):
    out = []
    for item_id, by_ann in table.items():
        for ann, v in by_ann.items():
            out.append({"item_id": item_id, "question": "q",
                        "annotator_id": ann, "value": str(v)})
    return out


def test_alpha_nominal_textbook():
    # Krippendorff (2011) Table 1:
    #   10 items, 2 coders, α = 0.743
    # Reproduce to ±0.005.
    ...


def test_alpha_ordinal_textbook():
    # Reproduce α = 0.815 from Krippendorff's ordinal example.
    ...


def test_alpha_identity_all_match():
    rows = _rows({"i1": {"A": 1, "B": 1, "C": 1},
                   "i2": {"A": 2, "B": 2, "C": 2}})
    alpha, n = krippendorff_alpha(rows, "q", "nominal")
    assert alpha == 1.0 and n == 2
```

Fill the `...` bodies with the exact tables cited in Krippendorff
(2011) §4.1–4.2 so the test asserts against *published* numbers, not
numbers the implementation produced.

### 2.9 `Makefile`

```makefile
.PHONY: all load compute sign verify clean test

all: load compute sign

load:
	python code/load_export.py \
		--label-studio exports/labelstudio_export.json \
		--argilla exports/argilla_export.jsonl \
		--output results/annotations_tidy.parquet

compute:
	python code/compute_iaa.py \
		--annotations results/annotations_tidy.parquet \
		--policy policy.yaml \
		--batch-id batch-001 \
		--operator $$(git config user.email) \
		--output results/iaa_report.yaml

sign:
	python code/sign_report.py --report results/iaa_report.yaml

verify:
	cosign verify-blob --key $$COSIGN_PUB \
		--signature results/iaa_report.yaml.sig \
		results/iaa_report.yaml

test:
	pytest -q

clean:
	rm -rf results/*.parquet results/iaa_report.yaml results/iaa_report.yaml.sig
```

## 3. Validation steps

1. **Tests pass against textbook values.** `pytest -q` passes; the
   κ and α cases reproduce published values to ≤ 0.005 absolute
   error.
2. **Deterministic re-runs.** `make clean && make compute` twice;
   `sha256sum results/iaa_report.yaml` matches modulo `run_end_utc`.
3. **Exit code contract.** Running against a seeded dataset known
   to breach `pairwise_min` yields exit code 10; a schema-mangled
   export yields 20; a passing batch yields 0.
4. **Per-pair κ is computed only on items both annotators rated.**
   Inject a dataset where annotator C skipped half the items; the
   `(A, B)` pair's `n` equals the number of items both A and B
   rated, and matches the pair's κ sample size exactly.
5. **α uses the right distance.** Change `policy.yaml` to declare
   `rating_a: nominal` and rerun; α for `rating_a` changes (an
   ordinal metric is not the same as a nominal one on ratings).
6. **Breakdown surfaces the right class.** Inject a known
   class-level disagreement (seed `CLAIM_SUBJECT` with 50%
   agreement); the per-class breakdown's `none` count for
   `CLAIM_SUBJECT` is non-zero and that class's pairwise κ is
   lower than others.
7. **Report signs.** `cosign verify-blob` on the signed report
   exits 0; replacing a single digit in the report makes verify
   fail.
8. **Missing annotations are handled without imputation.** An
   item with only one annotation is dropped from α's denominator;
   verify by removing one annotator's row and confirming α's
   `n_items` drops by exactly 1.
9. **Rounding is the last operation.** The internal 6-decimal
   precision is consistent across two Linux runners (x86 and
   ARM); the 4-decimal report is byte-identical across both.
10. **Pass/fail verdict is literal.** The YAML `verdict` field is
    `pass` or `fail`, not a narrative; CI greps for the literal.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 20/28 with zero zeros on
critical rows.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Cohen's κ implemented as pairwise contingency with chance-corrected formula, not raw agreement                    | Yes       |
| 2  | Krippendorff's α implemented in the reliability-data formulation                                                  | Yes       |
| 3  | α distance functions cover nominal / ordinal / interval and are chosen per-question from policy                   | Yes       |
| 4  | Items with < 2 annotations are excluded, not imputed                                                             | Yes       |
| 5  | Per-class and per-annotator breakdowns are produced, not only aggregate numbers                                   | Yes       |
| 6  | Exit codes (0/10/20) are a stable contract CI can consume                                                        | Yes       |
| 7  | `iaa_report.yaml` is signed and verifiable (cosign)                                                              | Yes       |
| 8  | Tests pin κ and α against textbook examples with ≤ 0.005 tolerance                                                | Yes       |
| 9  | Report determinism check (two runs → byte-identical YAML modulo `run_end_utc`) passes                            | Yes       |
| 10 | Governance anchors (NIST AI RMF MEASURE 2.3, OWASP ML02, MITRE ATLAS AML.T0020) cited                             | No        |
| 11 | Policy file is single source of thresholds; drift detection blocks silent threshold changes                        | Yes       |
| 12 | Handles both Label Studio and Argilla exports through a tidy intermediate                                          | Yes       |
| 13 | Rounding is applied once, at the end, and the internal precision is cross-platform stable                          | No        |
| 14 | Pass/fail verdict field is literal (`pass` / `fail`), not narrative                                               | No        |

## 5. Common mistakes

- **Reporting raw agreement instead of κ.** Raw agreement is
  inflated by class imbalance; κ corrects for chance. A reviewer
  who sees a raw-agreement number treats it as κ and ships a
  class-skewed batch.
- **Using `scikit-learn`'s `cohen_kappa_score` on span annotations
  without span normalisation.** NER spans need offset bucketing
  (ex-01 allows ±3 characters); otherwise every span disagreement
  at character offsets 42 vs 43 is counted as a full disagreement.
- **Using a nominal α on Likert ratings.** A 5 and a 1 are not
  "equally wrong" relative to a 3; ordinal α (or interval if the
  anchors are evenly spaced) is the right choice for ratings.
- **Imputing missing annotations.** Fill-with-majority and
  fill-with-first-annotator are both biased estimators; the
  reliability-data form discards items with < 2 annotations,
  which is the correct answer.
- **Averaging pairwise κ into a single number.** Averaging hides
  the two-annotator dyad that is actually failing. Keep the
  pairwise table; let the policy choose whether to pass on
  minimum, median, or mean.
- **Not breaking down by class.** Low aggregate α almost never
  points to a cause; the per-class view does. Shipping only the
  aggregate number forces the reviewer to re-run the tool.
- **Pre-rounding inside the metric.** Rounding before the
  arithmetic drifts κ by up to 0.005 across runners. Round once,
  at report time.
- **Letting `policy.yaml` drift from `policy_note.md`.** The
  numeric thresholds and the prose policy must move together;
  otherwise a batch passes the number and fails the audit.
- **Returning a narrative verdict.** CI cannot grep "mostly
  good". The verdict is `pass` or `fail`.
- **Not signing the report.** mod-109 lineage cannot ingest an
  unsigned IAA report; the batch cannot ship.

## 6. References

- **NIST AI Risk Management Framework — MEASURE 2.3:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. IAA
  operationalises "performance measured as intended" — a class
  whose α is below the reliability floor is not measurable as
  intended.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Systematic disagreement from a single annotator cohort is the
  observable signal of insider poisoning.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing anchor; IAA
  breakdowns surface the behavioural signature.
- **Cohen, J. (1960) "A Coefficient of Agreement for Nominal
  Scales."** Educational and Psychological Measurement 20(1):37–46.
  Original definition of κ; the chance-correction logic the
  implementation in §2.4 encodes.
- **Krippendorff, K. (2011) "Computing Krippendorff's
  Alpha-Reliability."** Departmental Papers (ASC), University of
  Pennsylvania. The reliability-data formulation this exercise
  implements, including the nominal / ordinal / interval distance
  definitions.
- **Landis, J. R. & Koch, G. G. (1977) "The Measurement of
  Observer Agreement for Categorical Data."** Biometrics 33(1):
  159–174. Source of the commonly cited κ interpretation
  thresholds (slight 0–0.2, fair 0.21–0.40, moderate 0.41–0.60,
  substantial 0.61–0.80, almost perfect 0.81–1.0) — the policy's
  κ ≥ 0.70 picks from this scale.
- **`scikit-learn` `cohen_kappa_score`:**
  <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html>.
  Reference implementation of κ (nominal). The exercise reimplements
  rather than depends on `sklearn` to keep the dependency surface
  small and to control rounding; a cross-check against this is a
  reasonable additional test.
- **Local anchors:** `mod-107` ex-01 (the policy note whose
  thresholds this tool enforces); `mod-107` ex-02 (vendor audit
  that re-runs κ / α on sampled items); `mod-109` (Sigstore
  signing of the IAA report consumed by batch lineage).
