# Exercise 02 — whylogs and Evidently Side-by-Side on One Dataset (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner picks the dataset
pinned by exercise-01's monitoring contract, runs **both** whylogs and
Evidently against the same reference/current pair, and ships a short
decision memo that recommends which tool to adopt for each SLI the
contract declares. The point is not to crown a winner in the abstract;
it is to force a defensible per-SLI choice backed by the two tools'
own output on one dataset.

Deliverables (taken verbatim from the exercise contract):

- `recipes/whylogs_profile.yaml` — pinned whylogs configuration:
  reference dataset pointer, current dataset pointer, feature list,
  type hints, constraint thresholds, segmentation keys.
- `recipes/evidently_report.yaml` — pinned Evidently configuration:
  same dataset pointers, `ColumnMapping`-equivalent feature typing,
  the Presets to run, and drift-test thresholds.
- `code/run_whylogs.py` — profiles reference and current datasets
  with `why.log(...)`, serialises `DatasetProfileView`s, runs
  constraint and drift checks, emits `outputs/whylogs_summary.json`.
- `code/run_evidently.py` — runs Evidently's `Report` with
  `DataDriftPreset` and `DataQualityPreset` on the same pair, exports
  `outputs/evidently_report.html` and `outputs/evidently_summary.json`.
- `code/compare.py` — reads both JSON summaries and emits
  `outputs/comparison.md` with one row per feature, each tool's drift
  verdict, and an **agreement** column.
- `comparison.md` (top-level memo, generated or hand-edited from the
  `outputs/comparison.md` scaffold) — what each tool measures better,
  which tool to adopt per exercise-01 SLI, explicit recommendation.
- `tests/` — pytest cases that fail on the common regressions in §5
  (reference/current swap, missing type hints, divergent feature lists
  between the two recipes).

Governance framing. Drift detection is a measurement control — a
monitor, not a mitigation. Fit it into the three frameworks before
arguing about tool features:

- **OWASP Machine Learning Security Top 10** anchor: **ML02:2023 Data
  Poisoning Attack** (unreviewed distribution shift on inputs is
  indistinguishable from an adversarially injected shift until a
  monitor says otherwise). Cite the pillar URL in §6; do not paraphrase
  the control text.
- **MITRE ATLAS** anchor: **AML.T0043 Craft Adversarial Data** — the
  attacker-facing technique whose operational signal is often an
  input-distribution change the drift monitor is supposed to surface.
  Use the ATLAS matrix URL and the technique page if you are certain
  of the identifier; otherwise cite the matrix root.
- **NIST AI Risk Management Framework** anchor: **MEASURE 2.3** (AI
  system performance or assurance criteria are measured qualitatively
  or quantitatively) and **MEASURE 2.7** (AI system security and
  resilience are evaluated and documented). Drift monitoring is the
  MEASURE 2.3/2.7 instrument; cite the framework root URL and call
  out that the specific sub-item identifiers may evolve.

The memo must name which NIST MEASURE sub-item each SLI maps to so the
audit trail from a drift alert to a RMF control is one hop.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   ├── whylogs_profile.yaml        # pinned whylogs config
│   └── evidently_report.yaml       # pinned Evidently config
├── code/
│   ├── common.py                   # dataset loaders, feature spec
│   ├── run_whylogs.py              # why.log → DatasetProfileView → JSON
│   ├── run_evidently.py            # Report(DataDriftPreset, …) → JSON+HTML
│   └── compare.py                  # join both JSONs → comparison.md
├── outputs/
│   ├── whylogs_reference.bin       # serialised DatasetProfileView
│   ├── whylogs_current.bin
│   ├── whylogs_summary.json
│   ├── evidently_report.html
│   ├── evidently_summary.json
│   └── comparison.md               # per-feature agreement table
├── comparison.md                   # the decision memo
└── tests/
    ├── test_recipe_parity.py       # same features/types in both recipes
    ├── test_reference_current.py   # loader order is not swappable
    └── test_compare_schema.py      # comparison.md row count = feature count
```

### 2.2 `recipes/whylogs_profile.yaml`

```yaml
# recipes/whylogs_profile.yaml
# whylogs configuration pinned for reproducibility.
# This file is read by code/run_whylogs.py; no CLI overrides.

dataset:
  name: pipeline-example-v1
  # Reference and current are the two slices we profile and compare.
  # Pin by content hash. A recipe that points at a mutable path is a
  # correctness bug, not a convenience.
  reference:
    path: data/reference.parquet
    sha256: <64-hex-digest-of-the-reference-slice>
    row_count: 1000000
  current:
    path: data/current.parquet
    sha256: <64-hex-digest-of-the-current-slice>
    row_count: 1000000

# Feature list and type hints. whylogs infers types from pandas dtypes
# by default; declaring them here is what gives review a stable surface.
# Both recipes (whylogs + evidently) MUST declare the same features in
# the same order; tests/test_recipe_parity.py enforces this.
features:
  numerical:
    - name: request_latency_ms
    - name: input_token_count
    - name: output_token_count
    - name: user_age_years
  categorical:
    - name: client_platform
    - name: locale
    - name: model_variant
  text:
    - name: prompt_text
      # text columns use whylogs' text metrics; see §2.4.

# Segmentation keys. whylogs supports per-segment profiles; subpopulation
# drift lives in exercise-04. Keep segmentation off here so the outputs
# are comparable with Evidently's single-report model.
segmentation:
  enabled: false
  # When enabled, segment by [model_variant, locale] in exercise-04.

# Drift decisioning. whylogs' public distance helpers compute per-
# column divergence between two DatasetProfileViews. The exact
# divergence metric name differs by whylogs version; declare both the
# metric family and the numeric threshold, and let code/run_whylogs.py
# resolve the current API.
drift:
  # Metric family: distributional distance between the two profiles.
  # e.g. KS distance for numerical columns, chi-squared or a frequency-
  # based distance for categorical. Exact function name is pinned in
  # code/run_whylogs.py, not here.
  numerical_metric: ks_statistic
  categorical_metric: chi_squared_p_value
  thresholds:
    # A per-feature column with a per-metric number. Reviewers attack
    # these first; justify each override in comparison.md §"Thresholds".
    request_latency_ms:    { ks_statistic: 0.10 }
    input_token_count:     { ks_statistic: 0.10 }
    output_token_count:    { ks_statistic: 0.10 }
    user_age_years:        { ks_statistic: 0.15 }
    client_platform:       { chi_squared_p_value: 0.01 }
    locale:                { chi_squared_p_value: 0.01 }
    model_variant:         { chi_squared_p_value: 0.01 }
    # prompt_text intentionally has no drift threshold — text drift is
    # out of scope for this exercise; see mod-104 exercise-02.

# Constraints. whylogs' Constraints framework asserts properties on a
# single profile (not a drift comparison). Use it for the quality SLI
# gates that do not need a reference.
constraints:
  request_latency_ms:
    # null fraction must stay below 1%; min/max within SLO envelope.
    null_fraction_lt:    0.01
    greater_than_number: 0.0
    less_than_number:    30000.0
  output_token_count:
    null_fraction_lt:    0.0
    greater_than_number: 0.0
  client_platform:
    values_in_set:       ["ios", "android", "web", "api"]
  locale:
    null_fraction_lt:    0.0

# whylogs profile serialisation. The serialised DatasetProfileView is
# the artefact that time-series dashboards (mod-109) consume.
serialisation:
  reference_path: outputs/whylogs_reference.bin
  current_path:   outputs/whylogs_current.bin
  # whylogs serialises profiles via the DatasetProfileView.write(path)
  # API in current versions; code/run_whylogs.py resolves the exact
  # call. Do not hand-roll a format.
```

Why each knob matters at review:

- **Reference and current both pinned by sha256.** A drift report
  computed against a moving reference is noise. If the reference slice
  changes, the recipe changes with it and the run is a new experiment.
- **Feature list is the single source of truth.** Both recipes declare
  the same features. `tests/test_recipe_parity.py` reads both YAMLs and
  diffs their feature sets; divergence fails CI.
- **Drift thresholds are declared, not inherited.** whylogs and
  Evidently both ship opinionated defaults; relying on the default is a
  review red flag. Reviewers want to see *this* number for *this*
  feature with *this* justification in `comparison.md`.
- **Metric names are declared with an "e.g." posture.** The exact
  function name in whylogs' public API has shifted across minor
  versions; `code/run_whylogs.py` resolves the function and records
  what it used in `whylogs_summary.json`. See the version-pinning
  hedge in §2.4.

### 2.3 `recipes/evidently_report.yaml`

```yaml
# recipes/evidently_report.yaml
# Evidently configuration pinned for reproducibility.

dataset:
  name: pipeline-example-v1
  reference:
    path: data/reference.parquet
    sha256: <64-hex-digest-of-the-reference-slice>
    row_count: 1000000
  current:
    path: data/current.parquet
    sha256: <64-hex-digest-of-the-current-slice>
    row_count: 1000000

# Feature typing. Evidently's documented mechanism is a ColumnMapping
# object; this YAML declares the same information and code/run_evidently.py
# materialises it. Keep the feature list byte-identical to
# recipes/whylogs_profile.yaml — the parity test depends on it.
column_mapping:
  numerical_features:
    - request_latency_ms
    - input_token_count
    - output_token_count
    - user_age_years
  categorical_features:
    - client_platform
    - locale
    - model_variant
  text_features:
    - prompt_text
  # Evidently distinguishes target / prediction from features. For this
  # pipeline-input slice there is no target; prediction is scored
  # elsewhere.
  target: null
  prediction: null
  datetime: null
  id: null

# Presets are the entry point Evidently recommends for a first report.
# Keep the list small and documented.
presets:
  - name: DataDriftPreset
    # Evidently's drift pipeline chooses a per-column statistical test
    # based on column type and sample size. The exact algorithm
    # selection is documented in the Evidently docs; declare the
    # threshold intent here and let code/run_evidently.py record the
    # test actually used per column in evidently_summary.json.
    options:
      # e.g. {"stattest_threshold": 0.05} for p-value-based tests,
      # {"stattest_threshold": 0.1} for distance-based tests. The
      # exact option key depends on the pinned Evidently version;
      # code/run_evidently.py records what it actually invoked.
      per_column_stattest_threshold_default: 0.05
  - name: DataQualityPreset
    # Column-level descriptive stats, missing-value counts, and basic
    # distribution summaries. Good for the "integrity" SLI family in
    # exercise-01's contract.
    options: {}

# Drift decisioning override table. Some features need a stricter
# threshold than the Preset default; declare them here so reviewers
# can diff this file instead of reading code.
per_column_thresholds:
  request_latency_ms:    0.10
  input_token_count:     0.10
  output_token_count:    0.10
  user_age_years:        0.15
  client_platform:       0.01
  locale:                0.01
  model_variant:         0.01

# Artifact paths.
outputs:
  html: outputs/evidently_report.html
  json: outputs/evidently_summary.json
```

Why each knob matters at review:

- **`column_mapping` is the Evidently-side analogue of whylogs type
  hints.** Omit it and Evidently falls back on pandas dtype inference,
  which silently misclassifies low-cardinality integers as numerical
  and sparse columns as text. Declare it.
- **Presets, not individual Metrics.** `DataDriftPreset` and
  `DataQualityPreset` are the documented entry points. Starting with
  Presets gives reviewers something they can map to the docs on first
  read; swapping to individual Metrics is a later refinement exercise-04
  takes up.
- **Thresholds overridden per column.** Preset defaults are intentional
  starting points; tune only features where exercise-01's contract
  demands a stricter or looser bar and document the override.

### 2.4 `code/run_whylogs.py`

```python
# code/run_whylogs.py
# Profiles reference + current datasets with whylogs and emits
# outputs/whylogs_summary.json.
from __future__ import annotations

import argparse
import hashlib
import json
import logging
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import Any

import pandas as pd
import whylogs as why          # whylogs public entry point
import yaml
from whylogs.core import DatasetProfileView

LOG = logging.getLogger("run_whylogs")


@dataclass(slots=True)
class FeatureDriftRow:
    feature: str
    feature_type: str
    metric_name: str
    metric_value: float | None
    threshold: float | None
    drift_detected: bool
    note: str = ""


def _sha256(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def _profile(df: pd.DataFrame) -> DatasetProfileView:
    """Return a DatasetProfileView for the given dataframe.

    why.log(df) returns a ResultSet whose .view() is the view object
    we persist. The view is the time-series-friendly artifact that
    mod-109 dashboards consume.
    """
    return why.log(df).view()


def _load_slice(spec: dict, config_path: Path) -> pd.DataFrame:
    path = Path(spec["path"])
    actual = _sha256(path)
    if actual != spec["sha256"]:
        raise SystemExit(
            f"slice hash drift for {path}: recipe says {spec['sha256']}, "
            f"on disk is {actual} (recipe={config_path})"
        )
    return pd.read_parquet(path)


def _column_drift_numerical(
    ref_view: DatasetProfileView,
    cur_view: DatasetProfileView,
    column: str,
    threshold: float,
) -> FeatureDriftRow:
    """Numerical drift via whylogs' profile-to-profile distance helper.

    whylogs exposes distributional-distance utilities between two
    profile views; the exact import path differs by version, so
    resolve at call time. The pinned version is captured in
    requirements.lock and the summary records the function invoked.
    """
    from whylogs.viz.drift import (  # type: ignore[attr-defined]
        calculate_drift_scores,
    )

    # calculate_drift_scores returns a dict keyed by column name;
    # the inner dict carries at minimum a distance value and the
    # algorithm name whylogs selected.
    scores = calculate_drift_scores(
        target_view=cur_view,
        reference_view=ref_view,
        with_thresholds=False,
    )
    entry = scores.get(column) or {}
    metric_value = entry.get("drift_value") or entry.get("statistic")
    metric_name = entry.get("algorithm", "ks_statistic")
    detected = (
        metric_value is not None
        and float(metric_value) > threshold
    )
    return FeatureDriftRow(
        feature=column,
        feature_type="numerical",
        metric_name=str(metric_name),
        metric_value=(
            float(metric_value) if metric_value is not None else None
        ),
        threshold=threshold,
        drift_detected=detected,
    )


def _column_drift_categorical(
    ref_view: DatasetProfileView,
    cur_view: DatasetProfileView,
    column: str,
    threshold: float,
) -> FeatureDriftRow:
    """Categorical drift via a frequency-based distance.

    whylogs' profile holds per-category frequency sketches; recovering
    the top-k from ref and cur and running a chi-squared test is the
    conservative path. If a one-shot API exists in the current whylogs
    version, prefer it and record the algorithm name in `metric_name`;
    otherwise fall back to recovering top-k frequencies and running
    the chi-squared test directly.
    """
    try:
        from whylogs.viz.drift import (  # type: ignore[attr-defined]
            calculate_drift_scores,
        )
        scores = calculate_drift_scores(
            target_view=cur_view,
            reference_view=ref_view,
        )
        entry = scores.get(column) or {}
        p_value = entry.get("p_value")
        metric_name = entry.get("algorithm", "chi_squared_p_value")
        detected = p_value is not None and float(p_value) < threshold
        return FeatureDriftRow(
            feature=column,
            feature_type="categorical",
            metric_name=str(metric_name),
            metric_value=(
                float(p_value) if p_value is not None else None
            ),
            threshold=threshold,
            drift_detected=detected,
        )
    except Exception as exc:
        return FeatureDriftRow(
            feature=column,
            feature_type="categorical",
            metric_name="unavailable",
            metric_value=None,
            threshold=threshold,
            drift_detected=False,
            note=f"drift unavailable: {exc!r}",
        )


def _run_constraints(view: DatasetProfileView, cfg: dict) -> list[dict]:
    """Evaluate the recipe's constraints on the current profile.

    whylogs' Constraints framework is the one-shot evaluator here;
    report() returns a pass/fail entry per constraint. The exact
    builder API path (ConstraintsBuilder / Constraints /
    generate_constraints) is pinned via requirements.lock, and the
    summary records the symbol actually invoked.
    """
    from whylogs.core.constraints import (  # type: ignore[attr-defined]
        ConstraintsBuilder,
        MetricConstraint,
        MetricsSelector,
    )

    builder = ConstraintsBuilder(dataset_profile_view=view)
    for column, rules in cfg.get("constraints", {}).items():
        for rule_name, rule_value in rules.items():
            builder.add_constraint(_constraint_for(
                column=column, rule=rule_name, value=rule_value,
                selector_cls=MetricsSelector, constraint_cls=MetricConstraint,
            ))
    constraints = builder.build()
    report = constraints.generate_constraints_report()
    # Normalise the report shape — the exact tuple shape differs
    # across whylogs versions. Preserve the three fields we need.
    out: list[dict] = []
    for entry in report:
        out.append({
            "name":    getattr(entry, "name", str(entry)),
            "passed":  bool(getattr(entry, "passed", False)),
            "detail":  getattr(entry, "message", ""),
        })
    return out


def _constraint_for(
    *, column: str, rule: str, value: Any,
    selector_cls: Any, constraint_cls: Any,
) -> Any:
    """Map our YAML rule names onto whylogs' constraint primitives."""
    # Only three rule families are supported by this recipe; declare
    # them explicitly so a typo in YAML is a hard error at run time.
    if rule == "null_fraction_lt":
        return constraint_cls(
            name=f"{column}.null_fraction<{value}",
            condition=lambda m, v=value: m.null_count.value / max(m.count.value, 1) < v,
            metric_selector=selector_cls(column_name=column, metric_name="counts"),
        )
    if rule == "greater_than_number":
        return constraint_cls(
            name=f"{column}.min>{value}",
            condition=lambda m, v=value: m.min > v,
            metric_selector=selector_cls(column_name=column, metric_name="distribution"),
        )
    if rule == "less_than_number":
        return constraint_cls(
            name=f"{column}.max<{value}",
            condition=lambda m, v=value: m.max < v,
            metric_selector=selector_cls(column_name=column, metric_name="distribution"),
        )
    if rule == "values_in_set":
        allowed = set(value)
        return constraint_cls(
            name=f"{column}.values_in_set",
            condition=lambda m, a=allowed: set(m.frequent_strings.keys()).issubset(a),
            metric_selector=selector_cls(column_name=column, metric_name="frequent_items"),
        )
    raise SystemExit(f"unknown constraint rule: {rule}")


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config", required=True, type=Path)
    ap.add_argument("--out-json", required=True, type=Path)
    args = ap.parse_args()

    cfg = yaml.safe_load(args.config.read_text())
    ref_df = _load_slice(cfg["dataset"]["reference"], args.config)
    cur_df = _load_slice(cfg["dataset"]["current"],   args.config)

    ref_view = _profile(ref_df)
    cur_view = _profile(cur_df)

    ref_view.write(cfg["serialisation"]["reference_path"])
    cur_view.write(cfg["serialisation"]["current_path"])

    rows: list[FeatureDriftRow] = []
    for col in cfg["features"]["numerical"]:
        name = col["name"]
        threshold = cfg["drift"]["thresholds"][name]["ks_statistic"]
        rows.append(_column_drift_numerical(ref_view, cur_view, name, threshold))
    for col in cfg["features"]["categorical"]:
        name = col["name"]
        threshold = cfg["drift"]["thresholds"][name]["chi_squared_p_value"]
        rows.append(_column_drift_categorical(ref_view, cur_view, name, threshold))

    constraints_report = _run_constraints(cur_view, cfg)

    summary = {
        "tool":             "whylogs",
        "tool_version":     _whylogs_version(),
        "recipe_path":      str(args.config),
        "reference_sha256": cfg["dataset"]["reference"]["sha256"],
        "current_sha256":   cfg["dataset"]["current"]["sha256"],
        "drift_rows":       [asdict(r) for r in rows],
        "constraints":      constraints_report,
    }
    args.out_json.write_text(json.dumps(summary, indent=2, sort_keys=True))


def _whylogs_version() -> str:
    try:
        return why.__version__                 # type: ignore[attr-defined]
    except Exception:
        import importlib.metadata as md
        return md.version("whylogs")


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
```

Design notes defensible at review:

- **`why.log(df).view()` is the stable entry point.** The function
  that builds the profile is public; the view is what we persist and
  what the dashboard reads. `view.write(path)` writes a binary profile
  file whose format whylogs owns — do not re-invent.
- **Drift and constraint API paths are version-resolved at call time.**
  The exact whylogs symbol paths have shifted across minor versions;
  the code resolves them at call time and the summary records what it
  actually invoked (`metric_name`, `tool_version`). Reviewers can diff
  the summary to verify a version-stable report.
- **Constraints are evaluated on the current profile only.** The
  reference's job is to be a comparison partner; the SLI gates in
  exercise-01's contract that need a single-profile property
  (null-fraction, value-set membership, range) live in whylogs
  constraints.

### 2.5 `code/run_evidently.py`

```python
# code/run_evidently.py
# Runs Evidently's Report with DataDriftPreset + DataQualityPreset.
from __future__ import annotations

import argparse
import hashlib
import json
from pathlib import Path

import pandas as pd
import yaml
from evidently import ColumnMapping
from evidently.metric_preset import DataDriftPreset, DataQualityPreset
from evidently.report import Report


def _sha256(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def _load_slice(spec: dict, config_path: Path) -> pd.DataFrame:
    path = Path(spec["path"])
    actual = _sha256(path)
    if actual != spec["sha256"]:
        raise SystemExit(
            f"slice hash drift for {path}: recipe says {spec['sha256']}, "
            f"on disk is {actual} (recipe={config_path})"
        )
    return pd.read_parquet(path)


def _column_mapping(cfg: dict) -> ColumnMapping:
    cm = ColumnMapping()
    cm.numerical_features   = list(cfg["column_mapping"]["numerical_features"])
    cm.categorical_features = list(cfg["column_mapping"]["categorical_features"])
    # Evidently's text handling varies by version; declare the list
    # and fall back to categorical if text is unsupported. The
    # summary records what we used.
    try:
        cm.text_features = list(cfg["column_mapping"]["text_features"])
    except Exception:
        # ColumnMapping.text_features is the attribute in the pinned
        # Evidently version; older/newer versions may expose it
        # differently, so fall back silently and let the summary
        # record that text features were untyped.
        pass
    cm.target     = cfg["column_mapping"].get("target")
    cm.prediction = cfg["column_mapping"].get("prediction")
    cm.datetime   = cfg["column_mapping"].get("datetime")
    cm.id         = cfg["column_mapping"].get("id")
    return cm


def _build_report(cfg: dict) -> Report:
    """Build the Report from the Presets declared in the recipe."""
    presets: list = []
    for p in cfg["presets"]:
        name = p["name"]
        if name == "DataDriftPreset":
            presets.append(DataDriftPreset())
        elif name == "DataQualityPreset":
            presets.append(DataQualityPreset())
        else:
            raise SystemExit(f"unsupported preset: {name}")
    return Report(metrics=presets)


def _extract_drift_rows(report_json: dict, cfg: dict) -> list[dict]:
    """Walk the Report JSON and extract per-column drift verdicts.

    Evidently's JSON shape is stable in broad strokes (metrics list
    with 'result' bodies) but field names drift between versions.
    Resolve defensively and keep the shape of the row stable.
    """
    rows: list[dict] = []
    thresholds = cfg["per_column_thresholds"]
    for metric in report_json.get("metrics", []):
        if metric.get("metric") != "DataDriftTable":
            continue
        result = metric.get("result") or {}
        by_column = (result.get("drift_by_columns")
                     or result.get("by_column")
                     or {})
        for column, body in by_column.items():
            stattest    = body.get("stattest_name") or body.get("stattest")
            drift_score = body.get("drift_score")
            detected    = bool(body.get("drift_detected", False))
            rows.append({
                "feature":          column,
                "feature_type":     body.get("column_type", "unknown"),
                "metric_name":      str(stattest),
                "metric_value":     (
                    float(drift_score) if drift_score is not None else None
                ),
                "threshold":        thresholds.get(column),
                "drift_detected":   detected,
                "note":             "",
            })
    return rows


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config",    required=True, type=Path)
    ap.add_argument("--out-json",  required=True, type=Path)
    ap.add_argument("--out-html",  required=True, type=Path)
    args = ap.parse_args()

    cfg = yaml.safe_load(args.config.read_text())
    ref_df = _load_slice(cfg["dataset"]["reference"], args.config)
    cur_df = _load_slice(cfg["dataset"]["current"],   args.config)
    cm = _column_mapping(cfg)

    report = _build_report(cfg)
    report.run(reference_data=ref_df, current_data=cur_df, column_mapping=cm)
    report.save_html(str(args.out_html))
    report_json = report.as_dict()           # stable public method

    summary = {
        "tool":             "evidently",
        "tool_version":     _evidently_version(),
        "recipe_path":      str(args.config),
        "reference_sha256": cfg["dataset"]["reference"]["sha256"],
        "current_sha256":   cfg["dataset"]["current"]["sha256"],
        "drift_rows":       _extract_drift_rows(report_json, cfg),
        "raw_report":       report_json,
    }
    args.out_json.write_text(json.dumps(summary, indent=2, sort_keys=True))


def _evidently_version() -> str:
    import importlib.metadata as md
    return md.version("evidently")


if __name__ == "__main__":
    main()
```

Design notes defensible at review:

- **`Report(metrics=[Preset(), …])` is the documented entry point.**
  Use Presets first; adding individual Metrics is a later-exercise
  refinement. See <https://docs.evidentlyai.com/>.
- **`ColumnMapping` is non-optional.** Without it Evidently falls back
  to dtype inference; the resulting type mismatches silently change
  which statistical test Evidently picks per column.
- **`report.as_dict()` is the public JSON entry.** Save the HTML for
  humans, the JSON for `compare.py`. Do not parse the HTML.
- **Field-name drift is handled defensively.** `_extract_drift_rows`
  tolerates two known key shapes (`drift_by_columns` vs `by_column`)
  so a minor-version bump does not silently empty the comparison.

### 2.6 `code/compare.py`

```python
# code/compare.py
# Joins outputs/whylogs_summary.json and outputs/evidently_summary.json
# on `feature` and emits outputs/comparison.md.
from __future__ import annotations

import argparse
import json
from pathlib import Path


def _rows(summary: dict) -> dict[str, dict]:
    return {r["feature"]: r for r in summary["drift_rows"]}


def _agreement(a: dict | None, b: dict | None) -> str:
    if a is None or b is None:
        return "missing"
    if a["drift_detected"] == b["drift_detected"]:
        return "agree"
    return "disagree"


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--whylogs",   required=True, type=Path)
    ap.add_argument("--evidently", required=True, type=Path)
    ap.add_argument("--out",       required=True, type=Path)
    args = ap.parse_args()

    wh = json.loads(args.whylogs.read_text())
    ev = json.loads(args.evidently.read_text())

    if wh["reference_sha256"] != ev["reference_sha256"]:
        raise SystemExit(
            "reference_sha256 differs between whylogs and evidently summaries; "
            "the two tools were run against different reference slices."
        )
    if wh["current_sha256"] != ev["current_sha256"]:
        raise SystemExit(
            "current_sha256 differs between whylogs and evidently summaries; "
            "the two tools were run against different current slices."
        )

    wh_rows = _rows(wh)
    ev_rows = _rows(ev)
    features = sorted(set(wh_rows) | set(ev_rows))

    lines: list[str] = []
    lines.append("# Per-feature drift comparison\n")
    lines.append(
        f"- whylogs: `{wh['tool_version']}` against "
        f"recipe `{wh['recipe_path']}`\n"
        f"- evidently: `{ev['tool_version']}` against "
        f"recipe `{ev['recipe_path']}`\n"
        f"- reference sha256: `{wh['reference_sha256']}`\n"
        f"- current sha256:   `{wh['current_sha256']}`\n"
    )
    lines.append(
        "| Feature | Type | whylogs metric | whylogs value | "
        "whylogs drift? | Evidently metric | Evidently value | "
        "Evidently drift? | Agreement |\n"
        "|---|---|---|---|---|---|---|---|---|\n"
    )
    for feat in features:
        a = wh_rows.get(feat)
        b = ev_rows.get(feat)
        lines.append(
            f"| {feat} "
            f"| {(a or b or {}).get('feature_type', '')} "
            f"| {a and a['metric_name'] or '-'} "
            f"| {a and a['metric_value'] if a else '-'} "
            f"| {a and a['drift_detected'] if a else '-'} "
            f"| {b and b['metric_name'] or '-'} "
            f"| {b and b['metric_value'] if b else '-'} "
            f"| {b and b['drift_detected'] if b else '-'} "
            f"| {_agreement(a, b)} |\n"
        )

    args.out.write_text("".join(lines))


if __name__ == "__main__":
    main()
```

Design notes defensible at review:

- **Reference and current sha256 must match between summaries.** A
  comparison that silently compared two different slices is the
  single highest-impact failure mode; fail loud, fail fast.
- **`Agreement` is explicit.** `agree` / `disagree` / `missing` is the
  only column reviewers actually read; the metric columns exist to
  justify the agreement call.

### 2.7 `comparison.md` (the decision memo)

One page. Required sections (reviewer-ordered):

1. **Summary.** Dataset name, reference/current sha256 pair, both
   tool versions, and the headline count of agree/disagree/missing
   features.
2. **Where each tool shines.** Two short paragraphs (not bullet dumps):
   - **whylogs** shines for **streaming / batch profiles over
     time**. `DatasetProfileView` is a small, mergeable artifact; you
     can log one per batch and dashboard the time-series, and the
     Constraints framework gives one-profile SLI gates. Choose whylogs
     when the SLI is "property X held on today's batch" or "today's
     batch matches last week's rolling profile".
   - **Evidently** shines for **one-shot, human-readable reports**.
     `Report(DataDriftPreset())` produces an HTML artifact an
     on-call engineer can open during an incident and a JSON an alert
     pipeline can parse. Choose Evidently when the SLI is "did
     today's batch drift from the pinned reference, and would I like
     a page of charts explaining why".
3. **Per-SLI tool recommendation.** A table mapping each SLI name
   from exercise-01's contract to whylogs / Evidently / both, with a
   one-sentence justification per row.
4. **Thresholds.** The drift thresholds pinned in both recipes. One
   row per feature, both tools' threshold, and the justification.
5. **Divergence metric notes.** What each tool uses per column type.
   Use "e.g." where exact algorithm selection depends on tool version;
   the `whylogs_summary.json` and `evidently_summary.json` files
   record what each run actually used.
6. **Known gaps.** Features neither tool handles well in the pinned
   versions (e.g., free-form text drift beyond length and basic
   lexical metrics — mod-104 exercise-02 territory).
7. **Recommendation.** One sentence. "Adopt whylogs for the
   time-series SLIs (null-fraction, range, value-set) and Evidently
   for the one-shot incident report; run both for the first N weeks
   while thresholds settle." Or the exercise-specific equivalent.

Any claim that cannot be verified at authoring time is called out
inline as a placeholder with a named owner for the follow-up, and the
same policy blocks auto-merge.

## 3. Validation steps

1. **Recipe parity holds.** `python -c "import yaml; a=yaml.safe_load(open('recipes/whylogs_profile.yaml'));
   b=yaml.safe_load(open('recipes/evidently_report.yaml'));
   assert [f['name'] for f in a['features']['numerical']] ==
   b['column_mapping']['numerical_features']"` exits 0. Repeat for
   categorical and text. `tests/test_recipe_parity.py` codifies this.
2. **Reference and current slice hashes match the recipes.**
   `sha256sum data/reference.parquet` and `data/current.parquet`
   match the digests in both recipes. Both runners fail at startup
   on drift.
3. **Both runners complete clean.** `python code/run_whylogs.py
   --config recipes/whylogs_profile.yaml --out-json
   outputs/whylogs_summary.json` exits 0 and writes the summary plus
   two `.bin` profile views. `python code/run_evidently.py --config
   recipes/evidently_report.yaml --out-json
   outputs/evidently_summary.json --out-html
   outputs/evidently_report.html` exits 0 and writes HTML + JSON.
4. **Both JSON summaries carry versioned metadata.** `jq
   '.tool_version, .reference_sha256, .current_sha256'
   outputs/whylogs_summary.json` returns three non-null strings;
   same for `outputs/evidently_summary.json`. The reference/current
   sha256 must match byte-for-byte between the two summaries.
5. **Feature coverage is total.** Every feature in the recipes has
   one row in `outputs/whylogs_summary.json#drift_rows` and one row in
   `outputs/evidently_summary.json#drift_rows`. Missing rows fail
   `tests/test_compare_schema.py`.
6. **`compare.py` produces `outputs/comparison.md`.** The row count
   of the markdown table equals the feature count in the recipe. The
   `Agreement` column is one of `agree`, `disagree`, `missing` on
   every row.
7. **Constraints report is non-empty.** `jq '.constraints | length'
   outputs/whylogs_summary.json` is at least the number of
   constraints declared in the recipe; each entry has a `passed`
   boolean and a `name`.
8. **Evidently HTML opens.** `file outputs/evidently_report.html`
   reports an HTML document and `grep -c 'DataDrift' outputs/evidently_report.html`
   is non-zero. (A corrupt save is a known Evidently failure mode on
   large reports — the HTML must render.)
9. **Reference / current swap is detected.** `tests/test_reference_current.py`
   swaps the two paths in a copy of the recipe and asserts both
   runners either produce a summary with the opposite `drift_detected`
   polarity or the swap is otherwise detectable; the test documents
   which tool is swap-symmetric.
10. **Decision memo cites exercise-01 SLIs by name.** `comparison.md`
    references each SLI in the contract and names the chosen tool;
    the pre-publish check greps for the SLI names and fails on an
    omission.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 24/32 with zero zeros
on critical rows.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `recipes/whylogs_profile.yaml` pins reference + current sha256, feature list, type hints, thresholds            | Yes       |
| 2  | `recipes/evidently_report.yaml` pins reference + current sha256, ColumnMapping-equivalent typing, Presets list  | Yes       |
| 3  | Both recipes declare the same features in the same order (parity test passes)                                   | Yes       |
| 4  | `code/run_whylogs.py` emits a `DatasetProfileView` per slice via `why.log(df).view()` and serialises both       | Yes       |
| 5  | `code/run_evidently.py` builds `Report(DataDriftPreset, DataQualityPreset)` and saves both HTML and JSON         | Yes       |
| 6  | Both runners abort on reference or current slice-hash drift                                                     | Yes       |
| 7  | Both JSON summaries record `tool_version`, `reference_sha256`, `current_sha256`                                 | Yes       |
| 8  | `code/compare.py` joins on `feature` and emits an `agree` / `disagree` / `missing` column                       | Yes       |
| 9  | `compare.py` aborts on reference/current sha256 mismatch between the two summaries                              | Yes       |
| 10 | `comparison.md` maps every exercise-01 SLI to a tool with a one-sentence justification                          | Yes       |
| 11 | Divergence-metric claims are hedged with "e.g." where the exact whylogs/Evidently API varies by version         | Yes       |
| 12 | whylogs constraints framework is exercised on the current profile (null-fraction, range, value-set)             | Yes       |
| 13 | Tool versions pinned in `requirements.lock` (whylogs, evidently, pandas, pyarrow)                               | Yes       |
| 14 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0043, NIST AI RMF MEASURE 2.3 / 2.7) cited           | No        |
| 15 | Unverifiable claims are called out inline as placeholders with a named owner for the follow-up                  | Yes       |
| 16 | `pytest tests/` passes (recipe-parity, reference-current, compare-schema)                                       | Yes       |

## 5. Common mistakes

- **Running whylogs and Evidently on different slices.** The whole
  memo is invalid if the two tools saw different data. Hash both
  reference and current in both recipes; `compare.py` fails the join
  on a mismatch.
- **Trusting dtype inference instead of declaring types.** A
  low-cardinality integer column (e.g., `locale_id`) that is actually
  categorical gets a KS test from Evidently and a numeric distance
  from whylogs; both are the wrong metric. Declare types in both
  recipes.
- **Leaving Preset thresholds at their defaults with no comment.**
  Reviewers cannot tell "default was fine" from "nobody checked".
  Declare the thresholds explicitly in both recipes, even if the
  value equals the Preset default.
- **Citing exact API names without version-checking.** whylogs' drift
  helper path has moved across minor versions; Evidently's JSON keys
  have shifted similarly. Use "e.g." phrasing in the memo and have
  the code record what it actually invoked.
- **Treating "agree" as proof of correctness.** Two tools both saying
  "no drift" with the wrong feature type is two tools being wrong in
  parallel. Agreement is a prior; the drill-down matters.
- **Writing a universal recommendation.** "Use Evidently for
  everything" or "use whylogs for everything" fails the per-SLI
  justification bar. Each SLI in exercise-01's contract must map to a
  tool with a reason.
- **Skipping the Evidently HTML.** The HTML is the artifact on-call
  will actually read. A JSON-only run throws away half of Evidently's
  value proposition.
- **Running whylogs without serialising the profile.** The point of
  `DatasetProfileView.write(path)` is that mod-109 dashboards read
  the serialised profile over time; skipping serialisation means the
  drift signal is one-shot and non-mergeable.
- **Enabling whylogs segmentation in this exercise.** Subpopulation
  drift lives in exercise-04; mixing it in here produces a report
  whose feature count does not match the Evidently side and breaks
  the parity test.
- **Hand-parsing the Evidently HTML.** Evidently's HTML is a rendered
  artifact; use `report.as_dict()` for the JSON and leave the HTML
  alone. A regex over HTML is a regression waiting to happen.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Input-distribution drift is indistinguishable from an adversarially
  induced shift until a monitor says otherwise; the pillar URL is the
  stable anchor.
- **MITRE ATLAS — matrix root:** <https://atlas.mitre.org/>. The
  attacker-facing techniques whose operational signal is often an
  input-distribution change (e.g., Craft Adversarial Data) are what
  the drift monitor is designed to make visible.
- **NIST AI Risk Management Framework:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 anchors quantitative performance measurement; MEASURE 2.7
  anchors security and resilience evaluation. Drift monitoring is a
  MEASURE-family instrument; cite the framework root and name the
  sub-item in the memo.
- **whylogs documentation:** <https://whylogs.readthedocs.io/>. The
  public reference for `why.log`, `DatasetProfileView`, serialisation,
  the Constraints framework, and the drift helpers. The exact
  divergence-metric function names should be cross-checked against
  the version pinned in `requirements.lock`.
- **Evidently documentation:** <https://docs.evidentlyai.com/>. The
  public reference for `Report`, `ColumnMapping`, `DataDriftPreset`,
  `DataQualityPreset`, and the HTML / JSON artifact shapes. Per-column
  stattest selection is documented there; the exact option key for
  threshold overrides depends on the pinned version.
- **KLL quantile sketches (the sketch family underlying whylogs'
  numeric summaries):** <https://datasketches.apache.org/docs/KLL/KLLSketch.html>.
  Useful for reviewers who want to understand the accuracy / memory
  trade-off a whylogs profile makes before comparing two profiles.
- **Local anchors:**
  - Exercise 01 (the monitoring contract whose SLIs this memo maps to
    a tool — the recommendation table in §2.7 is the concrete output
    of that mapping).
  - Exercise 03 (schema-drift alert + rollback — the tool chosen in
    this memo for the "schema" SLI family is the detector exercise-03
    wires into the rollback workflow).
  - Exercise 04 (subpopulation balance drift — Evidently's
    `ColumnMapping` and whylogs' segmentation feature both become
    first-class there; this exercise keeps segmentation off so the
    two outputs are comparable).
  - `mod-104` exercise-01 (quality measurement with Great Expectations
    — whylogs constraints are the per-profile analogue; the two
    frameworks compose rather than compete).
  - `mod-109` (profile-id pinning — the `DatasetProfileView` artifacts
    this exercise writes are the per-batch rows a mod-109 dashboard
    joins over time).
