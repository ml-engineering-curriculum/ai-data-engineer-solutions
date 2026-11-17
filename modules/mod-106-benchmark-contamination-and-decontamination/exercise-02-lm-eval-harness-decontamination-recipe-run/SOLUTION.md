# Exercise 02 — LM Eval Harness Decontamination Recipe Run (vs the Dolma Practice) (Solution)

## 1. Solution overview

This is an **implementation + comparison** exercise. The learner runs
EleutherAI's `lm-evaluation-harness` decontamination recipe over the
same slice that Exercise 01's firewall processed, re-runs the
`dolma`-style decontamination practice over the same slice, and
writes a one-page `choice.md` that defends how the two toolchains
differ on the same input and when to reach for each.

Deliverables:

- A pinned `requirements.lock` with the exact `lm-evaluation-harness`
  and `dolma` versions used; both are recorded in the report's
  `§1 toolchain` block.
- A `recipe/` directory containing the harness decontamination
  configuration and the invocation script.
- A `dolma_tagger/` directory with the `dolma` decontamination
  attributes configuration.
- `results/harness_matches.parquet` and
  `results/dolma_matches.parquet` with a shared schema so
  `compare_matches.py` can compute the overlap.
- `results/agreement.json` — Jaccard of matched-doc sets plus a
  Venn-style breakdown (`harness_only`, `dolma_only`, `both`).
- `results/contamination_report.yaml` — extends Exercise 01's report
  with a new `§7 recipe_runs` block that records both toolchains'
  parameters, versions, and match counts.
- `choice.md` — one page defending which recipe the organisation
  chooses as the pipeline default and why the other stays in the
  toolbox.

Governance framing. Exercise 01 built the pipeline-side firewall;
Exercise 02 answers the auditor's follow-up question: *does the
firewall agree with the community-standard decontamination recipes?*
If the two production recipes disagree on the same slice, the policy
note in Exercise 01 is incomplete and the firewall's `§6 pending_review`
queue should swell until the peer explains the gap. The OWASP ML Top
10 governance anchor remains **ML02:2023 Data Poisoning Attack**;
MITRE ATLAS **AML.T0019 Publish Poisoned Datasets** is the
adversary-side anchor; NIST AI RMF **MEASURE 2.3** (performance
measured as intended) and **MEASURE 4.2** (regularly evaluate AI
system trustworthy characteristics … feedback from relevant AI
actors) are the control anchors. Cite the exact URLs in the
References section rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── requirements.lock
├── Makefile
├── choice.md
├── recipe/
│   ├── decontaminate_config.yaml       # harness config
│   └── run_harness.sh
├── dolma_tagger/
│   ├── decontamination.yaml            # dolma attributes config
│   └── run_dolma.sh
├── code/
│   ├── normalise_matches.py            # harness → shared schema
│   ├── normalise_dolma.py              # dolma → shared schema
│   ├── compare_matches.py              # Jaccard + Venn + per-source delta
│   └── extend_report.py                # writes §7 recipe_runs
└── results/
    ├── harness_raw/                    # whatever the harness emits
    ├── dolma_raw/                      # whatever dolma emits
    ├── harness_matches.parquet
    ├── dolma_matches.parquet
    ├── agreement.json
    └── contamination_report.yaml       # §7 populated
```

### 2.2 `recipe/decontaminate_config.yaml` — lm-evaluation-harness

```yaml
# recipe/decontaminate_config.yaml
# EleutherAI lm-evaluation-harness decontamination inputs.
# The harness consumes a "training" corpus and an eval-task list, and
# emits a per-eval-task match report at a configured n-gram width.
# Pin the version with `lm-eval --version` and record it in the report.

n_gram_size: 13
training_data:
  format: jsonl
  path: data/slice.jsonl
  text_field: text
  doc_id_field: doc_id
tasks:
  - task_a            # must match the task names in the harness registry
  - task_b
  - task_c
output_dir: results/harness_raw/
```

Design notes defensible at review:

- **`n_gram_size: 13`** matches Exercise 01's firewall. If you pick a
  different `N` for the harness, the agreement study in §2.6 is
  measuring two different questions and must say so.
- **Task list is explicit.** The harness can iterate the full task
  registry; locking the list to the firewall's protected suites keeps
  the comparison honest. Add any task from Exercise 01's
  `suite_a..suite_c` list and *exclude* `suite_d` (the task the
  organisation deliberately includes in training).
- **Invocation path.** Exact command for the version pinned in
  `requirements.lock`. The decontamination entry-point name has moved
  more than once in the harness's history; record the one you use.

  ```bash
  # recipe/run_harness.sh
  set -euo pipefail
  lm_eval decontaminate \
    --config recipe/decontaminate_config.yaml \
    --output results/harness_raw/
  ```

### 2.3 `dolma_tagger/decontamination.yaml` — dolma

```yaml
# dolma_tagger/decontamination.yaml
# dolma ships decontamination as a tagger over attributes; run it with
# the attributes-cli and emit per-doc attributes that encode match
# counts per eval suite.
# Reference: https://github.com/allenai/dolma

streams:
  - name: slice_example_v1
    documents:
      - data/slice/*.jsonl.gz
    attributes:
      - decontamination
    decontamination:
      n_gram_size: 13
      bloom:
        size_in_bytes: 268_435_456     # 256 MB bloom
        estimated_elements: 10_000_000
        desired_false_positive_rate: 0.00001
      sources:
        - name: suite_a
          documents:
            - eval_suites/suite_a.jsonl.gz
        - name: suite_b
          documents:
            - eval_suites/suite_b.jsonl.gz
        - name: suite_c
          documents:
            - eval_suites/suite_c.jsonl.gz
```

Design notes:

- **Bloom sizing is explicit.** Dolma's decontamination tagger needs
  Bloom capacity and FP rate. We size for 10M eval n-grams at
  `FP = 1e-5`, matching Exercise 01.
- **Sources match firewall suites.** The firewall and the dolma
  tagger must scan the *same* eval shards; otherwise the agreement
  study in §2.6 is uninterpretable.
- **`jsonl.gz` not `jsonl`.** Dolma expects gzipped shards by default.
  Converting is a one-liner; omitting the `.gz` is the most common
  "my dolma run finished in 0.3 seconds" failure mode.

### 2.4 `code/normalise_matches.py` and `code/normalise_dolma.py`

Both toolchains produce their own output shapes. To compare them,
normalise to a shared schema:

```python
# code/normalise_matches.py
# Shared schema:
#   (doc_id: str, suite_id: str, matched_ngram_count: int,
#    toolchain: str)
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def normalise_harness(raw_dir: Path, output: Path) -> None:
    rows = []
    for jsonl in raw_dir.glob("*.jsonl"):
        for line in jsonl.read_text().splitlines():
            r = json.loads(line)
            # The harness emits per-task match records; map directly.
            rows.append(
                (r["doc_id"], r["task"], int(r["matched_ngram_count"]),
                 "lm-evaluation-harness")
            )
    pa.table(
        {
            "doc_id": [r[0] for r in rows],
            "suite_id": [r[1] for r in rows],
            "matched_ngram_count": [r[2] for r in rows],
            "toolchain": [r[3] for r in rows],
        }
    ).to_parquet(output)
```

```python
# code/normalise_dolma.py
# Dolma emits per-doc attribute files. Decontamination attributes are
# per-suite integer counts under `attributes.decontamination.<suite>`.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def normalise_dolma(raw_dir: Path, output: Path) -> None:
    rows = []
    for shard in raw_dir.glob("**/*.jsonl.gz"):
        # read + decompress with gzip; omitted for brevity.
        for record in iter_records(shard):
            doc_id = record["id"]
            attrs = record.get("attributes", {}).get("decontamination", {})
            for suite, count in attrs.items():
                if int(count) == 0:
                    continue
                rows.append((doc_id, suite, int(count), "dolma"))
    pa.table(
        {
            "doc_id": [r[0] for r in rows],
            "suite_id": [r[1] for r in rows],
            "matched_ngram_count": [r[2] for r in rows],
            "toolchain": [r[3] for r in rows],
        }
    ).to_parquet(output)
```

Design notes:

- **Zero-match rows are dropped.** A `matched_ngram_count = 0` row is
  not evidence of contamination; keeping it would make the "matched
  doc set" different sizes on each side for silly reasons. Document
  this choice in `choice.md`.
- **`suite_id` naming.** The harness uses the task name; dolma uses
  the source name. Map them to the same `suite_id` values the
  firewall uses so the comparison is one-to-one.

### 2.5 `code/compare_matches.py`

```python
# code/compare_matches.py
# Jaccard of matched-doc sets, plus Venn breakdown and per-source
# delta. Agreement is on *document identity*, not on exact match
# counts (the two toolchains tokenise / hash differently so count
# equality is not expected).
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pyarrow.parquet as pq


def compare(harness_path: Path, dolma_path: Path, output: Path) -> None:
    h = pq.read_table(harness_path).to_pandas()
    d = pq.read_table(dolma_path).to_pandas()

    h_pairs = set(zip(h["doc_id"], h["suite_id"]))
    d_pairs = set(zip(d["doc_id"], d["suite_id"]))

    both = h_pairs & d_pairs
    only_h = h_pairs - d_pairs
    only_d = d_pairs - h_pairs
    union = h_pairs | d_pairs

    summary = {
        "doc_suite_pair_counts": {
            "harness_only": len(only_h),
            "dolma_only": len(only_d),
            "both": len(both),
            "union": len(union),
        },
        "jaccard_doc_suite_pairs": (len(both) / len(union)) if union else 1.0,
        "per_suite": {},
    }
    for suite in sorted({s for _, s in union}):
        h_s = {d for d, s in h_pairs if s == suite}
        d_s = {d for d, s in d_pairs if s == suite}
        union_s = h_s | d_s
        summary["per_suite"][suite] = {
            "harness_matched_docs": len(h_s),
            "dolma_matched_docs": len(d_s),
            "jaccard": (len(h_s & d_s) / len(union_s)) if union_s else 1.0,
        }
    output.write_text(json.dumps(summary, indent=2, sort_keys=True))
```

Agreement is measured on **(doc_id, suite_id) pairs**, not on
`matched_ngram_count` equality. The two toolchains tokenise and
canonicalise differently enough that identical counts are unlikely
even when both correctly identify a contaminated document; the
operationally relevant question is *which documents does each
toolchain flag?*

### 2.6 `code/extend_report.py`

```python
# code/extend_report.py
# Open Exercise 01's contamination_report.yaml, append a §7 block, and
# re-emit. Leaves the Exercise 01 fields untouched.
from __future__ import annotations

import argparse
import hashlib
import json
import subprocess
from pathlib import Path

import yaml


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--base-report", required=True, type=Path)
    ap.add_argument("--harness-matches", required=True, type=Path)
    ap.add_argument("--dolma-matches", required=True, type=Path)
    ap.add_argument("--agreement", required=True, type=Path)
    ap.add_argument("--harness-version", required=True)
    ap.add_argument("--dolma-version", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    report = yaml.safe_load(args.base_report.read_text())
    agreement = json.loads(args.agreement.read_text())

    report["recipe_runs"] = {
        "lm_evaluation_harness": {
            "version": args.harness_version,
            "n_gram_size": 13,
            "matches_sha256": sha256_file(args.harness_matches),
            "matched_doc_count": agreement["doc_suite_pair_counts"][
                "harness_only"
            ] + agreement["doc_suite_pair_counts"]["both"],
        },
        "dolma": {
            "version": args.dolma_version,
            "n_gram_size": 13,
            "matches_sha256": sha256_file(args.dolma_matches),
            "matched_doc_count": agreement["doc_suite_pair_counts"][
                "dolma_only"
            ] + agreement["doc_suite_pair_counts"]["both"],
        },
        "agreement": agreement,
    }

    args.output.write_text(yaml.safe_dump(report, sort_keys=False))
```

### 2.7 `choice.md` — one page

The learner's `choice.md` must contain, at minimum:

- A table listing Exercise 01's firewall, the harness recipe, and the
  dolma recipe side-by-side on five decision axes (what it operates
  on, when it runs, how it tokenises, what it outputs, which part of
  the organisation runs it).
- One paragraph defending the chosen *pipeline default*. Reasonable
  defences are "harness is authoritative because it is the eval
  framework's own decontamination tool" or "dolma is authoritative
  because it is the data pipeline's native tagger and already runs
  per shard"; the reviewer looks for *a* defence, not a specific one.
- One paragraph on when the non-default is run anyway (release
  gating, suspected regression, new eval added to the firewall,
  etc.).
- A per-suite note on the biggest delta from `agreement.json` —
  which suite the two toolchains disagree on most, and a hypothesis
  for why.

### 2.8 `Makefile`

```makefile
.PHONY: all harness dolma normalise compare report

all: harness dolma normalise compare report

harness:
	bash recipe/run_harness.sh

dolma:
	bash dolma_tagger/run_dolma.sh

normalise:
	python code/normalise_matches.py \
		--raw-dir results/harness_raw/ \
		--output results/harness_matches.parquet
	python code/normalise_dolma.py \
		--raw-dir results/dolma_raw/ \
		--output results/dolma_matches.parquet

compare:
	python code/compare_matches.py \
		--harness results/harness_matches.parquet \
		--dolma results/dolma_matches.parquet \
		--output results/agreement.json

report:
	python code/extend_report.py \
		--base-report ../exercise-01-n-gram-based-contamination-firewall-authoring/results/contamination_report.yaml \
		--harness-matches results/harness_matches.parquet \
		--dolma-matches results/dolma_matches.parquet \
		--agreement results/agreement.json \
		--harness-version "$$(lm_eval --version 2>/dev/null || echo unknown)" \
		--dolma-version "$$(pip show dolma | awk '/^Version:/ {print $$2}')" \
		--output results/contamination_report.yaml
```

## 3. Validation steps

1. **Versions pinned.** `requirements.lock` records the exact
   installed versions of `lm-evaluation-harness` and `dolma`. Omit
   this and the report is unreproducible.
2. **Same eval shards on both sides.** The harness config and the
   dolma config reference the identical frozen eval files (same
   paths or same SHA-256 content hashes). A mismatch here invalidates
   the agreement study.
3. **Same `N` on both sides.** Both configs specify
   `n_gram_size: 13`; a mismatch is the single most common cause of
   phantom disagreement.
4. **Zero-match rows dropped during normalisation.** Spot-check the
   normalised parquets: `min(matched_ngram_count) >= 1`.
5. **Suite naming is canonical.** The `suite_id` values in both
   `harness_matches.parquet` and `dolma_matches.parquet` map to the
   firewall's `suite_a`/`suite_b`/`suite_c` naming. Do this before
   comparing.
6. **Agreement arithmetic checks.** `harness_only + both ==
   harness_matched_doc_count`, same for dolma. If these do not add
   up, the comparison is broken.
7. **Report is a superset of Exercise 01's.** Diff
   `results/contamination_report.yaml` against
   `../exercise-01-.../results/contamination_report.yaml`; every
   Exercise 01 key is present and unchanged, and the only addition
   is the `recipe_runs` block.
8. **Deterministic re-runs.** Rerun the harness and dolma on the
   same slice + configs; the match parquet SHA-256s are identical.
   If a toolchain is non-deterministic, document the cause in
   `choice.md`.
9. **No invented metrics.** Every number in §7 traces to a field in
   `agreement.json` or a file hash; no hand-entered values.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                               | Critical? |
|----|-------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Both toolchains' versions pinned in `requirements.lock` and recorded in `§7 recipe_runs`                                 | Yes       |
| 2  | Both configs reference the same eval shards (path or content hash)                                                      | Yes       |
| 3  | Both configs specify `n_gram_size: 13` (or both use a shared different `N` with a documented reason)                     | Yes       |
| 4  | Normalisation drops zero-match rows and maps `suite_id` to the firewall's canonical naming                              | Yes       |
| 5  | Agreement is measured on `(doc_id, suite_id)` pairs with Jaccard and Venn breakdown                                     | Yes       |
| 6  | Per-suite agreement block covers every suite in the firewall                                                            | Yes       |
| 7  | `§7 recipe_runs` is added without mutating Exercise 01's fields                                                         | Yes       |
| 8  | Match parquet SHAs are recorded; sidecar files match                                                                    | Yes       |
| 9  | `choice.md` defends a pipeline default and names when the non-default is run anyway                                     | Yes       |
| 10 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0019/T0020, NIST AI RMF MEASURE 2.3 / MEASURE 4.2) cited     | No        |
| 11 | Report determinism holds (same inputs → byte-identical YAML modulo timestamps)                                          | Yes       |

## 5. Common mistakes

- **Comparing counts instead of doc sets.** Two toolchains with
  different tokenisers will report different `matched_ngram_count`
  values even when they agree on which docs are contaminated. The
  agreement metric is on doc-suite pairs, not count equality.
- **Different `N` on each side.** A harness run at `N = 13` and a
  dolma run at `N = 8` can and will disagree on 20–40% of matches
  for mechanical reasons alone. Lock `N` first.
- **Running the harness over the raw slice while running dolma over
  the dedup'd slice.** Both must see the same input; otherwise the
  agreement study is measuring dedup, not decontamination.
- **Letting the harness pull its default task list.** The default
  changes with harness releases; pin the task list in
  `decontaminate_config.yaml`.
- **Treating `pip show dolma` as a provenance substitute.** The
  `git_sha` of the toolchain install is what `§1 toolchain` demands;
  for an unreleased dolma checkout, `pip show` reports `0.0.0.dev0`
  and the git SHA is the only identifier.
- **Overwriting Exercise 01's report in place.** `§7 recipe_runs`
  is additive; a fresh-run report that has lost Exercise 01's
  per-source rates is a regression.
- **Mixing `.jsonl` and `.jsonl.gz` inputs.** Dolma silently
  processes the wrong shards or none at all; verify via log line
  count and parquet row count before trusting the output.
- **Claiming one recipe "catches more" is better.** Higher match
  count ≠ higher precision. The agreement study and the quarantine
  false-positive sampling from Exercise 01 are what justify a
  pipeline default.

## 6. References

- **EleutherAI `lm-evaluation-harness`:**
  <https://github.com/EleutherAI/lm-evaluation-harness>. Pin the
  exact release used; the decontamination subcommand name and
  configuration schema have changed across releases.
- **AllenAI `dolma`:** <https://github.com/allenai/dolma>. The
  `attributes` / `tagger` system includes a decontamination tagger
  configured per source shard. Soldaini et al. 2024 describes the
  decontamination practice adopted for the Dolma corpus.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
- **MITRE ATLAS — AML.T0019 Publish Poisoned Datasets, AML.T0020
  Poison Training Data:** <https://atlas.mitre.org/>.
- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE 4.2:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 ("AI system performance or assurance criteria … are measured
  qualitatively or quantitatively as intended") is violated by a
  contaminated eval; MEASURE 4.2 anchors the practice of soliciting
  external methodology (here, community decontamination recipes).
- **Local anchors:** Exercise 01 (the firewall whose report this
  recipe run extends); `mod-105` Exercise 04 (the toolchain-comparison
  pattern this exercise reuses for `choice.md`); `mod-109` (signing
  the extended `contamination_report.yaml`).
