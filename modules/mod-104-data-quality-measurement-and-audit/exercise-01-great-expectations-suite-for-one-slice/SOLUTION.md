# Exercise 01 — Great Expectations Suite for One Slice (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner picks one slice from an
earlier module (mod-103's Common Crawl English slice is the canonical input)
and expresses its quality bar as an executable Great Expectations (GX) suite.
The deliverable is: a pinned GX project, a suite that fails loudly when the
slice drifts, a Checkpoint that runs the suite against the slice, and a
Data Docs artefact a reviewer can read.

The point of the exercise is to make the quality bar machine-checkable.
Prose quality criteria ("documents should be mostly English") are unverifiable
at review time. An Expectation ("≥ 95% of rows have `language == 'en'` with
`language_score ≥ 0.5`") is a function of the slice bytes; it either passes
or fails, and the failure identifies the rows.

A good solution produces:

1. A GX **DataContext** pinned by code (not CLI state) so `gx.get_context()`
   is byte-reproducible across machines.
2. A **Batch** that points at the slice (JSONL.gz, parquet, or a DuckDB view
   over the slice's manifest), with the batch identifier wired to the slice
   name + version from mod-103.
3. An **Expectation Suite** that covers: schema, nullability, value ranges,
   categorical membership, and at least one **distribution** expectation
   (row count envelope, mean language score, etc.).
4. A **Checkpoint** that runs the suite against the Batch and emits Data Docs
   plus a JSON `validation_result`.
5. A written mapping from each Expectation to a quality concern (not just
   "this column exists" — "this column exists because a null here means the
   acquire-stage decision was lost, which maps to NIST AI RMF `MEASURE 2.10`").

Governance framing: an unchecked slice can silently degrade — a drifted
language-ID threshold, a corrupted shard, a schema-shape change from a new
operator — and feed poisoned data to training. OWASP ML Top 10 **ML02 Data
Poisoning Attack** describes the threat; the GX suite is one of the
compensating controls referenced in NIST AI RMF `MEASURE 2.10`
("measurement of AI system trustworthy characteristics").

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── slice_pin.yaml
├── gx/
│   ├── great_expectations.yml
│   ├── expectations/
│   │   └── slice_cc_en_small_v1.json
│   ├── checkpoints/
│   │   └── slice_cc_en_small_v1.yml
│   └── uncommitted/                # generated; not committed
│       ├── data_docs/
│       └── validations/
├── code/
│   ├── 01_build_suite.py
│   ├── 02_run_checkpoint.py
│   └── common.py
└── reports/
    └── validation_result.json      # emitted by the checkpoint
```

Pin versions. GX changed its context + fluent-datasource APIs between 0.17
and 1.x; the suite below is written against the **GX 1.x** API. Record the
exact version in `requirements.lock` so the Checkpoint re-runs
deterministically. Re-verify the pinned GX 1.x minor against the actual
`requirements.lock` entry before publishing.

### 2.2 `slice_pin.yaml` — identify the slice under audit

```yaml
# slice_pin.yaml
# The slice this suite audits. Fields here are the "contract" that
# the Expectation Suite must stay in step with.
slice_name: "slice-cc-en-small-v1"
slice_version: "1.0.0"
source_manifest: "../mod-103/slice_manifest.json"
source_predicate_sha256: "<sha256 of mod-103 selection_predicate.sql>"
source_snapshot_id: "CC-MAIN-2024-30"
input:
  kind: "jsonl_gz"
  path: "../mod-103/output/slice-00000.jsonl.gz"
  observed_sha256: "<fill at audit time>"
schema:
  required_columns:
    - text
    - url
    - warc_record_id
    - language
    - language_score
  column_types:
    text: string
    url: string
    warc_record_id: string
    language: string
    language_score: float
envelope:
  min_rows: 80000
  max_rows: 120000
  required_language_share:
    en: [0.95, 1.00]
  language_score_mean: [0.70, 1.00]
```

The envelope figures come from the slice's own acceptance criteria in
mod-103 §2.4 (language-ID threshold = 0.5, English-only target). Do not
fabricate numbers — if your slice runs at a different scale, change the
envelope in this file and re-build the suite.

### 2.3 `code/01_build_suite.py` — compose the Expectation Suite

```python
# code/01_build_suite.py
# Build the GX 1.x Expectation Suite from slice_pin.yaml.
# The suite is composed programmatically so the YAML is the single source of truth.
from __future__ import annotations

import json
from pathlib import Path

import yaml
import great_expectations as gx
from great_expectations.core.expectation_configuration import ExpectationConfiguration

SUITE_NAME = "slice_cc_en_small_v1"
PIN = Path("slice_pin.yaml")


def build(pin: dict) -> list[ExpectationConfiguration]:
    cols = pin["schema"]["required_columns"]
    types = pin["schema"]["column_types"]
    env = pin["envelope"]
    exps: list[ExpectationConfiguration] = []

    # --- Shape / schema ----------------------------------------------------
    exps.append(ExpectationConfiguration(
        expectation_type="expect_table_columns_to_match_set",
        kwargs={"column_set": cols, "exact_match": True},
        meta={"rationale": "schema shape is contract between mod-103 and mod-104"},
    ))
    exps.append(ExpectationConfiguration(
        expectation_type="expect_table_row_count_to_be_between",
        kwargs={"min_value": env["min_rows"], "max_value": env["max_rows"]},
        meta={"rationale": "detects silent slice shrinkage/explosion"},
    ))

    # --- Per-column type + nullability ------------------------------------
    for c in cols:
        exps.append(ExpectationConfiguration(
            expectation_type="expect_column_values_to_not_be_null",
            kwargs={"column": c, "mostly": 1.0},
            meta={"rationale": f"{c} is load-bearing; a null = silent drop upstream"},
        ))
        exps.append(ExpectationConfiguration(
            expectation_type="expect_column_values_to_be_of_type",
            kwargs={"column": c, "type_": {
                "string": "StringType", "float": "DoubleType"
            }[types[c]]},
            meta={"rationale": "shape-drift catch"},
        ))

    # --- Value ranges / categoricals --------------------------------------
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_values_to_be_between",
        kwargs={"column": "language_score", "min_value": 0.5, "max_value": 1.0},
        meta={"rationale": "mod-103 threshold was 0.5; below = produced-but-not-filtered"},
    ))
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_value_lengths_to_be_between",
        kwargs={"column": "text", "min_value": 50, "max_value": 2_000_000},
        meta={"rationale": "guards both 'empty extract' and 'extractor pathology'"},
    ))

    # --- Distribution checks ----------------------------------------------
    en_lo, en_hi = env["required_language_share"]["en"]
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_value_counts_to_be_between",
        kwargs={"column": "language", "value": "en",
                "min_value": en_lo, "max_value": en_hi, "normalize": True},
        meta={"rationale": "slice contract is English-only within envelope"},
    ))
    m_lo, m_hi = env["language_score_mean"]
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_mean_to_be_between",
        kwargs={"column": "language_score", "min_value": m_lo, "max_value": m_hi},
        meta={"rationale": "mean collapses when the slice ingests noise"},
    ))

    # --- Hygiene checks ---------------------------------------------------
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_values_to_match_regex",
        kwargs={"column": "url", "regex": r"^https?://"},
        meta={"rationale": "URL must be fetchable or the manifest is lying"},
    ))
    exps.append(ExpectationConfiguration(
        expectation_type="expect_column_values_to_be_unique",
        kwargs={"column": "warc_record_id", "mostly": 0.999},
        meta={"rationale": "intra-slice dedup is mod-105's job; a non-unique id here = bug"},
    ))
    return exps


def main() -> None:
    pin = yaml.safe_load(PIN.read_text())
    context = gx.get_context(project_root_dir="gx")

    suite = context.suites.add_or_update(
        gx.ExpectationSuite(name=SUITE_NAME, expectations=build(pin))
    )
    out = Path("gx/expectations") / f"{SUITE_NAME}.json"
    out.write_text(json.dumps(suite.configuration, indent=2, sort_keys=True))
    print(f"[01] suite '{SUITE_NAME}' has {len(suite.expectations)} expectations")


if __name__ == "__main__":
    main()
```

### 2.4 `code/02_run_checkpoint.py` — execute against the slice

```python
# code/02_run_checkpoint.py
# Wire the suite to the slice via a fluent datasource + Checkpoint,
# write a JSON validation_result + regenerate Data Docs.
from __future__ import annotations

import hashlib
import json
from pathlib import Path

import yaml
import great_expectations as gx

SUITE_NAME = "slice_cc_en_small_v1"
PIN = Path("slice_pin.yaml")
OUT = Path("reports/validation_result.json")


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    pin = yaml.safe_load(PIN.read_text())
    slice_path = Path(pin["input"]["path"]).resolve()
    observed = sha256_file(slice_path)
    pin["input"]["observed_sha256"] = observed
    PIN.write_text(yaml.safe_dump(pin, sort_keys=False))

    context = gx.get_context(project_root_dir="gx")
    # One-shot fluent datasource over a pandas reader; for parquet swap to
    # read_parquet or use the DuckDB/SQL datasource.
    ds = context.data_sources.add_or_update_pandas(name="slice_fs")
    asset = ds.add_jsonl_asset(name="slice_jsonl")  # fluent API — verify exact method name against your GX 1.x minor
    batch_definition = asset.add_batch_definition_path(
        name="single_file", path=str(slice_path)
    )

    checkpoint = context.checkpoints.add_or_update(
        gx.Checkpoint(
            name="slice_cc_en_small_v1_cp",
            validation_definitions=[
                gx.ValidationDefinition(
                    name="slice_v",
                    data=batch_definition,
                    suite=context.suites.get(SUITE_NAME),
                )
            ],
            result_format={"result_format": "SUMMARY"},
        )
    )

    result = checkpoint.run()
    OUT.parent.mkdir(parents=True, exist_ok=True)
    OUT.write_text(json.dumps(result.describe_dict(), indent=2, sort_keys=True))
    context.build_data_docs()
    # exit code lets Makefile / CI fail-fast on a red suite
    raise SystemExit(0 if result.success else 1)


if __name__ == "__main__":
    main()
```

Compatibility note: GX 1.x renamed and moved several fluent APIs between
0.17 → 1.0 → 1.x. Confirm `add_jsonl_asset` / `add_batch_definition_path` /
`ValidationDefinition` / `Checkpoint` constructors against the exact minor
pinned in `requirements.lock`, and swap the names if they changed.

### 2.5 `gx/great_expectations.yml` (minimal)

```yaml
# gx/great_expectations.yml
config_version: 4
stores:
  expectations_store:
    class_name: ExpectationsStore
    store_backend:
      class_name: TupleFilesystemStoreBackend
      base_directory: expectations/
  validation_results_store:
    class_name: ValidationResultsStore
    store_backend:
      class_name: TupleFilesystemStoreBackend
      base_directory: uncommitted/validations/
  checkpoints_store:
    class_name: CheckpointStore
    store_backend:
      class_name: TupleFilesystemStoreBackend
      base_directory: checkpoints/
data_docs_sites:
  local_site:
    class_name: SiteBuilder
    store_backend:
      class_name: TupleFilesystemStoreBackend
      base_directory: uncommitted/data_docs/local_site/
    site_index_builder: {class_name: DefaultSiteIndexBuilder}
analytics_enabled: false
```

Compatibility note: GX's `great_expectations.yml` schema versions shift
between minor releases. Confirm `config_version` and the store class names
against the pinned release before committing.

### 2.6 `Makefile`

```makefile
.PHONY: all build run clean

all: run

build:
	python code/01_build_suite.py

run: build
	python code/02_run_checkpoint.py

clean:
	rm -rf gx/uncommitted reports/validation_result.json
```

### 2.7 `README.md` (excerpts the reviewer needs)

Record, with evidence:

- GX version (`pip show great_expectations | grep Version`).
- SHA-256 of the slice JSONL.gz at audit time (`observed_sha256` in
  `slice_pin.yaml`).
- Git SHA of the slice's producing repo (mod-103).
- Which column → quality characteristic each Expectation maps to (one line
  per Expectation; mirrors the `meta.rationale` field in the suite).

## 3. Validation steps

1. **The suite runs.** `make run` exits 0 on a known-good slice and 1 on a
   slice you deliberately break (e.g. truncate to 1,000 rows — the row-count
   Expectation must fail).
2. **The suite is reproducible.** `make clean && make run` on a fresh clone
   produces an identical `expectations/slice_cc_en_small_v1.json` (compare
   SHA-256). If anything varies, it's a bug in `01_build_suite.py`.
3. **Data Docs render.** `gx/uncommitted/data_docs/local_site/index.html`
   opens in a browser, shows the suite, and shows the latest validation.
4. **Each Expectation has a rationale.** `jq '.expectations[].meta.rationale'
   gx/expectations/slice_cc_en_small_v1.json` returns a non-empty string for
   every Expectation. A null rationale is a review failure.
5. **The slice pin round-trips.** `slice_pin.yaml::input.observed_sha256` is
   filled in after a run; re-running without changing the slice does not
   change the hash.
6. **Envelope figures match the slice.** Expectation values in the suite
   (row range, language shares, score mean) come from `slice_pin.yaml`, not
   from magic numbers in the Python code.
7. **No invented metrics.** If you did not actually run against the slice,
   do not write a `success: true` validation result into the report. Leave
   the file empty or stamp `<!-- unverified: ran against synthetic shard -->`.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 18/26 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                     | Critical? |
|---|-------------------------------------------------------------------------------|-----------|
| 1 | `slice_pin.yaml` identifies slice name + version + source manifest + SHA      | Yes       |
| 2 | Suite is built programmatically from the pin (not hand-edited JSON)           | Yes       |
| 3 | Every Expectation has a `meta.rationale` string                               | Yes       |
| 4 | Suite covers schema, nullability, value ranges, categoricals, distribution    | Yes       |
| 5 | Checkpoint exit code is non-zero on a deliberately broken slice               | Yes       |
| 6 | GX version is pinned in `requirements.lock`                                   | Yes       |
| 7 | Data Docs artefact is produced and committed as a path, not as HTML in repo   | No        |
| 8 | At least one distribution Expectation (not just row-level)                    | Yes       |
| 9 | Each Expectation maps to a column of the slice schema from mod-103            | Yes       |
| 10 | Governance anchor (OWASP ML Top 10 / NIST AI RMF) cited in README            | No        |
| 11 | Unverifiable claims are flagged explicitly (prose caveat or `<!-- unverified: ... -->`) | Yes       |
| 12 | `Makefile` reproduces suite + checkpoint from clean clone                    | No        |
| 13 | At least one hygiene check (regex / uniqueness) is present                   | No        |

## 5. Common mistakes

- **Hand-editing `expectations/*.json`.** The suite is code; hand edits drift
  and are impossible to code-review. Keep `01_build_suite.py` as the only
  path to produce the JSON.
- **Only row-level Expectations.** A suite that checks nullability and types
  but no aggregate will pass on a slice that is 100% empty strings. Add at
  least one `expect_column_mean_...` / `expect_table_row_count_...`.
- **Envelope figures invented from thin air.** Row counts and language
  shares must come from the mod-103 slice spec — `slice_pin.yaml` is the
  single source of truth, not the author's intuition.
- **`result_format: BASIC`.** The default hides which rows failed. Use
  `SUMMARY` (or `COMPLETE` during development) so Data Docs are useful.
- **Analytics enabled.** GX ships with opt-out telemetry; a corpus audit is
  not where you want to leak paths or table shapes. Set
  `analytics_enabled: false`.
- **Treating a green suite as "the data is high quality".** A GX suite
  measures the dimensions you encoded. It says nothing about dimensions you
  didn't — e.g. representativeness (exercise 04), contamination (mod-106),
  PII (mod-110). Call out the scope in the README.
- **Suite in the same repo as the slice without versioning.** A quality bar
  is a piece of policy; version it independently of the slice. A bumped
  slice may want a bumped envelope — keep both diffs legible.

## 6. References

- **Great Expectations documentation** — <https://docs.greatexpectations.io/>.
  Use the version-pinned docs that match `requirements.lock`; the fluent API
  names shifted in 1.x.
- **ISO/IEC 25012:2008** — Data Quality Model. Each Expectation in the
  suite should map to one or more of the 15 characteristics (completeness,
  accuracy, consistency, credibility, currentness, …). Exercise 02 formalises
  the mapping.
- **ISO/IEC 5259-2:2024** — Data quality for analytics and machine
  learning: Data quality measures. Extends 25012 with ML-specific
  characteristics; the companion mapping also lives in Exercise 02.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02
  (Data Poisoning Attack) frames why an un-audited slice is not neutral.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. Browse the current matrix
  at run time for techniques under "Poison Training Data" / "Supply Chain
  Compromise: ML Artifacts".
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MEASURE 2.10`
  is the anchor for "AI system trustworthy characteristics are measured".
- **Local anchors:** mod-103 Exercise 01 (the slice this suite audits);
  Exercise 02 of this module (ISO coverage); Exercise 04 of this module
  (subpopulation audit — scope boundary with the GX suite).
