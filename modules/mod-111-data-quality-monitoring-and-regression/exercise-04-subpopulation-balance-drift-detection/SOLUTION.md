# Exercise 04 — Subpopulation Balance Drift Detection (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds a detector
that monitors the balance of categorical subpopulations (language,
domain, label class, source bucket, protected-attribute proxy where
lawfully permitted) in a training or features dataset, and alerts when
the observed mix drifts materially from a pinned reference
distribution. The detector emits a per-subpopulation drift report plus
an advisory rebalance-hints note; it never auto-rewrites the dataset.

Deliverables (taken verbatim from the exercise contract):

- `recipes/subpop_contract.yaml` — declares the reference distribution,
  the subpopulation columns of interest, the drift metrics chosen (PSI
  as primary, chi-square as statistical companion, Jensen-Shannon as
  smooth secondary), per-metric thresholds, a `min_sample_size` gate,
  and a segmentation hierarchy (global, per-source, per-language).
- `code/detector.py` — implements PSI with the standard formula
  `sum((actual% - expected%) * ln(actual% / expected%))`, chi-square via
  `scipy.stats.chi2_contingency`, and Jensen-Shannon via
  `scipy.spatial.distance.jensenshannon`. Applies add-epsilon (Laplace)
  smoothing to zero cells and enforces a `min_sample_size` gate.
- `code/rebalance_hints.py` — advisory module. Given a drift report,
  suggests reweighting / resampling steps; never modifies the dataset.
- `reports/drift_report.md` — a worked example. Reference distribution,
  observed distribution, PSI per subpop, which crossed threshold,
  triage recommendation.
- `tests/test_detector.py` — pytest covering: identical distributions →
  PSI approx 0; total swap of two categories → PSI large; missing
  category in current (zero cells) → smoothing kicks in, detector does
  not NaN; single-row current slice → fails `min_sample_size` gate.

Governance framing. Subpopulation balance monitoring is the dataset-
level signal that catches two different risks at once: (1) *silent
corpus drift*, where an upstream pipeline change tilts the language or
source mix and degrades downstream fairness and accuracy; (2) *targeted
data poisoning* (OWASP ML Top 10 **ML02:2023 Data Poisoning Attack**),
where an adversary injects a disproportionate volume of one
subpopulation to shift decision boundaries. The MITRE ATLAS anchor is
the **ML Supply Chain Compromise** tactic family — curated subpop
balance is one of the few observable surfaces on which supply-chain
tampering of training data shows up. The NIST AI Risk Management
Framework anchors are the **MEASURE** pillar (fairness and bias
evaluation at the dataset level — the **MEASURE 2** subfunction
covers trustworthy-characteristic evaluation, and the balance detector
is the *dataset-level* input to that evaluation; the specific MEASURE
2.x sub-item (e.g., 2.11 harmful bias) that most tightly anchors
subpopulation-balance measurement should be confirmed against the
current RMF revision before being quoted in audit evidence)
and the **MANAGE** pillar
(mechanisms to supersede, disengage, or deactivate when posture
degrades — a major subpop drift is a `severity-major` trigger wired
into the sibling runbook from exercise-03). Cite the exact framework
URLs in the References section rather than paraphrasing the control
text.

A note on proxy columns. Where protected attributes (race, religion,
disability) cannot be stored, the detector operates on *documented
proxies* (language distribution for geographic/cultural balance in
text corpora; source-domain mix where the geography is baked into the
crawl host; an inferred register/formality bucket). The proxy choice
itself is policy-governed and ties back to `mod-101` (dataset
governance) and `mod-110` (the proxy must not reconstruct the
protected attribute to any materially useful precision). The recipe
records which proxy is being used and why; the detector refuses to
run against a proxy column that is not declared in the recipe.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── subpop_contract.yaml        # reference distribution + thresholds
├── code/
│   ├── detector.py                 # PSI, chi-square, JS
│   ├── rebalance_hints.py          # advisory re-weight / re-sample suggestions
│   ├── report.py                   # markdown + JSON report writer
│   └── __init__.py
├── reports/
│   └── drift_report.md             # worked example output
├── tests/
│   ├── test_detector.py
│   └── fixtures/
│       └── tiny_reference.json     # frozen reference for identity test
└── outputs/
    ├── drift_report.json           # machine-readable sibling of the markdown
    └── drift_report.md
```

### 2.2 `recipes/subpop_contract.yaml`

```yaml
# recipes/subpop_contract.yaml
# Subpopulation balance contract. Every knob the detector reads lives
# here; the detector refuses to run with CLI overrides for anything in
# this file except --current (the dataset snapshot to score) and
# --run-id.

dataset:
  id: training-slice-v7
  # Reference distribution is pinned by hash. If the reference file
  # drifts on disk, the detector aborts before scoring — otherwise
  # "drift" is just a reference change in disguise.
  reference_path: references/training-slice-v7.reference.json
  reference_sha256: <64-hex-digest>
  # The window over which the reference was computed. Documented here
  # so a reviewer can answer "why this reference".
  reference_window:
    start_utc: 2026-07-01T00:00:00Z
    end_utc:   2026-07-31T23:59:59Z
    rows:      4_812_300

# Columns the detector inspects. Each is classified as:
#   - direct         : the column *is* the subpop axis (e.g., label class)
#   - proxy          : the column stands in for an attribute that cannot
#                      be stored directly; policy_ref is mandatory
#   - operational    : source bucket, pipeline stage — not a fairness axis
#                      but balance here signals upstream pipeline drift
subpopulations:
  - name: language
    column: lang_id
    type: proxy
    policy_ref: mod-101/exercise-02/decisions/proxies.md#language
    # Fold rare languages into "other" to keep the chi-square table
    # from exploding. The threshold is deliberately conservative.
    rare_bucket:
      threshold_pct: 0.5
      other_label: "other"
  - name: source_domain
    column: source_bucket
    type: operational
    rare_bucket:
      threshold_pct: 1.0
      other_label: "other"
  - name: label_class
    column: label
    type: direct
    # Zero is a valid class; no rare-bucket folding.

metrics:
  # PSI is the primary signal. The thresholds below are *industry
  # convention* (no-change <0.1, moderate 0.1–0.25, major >0.25),
  # not universal law; record the convention as an explicit assumption.
  psi:
    enabled: true
    smoothing_epsilon: 1.0e-4   # add-epsilon (Laplace) for zero cells
    thresholds:
      no_change: 0.10
      moderate:  0.25
      major:     1.00   # ceiling only for severity classification
  # Chi-square is the statistical companion. Use the p-value as a
  # secondary gate, not the primary. With very large n, chi-square will
  # fire on tiny effects; keep it as a confirmatory signal.
  chi_square:
    enabled: true
    alpha: 0.01
    min_expected_cell: 5
  # Jensen-Shannon is bounded in [0, 1] (log base 2) and is smoother
  # than PSI for small shifts; keep it as a secondary.
  jensen_shannon:
    enabled: true
    base: 2
    thresholds:
      moderate: 0.05
      major:    0.15

# Guardrails.
gates:
  # The detector refuses to score a (subpop, segment) slice whose
  # current row count is below this. Below this, PSI is noise.
  min_sample_size: 500
  # If the number of distinct categories in `current` differs from
  # `reference` by more than this fraction, flag "category structure
  # drift" independently of PSI.
  category_structure_delta: 0.10

# Segmentation hierarchy. Report at each level; PSI is computed per
# subpop at each level separately.
segmentation:
  - global
  - by: source_bucket
  - by: lang_id

# Where results land. Both files are emitted per run; the JSON is for
# downstream alerting and the markdown is for humans.
outputs:
  report_md:   outputs/drift_report.md
  report_json: outputs/drift_report.json

# Severity-to-action mapping. The detector never auto-rewrites the
# dataset; it only recommends. See code/rebalance_hints.py.
severity_actions:
  no_change: log_only
  moderate:  open_ticket
  major:     page_oncall_and_block_training
```

Why each knob matters at review:

- **`reference_sha256` is a hard gate.** Without it, a silent change
  to the reference file is indistinguishable from a real drift — and
  the silent change is the more common failure mode.
- **`smoothing_epsilon` pinned in the recipe.** Different values give
  materially different PSI for zero cells; reviewers want to see the
  number, not infer it.
- **`min_sample_size` non-negotiable.** PSI computed on a few dozen
  rows is high-variance; enforce in code, refuse to score below.
- **PSI thresholds stated as convention.** The `0.1 / 0.25` bands are
  widely cited (Siddiqi's credit-scoring literature popularised them)
  but are not a universal law — keep them configurable per recipe.
  Treat the 0.1 / 0.25 bands as convention; some texts cite 0.1 / 0.2,
  and a primary-source citation is a reviewer-owned follow-up.
- **`min_expected_cell: 5`** is the standard chi-square rule of
  thumb; below it the chi-square approximation is untrustworthy.

### 2.3 `code/detector.py`

```python
# code/detector.py
# Subpopulation balance drift detector.
# Inputs:
#   - recipe (recipes/subpop_contract.yaml)
#   - reference distribution (frozen JSON)
#   - current dataset snapshot (parquet or jsonl)
# Outputs:
#   - outputs/drift_report.json (machine)
#   - outputs/drift_report.md   (human)
from __future__ import annotations

import argparse
import hashlib
import json
import logging
import math
from collections import Counter
from dataclasses import asdict, dataclass, field
from pathlib import Path
from typing import Any, Iterable

import numpy as np
import pyarrow.parquet as pq
import yaml
from scipy.spatial.distance import jensenshannon
from scipy.stats import chi2_contingency

LOG = logging.getLogger("subpop-detector")


# ---- data classes -----------------------------------------------------------

@dataclass(slots=True)
class SubpopResult:
    subpop_name: str
    segment: str                # e.g. "global", "source_bucket=crawl-a"
    n_reference: int
    n_current: int
    category_count_reference: int
    category_count_current: int
    categories: list[str]
    reference_pct: dict[str, float]
    current_pct: dict[str, float]
    psi: float | None
    chi_square_stat: float | None
    chi_square_p: float | None
    chi_square_dof: int | None
    jensen_shannon: float | None
    smoothing_applied: bool
    gated: bool
    gate_reason: str | None
    severity: str               # "no_change" | "moderate" | "major" | "gated"

    def to_dict(self) -> dict[str, Any]:
        return asdict(self)


# ---- metric primitives ------------------------------------------------------

def _align_categories(
    reference: dict[str, int],
    current: dict[str, int],
) -> tuple[list[str], np.ndarray, np.ndarray]:
    """Return (categories, ref_counts, cur_counts) over the union of keys.

    Missing categories materialise as zero. The caller decides whether
    to smooth.
    """
    cats = sorted(set(reference) | set(current))
    ref = np.array([reference.get(c, 0) for c in cats], dtype=np.float64)
    cur = np.array([current.get(c, 0) for c in cats], dtype=np.float64)
    return cats, ref, cur


def _normalise(counts: np.ndarray, eps: float) -> tuple[np.ndarray, bool]:
    """Convert counts to proportions. If any zero cells exist, apply
    add-epsilon smoothing *before* normalising, so neither PSI nor
    Jensen-Shannon has to defend against log(0).

    Returns (proportions, smoothing_applied).
    """
    had_zero = bool(np.any(counts == 0))
    if had_zero:
        counts = counts + eps
    total = counts.sum()
    if total <= 0:
        # degenerate; upstream gate should have caught it
        return np.full_like(counts, 1.0 / len(counts)), True
    return counts / total, had_zero


def psi(
    reference_counts: dict[str, int],
    current_counts: dict[str, int],
    eps: float,
) -> tuple[float, bool, list[str]]:
    """Population Stability Index.

    PSI = sum((cur_pct - ref_pct) * ln(cur_pct / ref_pct))

    Smoothing (add-eps on raw counts, applied uniformly to both
    distributions when either has a zero cell) is what keeps PSI
    finite when a category is absent on one side.
    """
    cats, ref, cur = _align_categories(reference_counts, current_counts)
    ref_p, ref_smoothed = _normalise(ref, eps)
    cur_p, cur_smoothed = _normalise(cur, eps)
    smoothed = ref_smoothed or cur_smoothed
    # avoid log(0) regardless — smoothing should have done it, this is
    # a belt-and-braces clamp.
    ref_p = np.clip(ref_p, eps, 1.0)
    cur_p = np.clip(cur_p, eps, 1.0)
    value = float(np.sum((cur_p - ref_p) * np.log(cur_p / ref_p)))
    return value, smoothed, cats


def chi_square(
    reference_counts: dict[str, int],
    current_counts: dict[str, int],
    min_expected_cell: int,
) -> tuple[float | None, float | None, int | None, bool]:
    """Chi-square test of independence on the 2xK contingency table.

    Returns (stat, p, dof, degenerate_flag). `degenerate_flag` is True
    when any expected cell < min_expected_cell — the test's
    approximation is untrustworthy there and the detector records the
    numbers but does not act on them.
    """
    _, ref, cur = _align_categories(reference_counts, current_counts)
    table = np.vstack([ref, cur])
    if np.any(table.sum(axis=1) == 0) or np.any(table.sum(axis=0) == 0):
        return None, None, None, True
    stat, p, dof, expected = chi2_contingency(table, correction=False)
    degenerate = bool(np.any(expected < min_expected_cell))
    return float(stat), float(p), int(dof), degenerate


def js_distance(
    reference_counts: dict[str, int],
    current_counts: dict[str, int],
    eps: float,
    base: float = 2.0,
) -> float:
    """Jensen-Shannon distance (sqrt of the JS divergence).

    scipy.spatial.distance.jensenshannon returns the distance (sqrt
    of the divergence). Clamp to [0, 1] when base=2; small numerical
    bleed past 1.0 happens for near-orthogonal distributions.
    """
    _, ref, cur = _align_categories(reference_counts, current_counts)
    ref_p, _ = _normalise(ref, eps)
    cur_p, _ = _normalise(cur, eps)
    d = float(jensenshannon(ref_p, cur_p, base=base))
    if math.isnan(d):
        # happens iff one side is all-zero after normalise, which
        # smoothing should have prevented; defensive clamp.
        return 0.0
    return min(d, 1.0)


# ---- severity ---------------------------------------------------------------

def _severity_from_psi(value: float, thr: dict[str, float]) -> str:
    if value < thr["no_change"]:
        return "no_change"
    if value < thr["moderate"]:
        return "moderate"
    return "major"


# ---- top-level scorer -------------------------------------------------------

def score_subpop(
    subpop_name: str,
    segment: str,
    reference_counts: dict[str, int],
    current_counts: dict[str, int],
    cfg: dict,
) -> SubpopResult:
    gate = cfg["gates"]
    n_ref = int(sum(reference_counts.values()))
    n_cur = int(sum(current_counts.values()))

    # Gate: refuse to score tiny slices. PSI on a handful of rows is
    # noise. Downstream alerting ignores gated rows.
    if n_cur < gate["min_sample_size"]:
        return SubpopResult(
            subpop_name=subpop_name,
            segment=segment,
            n_reference=n_ref,
            n_current=n_cur,
            category_count_reference=len(reference_counts),
            category_count_current=len(current_counts),
            categories=sorted(set(reference_counts) | set(current_counts)),
            reference_pct={}, current_pct={},
            psi=None, chi_square_stat=None, chi_square_p=None,
            chi_square_dof=None, jensen_shannon=None,
            smoothing_applied=False, gated=True,
            gate_reason=(
                f"n_current={n_cur} < min_sample_size="
                f"{gate['min_sample_size']}"),
            severity="gated",
        )

    psi_cfg = cfg["metrics"]["psi"]
    chi_cfg = cfg["metrics"]["chi_square"]
    js_cfg  = cfg["metrics"]["jensen_shannon"]

    psi_val, smoothed, cats = psi(
        reference_counts, current_counts, psi_cfg["smoothing_epsilon"],
    )
    chi_stat, chi_p, chi_dof, chi_degen = chi_square(
        reference_counts, current_counts, chi_cfg["min_expected_cell"],
    )
    js_val = js_distance(
        reference_counts, current_counts,
        psi_cfg["smoothing_epsilon"], base=js_cfg["base"],
    )

    ref_total = max(1, n_ref)
    cur_total = max(1, n_cur)
    ref_pct = {c: reference_counts.get(c, 0) / ref_total for c in cats}
    cur_pct = {c: current_counts.get(c,  0) / cur_total for c in cats}

    severity = _severity_from_psi(psi_val, psi_cfg["thresholds"])

    # Category-structure delta is a second-order signal: a large change
    # in the *number* of distinct categories is worth flagging even if
    # PSI happens to be modest (e.g., many tiny new categories that all
    # land in the rare-bucket).
    ref_cats = set(c for c, n in reference_counts.items() if n > 0)
    cur_cats = set(c for c, n in current_counts.items()  if n > 0)
    if ref_cats:
        delta = abs(len(cur_cats) - len(ref_cats)) / len(ref_cats)
        if delta > cfg["gates"]["category_structure_delta"] \
                and severity == "no_change":
            severity = "moderate"

    return SubpopResult(
        subpop_name=subpop_name,
        segment=segment,
        n_reference=n_ref,
        n_current=n_cur,
        category_count_reference=len(ref_cats),
        category_count_current=len(cur_cats),
        categories=cats,
        reference_pct=ref_pct,
        current_pct=cur_pct,
        psi=psi_val,
        chi_square_stat=chi_stat,
        chi_square_p=chi_p,
        chi_square_dof=chi_dof,
        jensen_shannon=js_val,
        smoothing_applied=smoothed,
        gated=False,
        gate_reason=None,
        severity=severity,
    )


# ---- IO ---------------------------------------------------------------------

def load_recipe(path: Path) -> dict:
    cfg = yaml.safe_load(path.read_text())
    return cfg


def load_reference(path: Path, expected_sha256: str) -> dict[str, dict[str, int]]:
    data = path.read_bytes()
    actual = hashlib.sha256(data).hexdigest()
    if actual != expected_sha256:
        raise SystemExit(
            f"reference drift: recipe says {expected_sha256}, "
            f"on disk is {actual}"
        )
    return json.loads(data)


def read_current(path: Path, columns: Iterable[str]) -> dict[str, list]:
    if path.suffix == ".parquet":
        tbl = pq.read_table(path, columns=list(columns))
        return {c: tbl.column(c).to_pylist() for c in columns}
    rows = [json.loads(l) for l in path.read_text().splitlines() if l.strip()]
    return {c: [r.get(c) for r in rows] for c in columns}


def count(values: Iterable) -> dict[str, int]:
    c = Counter(str(v) if v is not None else "__null__" for v in values)
    return dict(c)


# ---- entrypoint -------------------------------------------------------------

def run(cfg_path: Path, current_path: Path, run_id: str,
        out_json: Path, out_md: Path) -> list[SubpopResult]:
    cfg = load_recipe(cfg_path)
    ref_counts_all = load_reference(
        Path(cfg["dataset"]["reference_path"]),
        cfg["dataset"]["reference_sha256"],
    )

    cols = [sp["column"] for sp in cfg["subpopulations"]]
    seg_cols = [s["by"] for s in cfg["segmentation"] if isinstance(s, dict)]
    data = read_current(current_path, set(cols) | set(seg_cols))

    results: list[SubpopResult] = []
    for sp in cfg["subpopulations"]:
        col = sp["column"]
        ref = ref_counts_all.get(sp["name"], {}).get("global", {})
        cur = count(data[col])
        results.append(score_subpop(
            sp["name"], "global", ref, cur, cfg))

        for seg in cfg["segmentation"]:
            if not isinstance(seg, dict):
                continue
            seg_col = seg["by"]
            seg_values = sorted(set(data[seg_col]))
            for sv in seg_values:
                mask = [v == sv for v in data[seg_col]]
                sub_vals = [v for v, m in zip(data[col], mask) if m]
                sub_cur = count(sub_vals)
                sub_ref = (ref_counts_all.get(sp["name"], {})
                                        .get(f"{seg_col}={sv}", {}))
                results.append(score_subpop(
                    sp["name"], f"{seg_col}={sv}", sub_ref, sub_cur, cfg))

    out_json.parent.mkdir(parents=True, exist_ok=True)
    out_json.write_text(json.dumps(
        {"run_id": run_id,
         "recipe_sha256": hashlib.sha256(cfg_path.read_bytes()).hexdigest(),
         "results": [r.to_dict() for r in results]},
        indent=2, sort_keys=True))
    out_md.write_text(_render_markdown(cfg, results, run_id))
    return results


def _render_markdown(cfg: dict, results: list[SubpopResult],
                     run_id: str) -> str:
    # Rendering kept in report.py in the full source tree; inlined here
    # for the solution writeup. See code/report.py for the production
    # version.
    lines = [f"# Subpopulation balance drift — {run_id}", ""]
    lines.append(f"Reference: `{cfg['dataset']['reference_path']}`  ")
    lines.append(f"Recipe:    `recipes/subpop_contract.yaml`  ")
    lines.append("")
    lines.append("| Subpop | Segment | n_ref | n_cur | PSI | JS | chi2 p | severity |")
    lines.append("|--------|---------|------:|------:|----:|---:|-------:|:---------|")
    for r in results:
        psi_s = "gated" if r.gated else f"{r.psi:.4f}"
        js_s  = "—" if r.jensen_shannon is None else f"{r.jensen_shannon:.4f}"
        p_s   = "—" if r.chi_square_p is None else f"{r.chi_square_p:.2e}"
        lines.append(
            f"| {r.subpop_name} | {r.segment} | {r.n_reference} | "
            f"{r.n_current} | {psi_s} | {js_s} | {p_s} | {r.severity} |"
        )
    return "\n".join(lines) + "\n"


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config",   required=True, type=Path)
    ap.add_argument("--current",  required=True, type=Path)
    ap.add_argument("--run-id",   required=True)
    ap.add_argument("--out-json", default=Path("outputs/drift_report.json"),
                    type=Path)
    ap.add_argument("--out-md",   default=Path("outputs/drift_report.md"),
                    type=Path)
    args = ap.parse_args()
    run(args.config, args.current, args.run_id, args.out_json, args.out_md)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
```

Design notes defensible at review:

- **Smoothing lives inside `_normalise`,** so every metric that reaches
  for proportions goes through the same path and smoothing is applied
  symmetrically (asymmetric smoothing biases PSI away from zero even
  on identical distributions).
- **Chi-square uses `correction=False`.** Yates' correction is only
  defined for 2x2 tables; the detector runs on 2xK and the correction
  is undefined for K>2.
- **`min_sample_size` is checked before any metric is computed.** PSI
  on `n_current=37` is theatre, not evidence.
- **Null is a category.** `None` values flow to `"__null__"` so a
  surge of nulls — a classic pipeline failure — shows up as a subpop
  shift rather than disappearing silently.
- **Category-structure delta is a secondary promotion to `moderate`.**
  A dataset where PSI is 0.08 but the number of distinct source
  domains doubled is more interesting than PSI alone implies.

### 2.4 `code/rebalance_hints.py`

```python
# code/rebalance_hints.py
# Advisory only. Given a drift report, suggest per-subpop reweighting
# or resampling steps. The detector never modifies the dataset; it
# can only recommend. Every hint carries a human review gate.
from __future__ import annotations

import json
from dataclasses import dataclass
from pathlib import Path
from typing import Any


@dataclass(slots=True)
class Hint:
    subpop_name: str
    segment: str
    severity: str
    kind: str           # "reweight" | "oversample" | "undersample" | "investigate"
    category: str | None
    current_pct: float | None
    reference_pct: float | None
    weight_suggestion: float | None
    rationale: str
    approvals_required: list[str]

    def to_dict(self) -> dict[str, Any]:
        return self.__dict__


REWEIGHT_CLIP = (0.25, 4.0)    # never suggest more aggressive than 4x


def suggest(result: dict) -> list[Hint]:
    if result["gated"] or result["severity"] == "no_change":
        return []

    hints: list[Hint] = []
    ref = result["reference_pct"]
    cur = result["current_pct"]

    # The categories that moved the needle on PSI are the biggest
    # (|cur - ref|) movers. Restrict to the top 3 per subpop so the
    # review burden stays bounded.
    movers = sorted(
        ref.keys(),
        key=lambda c: abs(cur.get(c, 0) - ref.get(c, 0)),
        reverse=True,
    )[:3]

    for cat in movers:
        rp = ref.get(cat, 0.0)
        cp = cur.get(cat, 0.0)
        if rp == 0 and cp == 0:
            continue
        if cp == 0:
            # Category vanished — reweighting cannot bring it back.
            hints.append(Hint(
                subpop_name=result["subpop_name"],
                segment=result["segment"],
                severity=result["severity"],
                kind="investigate",
                category=cat,
                current_pct=cp,
                reference_pct=rp,
                weight_suggestion=None,
                rationale=(
                    "category present in reference but absent from current; "
                    "investigate upstream pipeline before any resampling."
                ),
                approvals_required=["data-eng", "ds-owner"],
            ))
            continue
        # Reweighting toward the reference is the standard move; cap
        # the weight so a vanishing category does not blow up into a
        # thousand-x reweight.
        raw = rp / cp if cp > 0 else None
        w = None if raw is None else max(REWEIGHT_CLIP[0], min(REWEIGHT_CLIP[1], raw))
        kind = "oversample" if (raw is not None and raw > 1.0) else "undersample"
        hints.append(Hint(
            subpop_name=result["subpop_name"],
            segment=result["segment"],
            severity=result["severity"],
            kind=kind,
            category=cat,
            current_pct=cp,
            reference_pct=rp,
            weight_suggestion=w,
            rationale=(
                f"category {cat!r} has drifted from "
                f"{rp:.2%} (ref) to {cp:.2%} (cur); "
                f"suggest {kind} weight clipped to {REWEIGHT_CLIP}."
            ),
            approvals_required=(
                ["ds-owner"] if result["severity"] == "moderate"
                else ["ds-owner", "fairness-reviewer"]
            ),
        ))

    return hints


def main() -> None:
    import argparse
    ap = argparse.ArgumentParser()
    ap.add_argument("--drift-json", required=True, type=Path)
    ap.add_argument("--out-json",   required=True, type=Path)
    args = ap.parse_args()
    report = json.loads(args.drift_json.read_text())
    out = []
    for res in report["results"]:
        out.extend(h.to_dict() for h in suggest(res))
    args.out_json.write_text(json.dumps(
        {"run_id": report["run_id"], "hints": out}, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Design notes: hints are advisory and never auto-applied — the module
writes a JSON that humans review. Reweight is clipped to `[0.25, 4.0]`
(a category that nearly vanished with a 10% reference share implies a
100x weight, enough to overfit on a handful of rows and masquerade as
balance; the clip is policy, not tuning). Vanished categories get an
`investigate` hint, not a weight. Approvals scale with severity:
`moderate` is dataset-owner only; `major` adds a fairness reviewer
whose role is to check the reweight does not re-import a
protected-attribute signal.

### 2.5 `reports/drift_report.md` (worked example)

```markdown
# Subpopulation balance drift — run 2026-11-24.01

Reference: `references/training-slice-v7.reference.json`  
Recipe:    `recipes/subpop_contract.yaml`  
Current:   `snapshots/2026-11-24.parquet` (rows: 1,204,788)

## Headline

- `language` global subpop: **PSI = 0.31 → severity = major**.
  `en` share fell from 68.2% → 59.1%; `de` share rose from 4.1% →
  11.9%. Investigate upstream crawler bucketing before training
  resumes.
- `label_class` global: **PSI = 0.06 → no_change**.
- `source_domain` segment `source_bucket=crawl-c`: **gated**
  (`n_current = 312 < min_sample_size = 500`). Report but do not act.

## Per-subpop table

| Subpop        | Segment                 | n_ref      | n_cur     | PSI    | JS    | chi2 p    | severity   |
|---------------|-------------------------|-----------:|----------:|-------:|------:|----------:|------------|
| language      | global                  | 4,812,300  | 1,204,788 | 0.3104 | 0.182 | <1e-300   | major      |
| language      | source_bucket=crawl-a   |   983,120  |   241,009 | 0.0812 | 0.087 | 2.1e-120  | no_change  |
| language      | source_bucket=crawl-b   | 1,901,200  |   475,101 | 0.2214 | 0.141 | <1e-300   | moderate   |
| language      | source_bucket=crawl-c   |   712,980  |       312 | —      | —     | —         | gated      |
| label_class   | global                  | 4,812,300  | 1,204,788 | 0.0598 | 0.044 | 4.0e-14   | no_change  |
| source_domain | global                  | 4,812,300  | 1,204,788 | 0.1902 | 0.128 | <1e-300   | moderate   |

## Movers (top three per fired subpop)

- `language.global`:
  - `en`: 68.2% → 59.1% (undersample, weight 1.15)
  - `de`: 4.1% → 11.9% (oversample clipped to 0.34)
  - `fr`: 6.0% → 4.7% (oversample, weight 1.28)
- `source_domain.global`:
  - `crawl-b`: 39.5% → 44.0% (undersample, weight 0.90)
  - `crawl-c`: 14.8% → 0.03% — **vanished**. Investigate; do not
    reweight.

## Triage recommendation

1. **Block training promotion** of this slice (`severity=major` on
   a declared proxy subpopulation). Wire to the exercise-03
   runbook's major-drift path.
2. **Open `crawl-c` investigation ticket.** A full subpopulation
   vanishing is almost never a drift; it is a pipeline bug. Owner:
   data-eng. SLA: 24h.
3. **Do not auto-rebalance `de` upward.** The share jump from 4% →
   12% is large enough that the proxy risk (does the German bucket
   over-represent a demographic that cannot be stored?) needs a
   fairness-reviewer sign-off before any reweight is applied.
4. **Re-score next ingest window** after the crawler is fixed. Keep
   the current reference pinned; if the drift persists after the
   pipeline bug is closed, the reference itself needs a versioned
   update (new `training-slice-v8.reference.json`).
```

### 2.6 `tests/test_detector.py`

```python
# tests/test_detector.py
from __future__ import annotations

import math

import pytest

from code.detector import (
    psi, chi_square, js_distance, score_subpop,
)


CFG = {
    "gates": {
        "min_sample_size": 500,
        "category_structure_delta": 0.10,
    },
    "metrics": {
        "psi": {
            "smoothing_epsilon": 1e-4,
            "thresholds": {"no_change": 0.10, "moderate": 0.25, "major": 1.0},
        },
        "chi_square": {"alpha": 0.01, "min_expected_cell": 5},
        "jensen_shannon": {"base": 2, "thresholds": {"moderate": 0.05, "major": 0.15}},
    },
}


def test_identical_distributions_psi_zero() -> None:
    ref = {"en": 7000, "de": 2000, "fr": 1000}
    cur = dict(ref)
    value, smoothed, _ = psi(ref, cur, eps=1e-4)
    assert value == pytest.approx(0.0, abs=1e-12)
    assert smoothed is False


def test_total_swap_psi_is_large() -> None:
    # Two-category flip: 90/10 -> 10/90. PSI should be well above the
    # "major" band (> 0.25) and the JS distance should be bounded in
    # [0, 1].
    ref = {"a": 9000, "b": 1000}
    cur = {"a": 1000, "b": 9000}
    value, smoothed, _ = psi(ref, cur, eps=1e-4)
    assert value > 1.0
    js = js_distance(ref, cur, eps=1e-4, base=2.0)
    assert 0.0 <= js <= 1.0
    assert js > 0.5
    assert smoothed is False


def test_missing_category_triggers_smoothing() -> None:
    # Category "fr" present in reference, absent in current. Without
    # smoothing this would be log(0); the detector must emit a finite
    # PSI and flag smoothing_applied=True.
    ref = {"en": 7000, "de": 2000, "fr": 1000}
    cur = {"en": 7500, "de": 2500}
    value, smoothed, _ = psi(ref, cur, eps=1e-4)
    assert math.isfinite(value)
    assert smoothed is True
    # And it must be a non-trivially large signal, because a missing
    # category is a real drift.
    assert value > 0.05


def test_min_sample_size_gate_blocks_scoring() -> None:
    # Single-row current slice: the gate must fire and the result must
    # not expose a PSI number (which would be noise).
    ref = {"en": 7000, "de": 3000}
    cur = {"en": 1}
    res = score_subpop("language", "global", ref, cur, CFG)
    assert res.gated is True
    assert res.severity == "gated"
    assert res.psi is None
    assert "min_sample_size" in (res.gate_reason or "")


def test_chi_square_flags_degenerate_expected_cells() -> None:
    # 2xK table with an all-zero column after alignment should be
    # reported as degenerate (None metrics), not crash.
    ref = {"a": 100, "b": 0, "c": 50}
    cur = {"a": 50,  "b": 0, "c": 25}
    stat, p, dof, degen = chi_square(ref, cur, min_expected_cell=5)
    # column "b" is all-zero -> we reject before scipy sees it
    assert stat is None and p is None and dof is None
    assert degen is True


def test_category_structure_delta_promotes_to_moderate() -> None:
    # PSI modest (< 0.10) but the number of distinct categories moves
    # more than the structure-delta threshold -> severity promoted.
    ref = {f"c{i}": 1000 for i in range(10)}
    cur = {f"c{i}": 1000 for i in range(10)}
    # Add many tiny new categories that each contribute almost nothing
    # to PSI but change the category count by >10%.
    for i in range(10, 13):
        cur[f"c{i}"] = 1
    res = score_subpop("source_domain", "global", ref, cur, CFG)
    assert res.gated is False
    # PSI alone would be < 0.10 but structure delta is +30%
    assert res.severity in {"moderate", "major"}
```

Design notes: the identity test uses `abs=1e-12` (numerical slack,
not a modelling knob). The swap test asserts the *band* (`> 1.0`),
not an exact value — the exact PSI depends on which smoothing path
fires. The smoothing test asserts `smoothing_applied=True` because a
detector that silently smooths looks the same as one that cannot see
zero cells; the flag is what makes smoothing auditable. The
`min_sample_size` test asserts `psi is None` — emitting a value on a
gated slice is worse than emitting none, because downstream alerting
would act on it.

## 3. Validation steps

1. **Reference hash matches the recipe.** `sha256sum
   references/training-slice-v7.reference.json` equals
   `dataset.reference_sha256` in `recipes/subpop_contract.yaml`. A
   drift aborts the detector at startup.
2. **Recipe loads cleanly.** `python -c "import yaml, pathlib;
   yaml.safe_load(pathlib.Path('recipes/subpop_contract.yaml').read_text())"`
   exits 0; every subpopulation entry has a `column`, `type`, and
   (for `type=proxy`) a `policy_ref`.
3. **Detector runs end-to-end on a worked snapshot.** `python
   code/detector.py --config recipes/subpop_contract.yaml --current
   snapshots/2026-11-24.parquet --run-id bench` produces both
   `outputs/drift_report.json` and `outputs/drift_report.md`.
4. **Identity is identity.** `python code/detector.py --current
   references/training-slice-v7.reference.json …` (feeding the
   reference back as the current snapshot) emits `severity=no_change`
   for every non-gated row and `psi ≈ 0` to within 1e-9.
5. **Gated rows are honoured.** Every row in
   `outputs/drift_report.json` with `gated=true` also has `psi=null`,
   `severity="gated"`, and a human-readable `gate_reason`.
6. **Smoothing is auditable.** Every result with a zero cell in either
   distribution has `smoothing_applied=true` in the JSON; no result
   has `psi=NaN` or `psi=Infinity`.
7. **Chi-square degeneracy is handled.** For any subpop where
   `chi_square_stat` is `null`, the JSON records a degenerate reason
   rather than crashing the detector.
8. **Rebalance hints never auto-apply.** `python
   code/rebalance_hints.py --drift-json outputs/drift_report.json
   --out-json outputs/rebalance_hints.json` writes a hints file; it
   does not touch the dataset, and the pipeline has no hook that reads
   it automatically.
9. **PSI thresholds are configurable.** Swapping
   `metrics.psi.thresholds.moderate` from 0.25 to 0.20 changes the
   `severity` column on at least one row of the worked example
   without any code change.
10. **Unit tests pass.** `pytest tests/` passes, including the
    identity, swap, smoothing, min-sample-size, chi-square-degeneracy,
    and category-structure-delta tests.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `recipes/subpop_contract.yaml` pins reference path + hash, subpops, metric thresholds, `min_sample_size`             | Yes       |
| 2  | Every `type: proxy` subpop carries a `policy_ref` pointing at a governance decision                                  | Yes       |
| 3  | PSI implementation matches `sum((cur_pct - ref_pct) * ln(cur_pct / ref_pct))`                                        | Yes       |
| 4  | Add-epsilon smoothing is applied symmetrically to both distributions and recorded with `smoothing_applied`           | Yes       |
| 5  | `min_sample_size` gate refuses to score and emits a human-readable `gate_reason`                                     | Yes       |
| 6  | Chi-square declines to score degenerate tables (zero rows / zero cols / expected-cell < threshold)                   | Yes       |
| 7  | Jensen-Shannon implemented via `scipy.spatial.distance.jensenshannon` with explicit `base`                           | No        |
| 8  | `code/rebalance_hints.py` is advisory; no automated hook applies the hints to the dataset                            | Yes       |
| 9  | Reweight suggestions are clipped to a documented range; vanished categories get an `investigate` hint, not a weight  | Yes       |
| 10 | Nulls in subpop columns flow to an explicit `__null__` bucket; a surge of nulls shows up as drift                    | Yes       |
| 11 | Worked `reports/drift_report.md` shows reference pct, current pct, PSI, severity, and triage                         | No        |
| 12 | PSI thresholds (<0.1 / 0.1–0.25 / >0.25) are stated as *convention*, not law, in the recipe and the writeup          | Yes       |
| 13 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS, NIST AI RMF MEASURE + MANAGE) cited with URLs                 | No        |
| 14 | `pytest tests/` passes (identity, swap, smoothing, min_sample_size, chi-square degeneracy)                           | Yes       |
| 15 | Report emitted in both JSON (alerting) and markdown (humans); both contain the same row set                          | No        |

## 5. Common mistakes

- **Asymmetric smoothing.** Applying `+eps` only to the side with the
  zero cell biases PSI away from zero even for identical
  distributions. Smooth both distributions whenever either has a zero.
- **Scoring on tiny slices.** PSI on a few hundred rows is dominated
  by sampling noise; the first alert on a per-language-per-source
  slice will be a spurious "major". Enforce `min_sample_size` in code,
  not in a wiki page.
- **Treating the chi-square p-value as the primary signal.** With
  dataset-sized `n`, chi-square will fire on effects too small to
  matter. Keep chi-square as a confirmatory secondary; PSI is the
  effect-size signal.
- **Hard-coding the "0.1 / 0.25" PSI thresholds as universal law.**
  They are a widely cited *convention* (credit-scoring literature
  popularised them); state the convention, keep it configurable,
  document the choice in the recipe.
- **Silently dropping null values.** A pipeline bug that fills a
  subpop column with nulls makes the drift *invisible* if nulls are
  skipped. Route nulls to `__null__` and let them show up in the
  table.
- **Rebalancing automatically on the detector's output.** The detector
  does not know whether the drift is a pipeline bug (fix the bug, do
  not reweight) or a legitimate shift (reweight with fairness review)
  or a poisoning attempt (quarantine, do not reweight). Hints are
  advisory; humans decide.
- **Scoring against a reference file that is not hash-pinned.** If the
  reference can drift on disk, every alert is ambiguous — "is this
  drift in the data or in the reference?". Hash-pin the reference.
- **Running on a protected attribute directly.** Even where lawful,
  the detector is on the audit path for `mod-110`. Use a *declared
  proxy* (language for cultural balance, source bucket for geography)
  and record the proxy choice in the recipe so the governance review
  can see it.
- **Collapsing rare categories without recording it.** A 0.5%
  `rare_bucket.threshold_pct` with no record in the report means a
  reviewer cannot tell whether a "no change" came from stability or
  from a too-aggressive rare-bucket fold. Emit the fold in the JSON.
- **Relying on a single metric.** PSI, chi-square, and JS each have
  different failure modes (PSI is unbounded and sensitive to zero
  cells; chi-square fires on trivial effects at scale; JS is smoother
  but bounded, hiding extreme shifts near the cap). Report all three
  and act on the convergence.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Targeted subpopulation injection is one observable surface of a
  poisoning campaign; the balance detector is the dataset-level
  signal that makes it visible.
- **MITRE ATLAS:** <https://atlas.mitre.org/>. Supply-chain compromise
  of training data manifests as subpopulation-mix shifts upstream of
  training; balance monitoring is the detective control.
- **NIST AI Risk Management Framework — MEASURE pillar (fairness and
  bias evaluation at the dataset level) and MANAGE pillar
  (deactivation when posture degrades):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. The
  subpop balance detector is the dataset-level input to the MEASURE
  pillar's trustworthy-characteristic evaluation; the severity →
  action mapping in the recipe is the MANAGE-pillar wiring.
- **scipy.stats:** <https://docs.scipy.org/doc/scipy/reference/stats.html>.
  `chi2_contingency` is the reference implementation used by the
  detector; the degree-of-freedom and expected-cell semantics come
  from this doc.
- **scipy.spatial.distance.jensenshannon:**
  <https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.distance.jensenshannon.html>.
  Returns the Jensen-Shannon *distance* (square root of the
  divergence); the detector uses `base=2` so the distance is bounded
  in `[0, 1]`.
- **PSI convention (0.1 / 0.25 bands):** the Population Stability
  Index with the widely cited "no change / moderate / major" bands at
  0.1 and 0.25 was popularised by the credit-scoring literature
  (Naeem Siddiqi, *Credit Risk Scorecards*). Cite as convention, not
  universal law. Some sources use 0.1 / 0.2; a primary-source
  citation for the exact 0.1 / 0.25 bands is a reviewer-owned
  follow-up.
- **Evidently AI data-drift docs:**
  <https://docs.evidentlyai.com/>. `column_mapping` and the
  `DataDriftPreset` are the mental model the detector mirrors for
  categorical columns; see the sibling exercise-02 for the
  side-by-side comparison.
- **whylogs segmentation docs:**
  <https://whylogs.readthedocs.io/en/latest/>. The per-segment
  profiling model whylogs exposes is the production pattern the
  `segmentation` block in the recipe mirrors.
- **Local anchors:** `exercise-01` (subpopulation balance SLI — the
  SLI is the summary; this detector is what makes the SLI
  observable); `exercise-02` (Evidently column_mapping / whylogs
  segmentation comparison — this detector is the hand-rolled minimum
  that both frameworks implement); `exercise-03` (schema drift alert
  and rollback — a `severity=major` subpop drift is wired into that
  exercise's severity-major path, not a separate alert); `mod-101`
  (dataset governance — the governance of proxy choice lives there);
  `mod-110` (PII and privacy risk for training data — the proxy
  chosen here must not reconstruct a protected attribute, which is
  the acceptance criterion mod-110 enforces).
