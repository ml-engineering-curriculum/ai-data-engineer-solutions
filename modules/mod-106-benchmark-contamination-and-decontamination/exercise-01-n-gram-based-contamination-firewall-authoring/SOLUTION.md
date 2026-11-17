# Exercise 01 — N-Gram-Based Contamination Firewall Authoring (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner stands up an
n-gram-based contamination firewall that scans a training-data slice
against a set of **frozen evaluation suites** and produces a per-slice
`contamination_report.yaml` plus a signed quarantine manifest. The
firewall is the pipeline-side control that `mod-106` installs on top
of the `mod-105` dedup primitive — it does not re-discover duplicates;
it answers one question only: *does this training slice overlap
evaluation n-grams at a configured threshold?*

Deliverables:

- A documented **n-gram policy note** pinned before any code runs
  (tokeniser, `N`, canonical form, which eval suites are inside the
  firewall, which are deliberately outside).
- An **eval-suite index** built from the frozen evaluation shards
  (hashes computed over canonicalised n-grams, written to a
  content-addressable sidecar).
- A streaming **slice scanner** that emits, per document, the number
  and identity of matching eval n-grams.
- A **verdict rule** that maps n-gram-overlap counts to one of
  `{clean, contaminated, quarantine_review}` per document and
  aggregates per-source and per-slice rates.
- A `contamination_report.yaml` — **§1 identity, §2 n-gram policy,
  §3 per-eval-suite match counts, §4 per-source contamination rate,
  §5 quarantine manifest reference** — that `mod-109` can sign.

Governance framing. Benchmark contamination is a measurement-integrity
failure that flows from `mod-104` (data quality) and `mod-105` (dedup)
into `mod-111` (quality monitoring). The OWASP Machine Learning
Security Top 10 governance anchor is **ML02:2023 Data Poisoning
Attack** — benchmark contamination is a subtype of training-data
contamination because the eval set becomes part of the training set
and silently biases the measurement surface the organisation uses to
make go/no-go decisions. The MITRE ATLAS techniques in scope are
**AML.T0019 Publish Poisoned Datasets** (an attacker leaks benchmark
items into open web dumps the pipeline ingests) and **AML.T0020
Poison Training Data**. The NIST AI Risk Management Framework control
anchors are **MEASURE 2.7** (AI system security and resilience are
measured) and **MEASURE 2.3** (AI system performance evaluated as
intended) — a contaminated eval directly violates MEASURE 2.3 because
the "performance" number is not of the system under test. Cite the
exact framework URLs listed in the References section rather than
paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── policy_note.md                 # written before any code; firewall contract
├── code/
│   ├── canonicalise.py            # same canonical form as mod-105 ex-01
│   ├── ngram.py                   # tokenisation + sliding n-gram window
│   ├── build_eval_index.py        # hash eval n-grams into a Bloom + oracle set
│   ├── scan_slice.py              # stream training docs, emit match counts
│   ├── verdict.py                 # per-doc {clean|contaminated|review} rule
│   ├── quarantine.py              # route contaminated docs + sign manifest
│   └── emit_report.py             # writes contamination_report.yaml
├── eval_suites/                   # frozen, content-addressed eval shards
│   ├── suite_a.jsonl              # e.g., an academic QA eval
│   ├── suite_a.jsonl.sha256
│   ├── suite_b.jsonl              # e.g., a reasoning eval
│   └── suite_b.jsonl.sha256
└── results/
    ├── eval_index.parquet         # (suite_id, ngram_hash_hi, ngram_hash_lo)
    ├── eval_index.parquet.sha256
    ├── eval_bloom.bin             # bloom filter over all eval n-grams
    ├── doc_matches.parquet        # per-doc: suite_id, matched_ngram_count
    ├── per_source_rates.parquet
    ├── quarantine/                # routed docs grouped by verdict code
    ├── quarantine_manifest.json
    ├── quarantine_manifest.json.sig
    └── contamination_report.yaml
```

### 2.2 `policy_note.md` — write this before any code

```markdown
# Firewall policy for slice <slice-id>

**Decision date:** 2026-11-17
**Author:** <operator email>
**Scope:** this policy applies only to slice <slice-id>. Any change to
`N`, the tokeniser, the canonical form, or the eval-suite set
invalidates `results/eval_index.parquet`, `results/doc_matches.parquet`,
and `results/contamination_report.yaml`. A changed policy requires
re-indexing and a full rescan.

## N and tokenisation

| Layer           | Choice                                      | Rationale                                                                                                       |
|-----------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `N`             | 13                                          | Default from the GPT-3 contamination test (Brown et al. 2020, §C.1). Longer N drops recall, shorter drops precision. |
| Token unit      | whitespace-split tokens on the canonical form | Keeps the firewall tokeniser portable across training tokenisers; see §2 of mod-105 ex-01 for the canonical form. |
| Canonical form  | NFC + collapse-whitespace + preserve-case   | Identical to mod-105 ex-01 so eval and training are compared against the same byte primitive.                   |
| Hash            | xxh3-128 over UTF-8 canonical n-gram bytes  | 128-bit fingerprint; collision probability is negligible at eval-suite scale (≤ 10^7 n-grams).                  |

## Firewall surface

| Eval suite id | Role                                                | In firewall? | Rationale                                                                 |
|---------------|-----------------------------------------------------|--------------|---------------------------------------------------------------------------|
| suite_a       | academic-QA held-out                                | yes          | The decision benchmark for this model line.                                |
| suite_b       | reasoning held-out                                  | yes          | Reported in model cards; must stay clean.                                  |
| suite_c       | internal canary eval (never published)              | yes          | Kept inside the firewall to catch pipeline regressions; never in training. |
| suite_d       | public-domain eval we deliberately include          | no           | Part of the training mixture by design; documented as excluded.            |

## Verdict thresholds

| Verdict             | Rule                                                             |
|---------------------|------------------------------------------------------------------|
| `clean`             | 0 matching eval n-grams in the doc.                              |
| `quarantine_review` | 1–`T` matching eval n-grams; ai-risk-engineer reviews.           |
| `contaminated`      | > `T` matching eval n-grams OR any single suite's doc appears verbatim. |

`T` is set to **1** for `N = 13`: at `N = 13` on canonicalised text, a
single matching n-gram is strong evidence of shared provenance. The
`quarantine_review` bucket exists for borderline cases (quotations,
boilerplate) that the peer triages.
```

Commit this file in its own commit before any `code/*.py`. The
`git log --follow policy_note.md` trail is the audit evidence that
the firewall contract pre-dates the implementation.

### 2.3 `code/canonicalise.py`

Reuse the canonical form from `mod-105` exercise-01 verbatim. Repeating
it here would drift; import it from the shared module or vendor it
behind the same `NORMALISATION_VERSION` constant.

```python
# code/canonicalise.py
from __future__ import annotations

import unicodedata

NORMALISATION_VERSION = "nfc-ws-collapse-v1"


def canonicalise(text: str) -> str:
    collapsed = " ".join(text.strip().split())
    return unicodedata.normalize("NFC", collapsed)
```

### 2.4 `code/ngram.py`

```python
# code/ngram.py
# Sliding-window n-gram generator over canonicalised whitespace tokens.
from __future__ import annotations

from collections.abc import Iterator

import xxhash

from canonicalise import canonicalise

N = 13


def tokens(text: str) -> list[str]:
    return canonicalise(text).split()


def ngrams(text: str, n: int = N) -> Iterator[tuple[int, int]]:
    """Yield (hi, lo) halves of the xxh3-128 hash of each n-gram.

    Splitting the 128-bit hash into two 64-bit integers keeps Parquet
    and DuckDB storage efficient and avoids BigInt arithmetic in SQL.
    """
    toks = tokens(text)
    if len(toks) < n:
        return
    for i in range(len(toks) - n + 1):
        gram = " ".join(toks[i : i + n]).encode("utf-8")
        h = xxhash.xxh3_128_intdigest(gram)
        yield (h >> 64), (h & ((1 << 64) - 1))
```

Design notes defensible at review:

- **Tokenisation is whitespace, not a learned tokeniser.** The firewall
  must not depend on the training tokeniser — the eval index is built
  once and reused across every training-tokeniser variant.
- **Hash is xxh3-128.** SHA-256 per n-gram would make the index pass
  10–30× slower with no correctness benefit at eval-suite scale.
  Collision analysis: at 10^7 eval n-grams, birthday-bound collision
  probability is < 10^-24. This is equivalent to zero.
- **`i + n` boundary.** The sliding window stops at `len(toks) - n + 1`
  so the last n-gram is `toks[-n:]`. Off-by-one here silently drops
  the final n-gram of every document.

### 2.5 `code/build_eval_index.py`

```python
# code/build_eval_index.py
# Build the eval n-gram index as (Bloom filter, parquet oracle set).
# The Bloom filter is the hot path; the parquet set is the oracle
# verifier used by scan_slice to resolve Bloom false positives.
from __future__ import annotations

import argparse
import hashlib
import json
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
from pybloom_live import BloomFilter

from ngram import ngrams

BLOOM_FP_RATE = 1e-5  # one order below mod-105's 1e-4 — eval index is tiny
                      # and we pay for precision at scan time.


def build_index(eval_files: list[Path], output_dir: Path) -> dict:
    output_dir.mkdir(parents=True, exist_ok=True)
    rows: list[tuple[str, int, int]] = []
    suite_counts: dict[str, int] = {}
    for path in eval_files:
        suite_id = path.stem
        count = 0
        for line in path.read_text().splitlines():
            record = json.loads(line)
            for text in iter_eval_strings(record):
                for hi, lo in ngrams(text):
                    rows.append((suite_id, hi, lo))
                    count += 1
        suite_counts[suite_id] = count

    bf = BloomFilter(capacity=max(len(rows), 1), error_rate=BLOOM_FP_RATE)
    for _, hi, lo in rows:
        bf.add((hi, lo))
    with (output_dir / "eval_bloom.bin").open("wb") as fh:
        bf.tofile(fh)

    tbl = pa.table(
        {
            "suite_id": [r[0] for r in rows],
            "ngram_hash_hi": [r[1] for r in rows],
            "ngram_hash_lo": [r[2] for r in rows],
        }
    )
    pq.write_table(tbl, output_dir / "eval_index.parquet")

    sidecar = output_dir / "eval_index.parquet.sha256"
    h = hashlib.sha256((output_dir / "eval_index.parquet").read_bytes()).hexdigest()
    sidecar.write_text(h + "\n")

    return {"ngrams_indexed_per_suite": suite_counts, "ngrams_total": len(rows)}


def iter_eval_strings(record: dict) -> list[str]:
    """Which fields of the eval record count as 'eval text'.

    Chapter policy: both question and reference answers are in-scope,
    because benchmark memorisation is as damaging for answers as for
    questions (a model that memorised the gold answer will score well
    without memorising the question text).
    """
    out = []
    for key in ("question", "query", "input", "prompt", "answer", "target",
                "gold", "reference"):
        v = record.get(key)
        if isinstance(v, str):
            out.append(v)
        elif isinstance(v, list):
            out.extend(x for x in v if isinstance(x, str))
    return out


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--eval-dir", required=True, type=Path)
    ap.add_argument("--output-dir", required=True, type=Path)
    ap.add_argument("--metrics-out", required=True, type=Path)
    args = ap.parse_args()
    eval_files = sorted(args.eval_dir.glob("*.jsonl"))
    metrics = build_index(eval_files, args.output_dir)
    args.metrics_out.write_text(json.dumps(metrics, indent=2, sort_keys=True))
```

Two design choices worth defending at review:

- **Bloom FP rate at 1e-5, not the mod-105 default of 1e-4.** The
  firewall's job is to be precise; false positives force expensive
  oracle lookups and noisy `quarantine_review` queues. The eval index
  is small enough (≤ 10^7 entries typically) that one extra order of
  magnitude of FP precision is cheap.
- **Oracle set is Parquet, not a Python `set` pickle.** The Parquet
  file is signable at the slice artifact layer (`mod-109`) and can be
  queried from DuckDB without re-materialising the full set in memory.

### 2.6 `code/scan_slice.py`

```python
# code/scan_slice.py
# Stream training docs. For each doc, compute its n-gram hashes,
# query the Bloom filter, verify positives against the Parquet oracle.
from __future__ import annotations

import argparse
from collections import defaultdict
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
from pybloom_live import BloomFilter

from ngram import ngrams


def load_bloom(path: Path) -> BloomFilter:
    with path.open("rb") as fh:
        return BloomFilter.fromfile(fh)


def load_oracle(path: Path) -> dict[tuple[int, int], set[str]]:
    tbl = pq.read_table(path)
    oracle: dict[tuple[int, int], set[str]] = defaultdict(set)
    for suite, hi, lo in zip(
        tbl.column("suite_id").to_pylist(),
        tbl.column("ngram_hash_hi").to_pylist(),
        tbl.column("ngram_hash_lo").to_pylist(),
    ):
        oracle[(hi, lo)].add(suite)
    return oracle


def scan(slice_parquet: Path, bloom_path: Path, oracle_path: Path,
         output_dir: Path) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    bf = load_bloom(bloom_path)
    oracle = load_oracle(oracle_path)

    rows: list[tuple[str, str, int]] = []  # (doc_id, suite_id, match_count)
    reader = pq.ParquetFile(slice_parquet)
    for batch in reader.iter_batches(batch_size=10_000,
                                      columns=["doc_id", "source", "text"]):
        for doc_id, source, text in zip(
            batch.column("doc_id").to_pylist(),
            batch.column("source").to_pylist(),
            batch.column("text").to_pylist(),
        ):
            per_suite_counts: dict[str, int] = defaultdict(int)
            for hi, lo in ngrams(text):
                if (hi, lo) in bf:
                    suites = oracle.get((hi, lo), set())  # resolves FPs
                    for s in suites:
                        per_suite_counts[s] += 1
            for suite, c in per_suite_counts.items():
                rows.append((doc_id, suite, c))
            # record source alongside doc_id in a sidecar parquet
    pa.table(
        {
            "doc_id": [r[0] for r in rows],
            "suite_id": [r[1] for r in rows],
            "matched_ngram_count": [r[2] for r in rows],
        }
    ).to_parquet(output_dir / "doc_matches.parquet")
```

Design notes:

- **Bloom-then-oracle.** The Bloom filter is the hot path; the oracle
  is only consulted for positive hits. At `FP = 1e-5` the oracle is
  touched roughly once per 100k n-grams on unrelated training text.
- **Per-suite granularity is load-bearing.** A document that matches
  five n-grams in one suite is contaminated; the same document
  matching one n-gram in each of five suites is probably boilerplate.
  The verdict rule in §2.7 reads the per-suite column, not the sum.

### 2.7 `code/verdict.py`

```python
# code/verdict.py
# Per-doc verdict from the per-suite match counts.
from __future__ import annotations

from typing import Literal

Verdict = Literal["clean", "quarantine_review", "contaminated"]
THRESHOLD_T = 1  # tuned in policy_note.md; a single N=13 n-gram is enough.


def doc_verdict(per_suite_counts: dict[str, int]) -> Verdict:
    if not per_suite_counts:
        return "clean"
    max_hits = max(per_suite_counts.values())
    if max_hits > THRESHOLD_T:
        return "contaminated"
    # Exactly THRESHOLD_T in one or more suites, no single suite exceeded T.
    return "quarantine_review"
```

### 2.8 `code/quarantine.py`

```python
# code/quarantine.py
# Route contaminated / review docs to a signed quarantine bucket.
from __future__ import annotations

import argparse
import hashlib
import json
import subprocess
from pathlib import Path


def route_and_sign(verdict_parquet: Path, slice_parquet: Path,
                   output_dir: Path, cosign_key: str) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    (output_dir / "contaminated").mkdir(exist_ok=True)
    (output_dir / "quarantine_review").mkdir(exist_ok=True)
    # … join verdict_parquet with slice_parquet on doc_id and write each
    # bucket as a parquet shard. Pseudocoded for brevity — the critical
    # correctness property is that the sum of rows across the three
    # buckets (clean, contaminated, quarantine_review) equals the input
    # slice's row count (row-conservation).

    manifest = {
        "slice_id": "<set via CLI>",
        "buckets": {
            name: {"row_count": _rows(output_dir / name),
                    "sha256": _sha(output_dir / name)}
            for name in ("contaminated", "quarantine_review")
        },
    }
    manifest_path = output_dir / "quarantine_manifest.json"
    manifest_path.write_text(json.dumps(manifest, indent=2, sort_keys=True))

    subprocess.check_call([
        "cosign", "sign-blob", "--yes",
        "--key", cosign_key,
        "--output-signature", str(manifest_path) + ".sig",
        str(manifest_path),
    ])


def _rows(bucket: Path) -> int:
    ...  # sum parquet row counts in the directory


def _sha(bucket: Path) -> str:
    ...  # SHA-256 over sorted-filename-concatenated parquet content hashes
```

The sign step is **not optional** — the quarantine manifest is a
load-bearing artifact for `mod-109` provenance and must be verifiable
before `mod-108` synthetic generation reads from the `clean` bucket.

### 2.9 `code/emit_report.py`

```python
# code/emit_report.py
# Writes contamination_report.yaml, structured as:
#   §1 identity   (slice_id, slice_version, toolchain, operator, run window)
#   §2 ngram policy  (N, tokeniser, canonical form, hash algo)
#   §3 per_eval_suite_match_counts  (suite_id → matched docs / matched ngrams)
#   §4 per_source_contamination_rate  (source → contaminated ratio)
#   §5 quarantine_manifest_reference  (path + sha256 + signature path)
#   §6 ai_risk_engineer_review  (status defaults to `pending_review`)
from __future__ import annotations

import argparse
import hashlib
import json
import socket
import subprocess
from datetime import datetime, timezone
from pathlib import Path

import pyarrow.parquet as pq
import yaml


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--slice-id", required=True)
    ap.add_argument("--slice-version", required=True)
    ap.add_argument("--operator", required=True)
    ap.add_argument("--slice-parquet", required=True, type=Path)
    ap.add_argument("--doc-matches", required=True, type=Path)
    ap.add_argument("--quarantine-manifest", required=True, type=Path)
    ap.add_argument("--quarantine-signature", required=True, type=Path)
    ap.add_argument("--eval-index", required=True, type=Path)
    ap.add_argument("--policy-note", required=True, type=Path)
    ap.add_argument("--run-start-utc", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    matches = pq.read_table(args.doc_matches).to_pandas()
    slice_tbl = pq.read_table(args.slice_parquet, columns=["doc_id", "source"])

    per_suite = matches.groupby("suite_id").agg(
        matched_docs=("doc_id", "nunique"),
        matched_ngrams=("matched_ngram_count", "sum"),
    ).to_dict(orient="index")

    contaminated_docs = set(
        matches.groupby("doc_id")["matched_ngram_count"].max().loc[
            lambda s: s > 0
        ].index
    )
    slice_df = slice_tbl.to_pandas()
    slice_df["contaminated"] = slice_df["doc_id"].isin(contaminated_docs)
    per_source = (
        slice_df.groupby("source")["contaminated"]
        .mean()
        .round(6)
        .to_dict()
    )

    git_sha = subprocess.check_output(["git", "rev-parse", "HEAD"]).decode().strip()
    report = {
        "slice_id": args.slice_id,
        "slice_version": args.slice_version,
        "toolchain": {
            "name": "mod-106-exercise-01-ngram-firewall",
            "git_sha": git_sha,
            "policy_note_sha256": sha256_file(args.policy_note),
            "eval_index_sha256": sha256_file(args.eval_index),
        },
        "operator": args.operator,
        "run_start_utc": args.run_start_utc,
        "run_end_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "compute_shape": {"host": socket.gethostname()},
        "ngram_policy": {
            "n": 13,
            "tokeniser": "whitespace",
            "canonical_form_version": "nfc-ws-collapse-v1",
            "hash_algo": "xxh3_128",
        },
        "per_eval_suite_match_counts": per_suite,
        "per_source_contamination_rate": per_source,
        "quarantine_manifest": {
            "path": str(args.quarantine_manifest),
            "sha256": sha256_file(args.quarantine_manifest),
            "signature_path": str(args.quarantine_signature),
        },
        "ai_risk_engineer_review": {
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

Determinism safeguards:

- `yaml.safe_dump(..., sort_keys=False)` paired with an explicitly
  ordered dict — the only non-deterministic field is `run_end_utc`,
  which §3 of validation excludes from the determinism check.
- `ai_risk_engineer_review.status` defaults to `pending_review`; the
  firewall author must not pre-populate a signed verdict. The peer
  signs separately, per `mod-105` exercise-05's handoff convention.

### 2.10 `Makefile`

```makefile
.PHONY: all index scan verdict quarantine report clean

all: index scan verdict quarantine report

index:
	python code/build_eval_index.py --eval-dir eval_suites/ \
		--output-dir results/ --metrics-out results/index_metrics.json

scan:
	python code/scan_slice.py \
		--slice data/slice.parquet \
		--bloom results/eval_bloom.bin \
		--oracle results/eval_index.parquet \
		--output-dir results/

verdict:
	python code/apply_verdict.py \
		--matches results/doc_matches.parquet \
		--slice data/slice.parquet \
		--output results/per_doc_verdict.parquet

quarantine:
	python code/quarantine.py \
		--verdict results/per_doc_verdict.parquet \
		--slice data/slice.parquet \
		--output-dir results/quarantine/ \
		--cosign-key $$COSIGN_KEY

report:
	python code/emit_report.py \
		--slice-id slice-example-v1 --slice-version 1.0.0 \
		--operator $$(git config user.email) \
		--slice-parquet data/slice.parquet \
		--doc-matches results/doc_matches.parquet \
		--quarantine-manifest results/quarantine/quarantine_manifest.json \
		--quarantine-signature results/quarantine/quarantine_manifest.json.sig \
		--eval-index results/eval_index.parquet \
		--policy-note policy_note.md \
		--run-start-utc $$(date -u +%Y-%m-%dT%H:%M:%SZ) \
		--output results/contamination_report.yaml

clean:
	rm -rf results/*.parquet results/eval_bloom.bin \
	       results/contamination_report.yaml results/quarantine/
```

## 3. Validation steps

1. **`policy_note.md` pre-dates the code.** `git log --follow
   policy_note.md | tail -1` returns a commit older than any
   `code/*.py` commit. A retro-fitted policy fails review.
2. **Eval-index sidecar hash matches the Parquet.**
   `sha256sum results/eval_index.parquet` equals
   `cat results/eval_index.parquet.sha256`.
3. **Row conservation across quarantine buckets.** The sum of row
   counts in `results/quarantine/{clean,quarantine_review,contaminated}`
   equals the pre-scan slice row count.
4. **Bloom false-positive rate within budget.** Run an oracle-only
   scan (ignore the Bloom filter) over a 10k-doc held-out training
   set that is known clean; observed FPs divided by n-gram count must
   be below `2 × BLOOM_FP_RATE`.
5. **Report determinism.** `make clean && make report && sha256sum
   results/contamination_report.yaml` twice; the two hashes match
   modulo `run_end_utc`. Non-determinism elsewhere is a bug.
6. **Quarantine signature verifies.**
   `cosign verify-blob --key <pub> \
      --signature results/quarantine/quarantine_manifest.json.sig \
      results/quarantine/quarantine_manifest.json` exits 0.
7. **Pending-review status is literal.**
   `yq '.ai_risk_engineer_review.status'
      results/contamination_report.yaml`
   returns `pending_review`. The firewall author must not sign on
   behalf of the peer.
8. **Known-contaminated canary round-trips.** Inject a canary document
   that embeds a known eval n-gram into the slice; the verdict must
   be `contaminated` and the doc must land in
   `results/quarantine/contaminated/`.
9. **Known-clean canary round-trips.** A canary document containing
   no eval n-grams must land in the `clean` bucket with zero matches.
10. **Per-source rates present.** Every distinct `source` value in the
    slice appears as a key in `per_source_contamination_rate`. A `0.0`
    rate is valid and distinct from a missing key.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `policy_note.md` exists and is older than any `code/*.py` commit                                                    | Yes       |
| 2  | `N`, tokeniser, canonical form, hash algo are pinned in both the policy note and `§2` of the report                  | Yes       |
| 3  | Eval-suite index built once; sidecar `.sha256` matches the Parquet                                                  | Yes       |
| 4  | Bloom filter + Parquet oracle architecture (not Bloom alone, not Python-set alone)                                   | Yes       |
| 5  | Verdict rule reads per-suite counts, not a sum across suites                                                        | Yes       |
| 6  | Per-doc verdict covers `{clean, quarantine_review, contaminated}` with documented thresholds                        | Yes       |
| 7  | Row-conservation holds across quarantine buckets                                                                    | Yes       |
| 8  | `quarantine_manifest.json` is signed with cosign; signature verifies in validation step 6                           | Yes       |
| 9  | `ai_risk_engineer_review.status` defaults to `pending_review`; no self-sign                                         | Yes       |
| 10 | Report determinism check (two runs → byte-identical YAML modulo `run_end_utc`) passes                                | Yes       |
| 11 | Per-source contamination rates cover every source value in the slice                                                | Yes       |
| 12 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0019/T0020, NIST AI RMF MEASURE 2.3 / 2.7) cited          | No        |
| 13 | Canary docs (one contaminated, one clean) round-trip through the firewall and land in the expected buckets           | Yes       |

## 5. Common mistakes

- **Scanning raw text instead of the canonical form.** Trailing
  whitespace or Unicode-variant spelling will miss a verbatim eval
  match. Canonicalise first, then shingle, every time.
- **Using the training tokeniser.** The firewall must be portable
  across tokeniser versions; whitespace tokens on the canonical form
  is the lowest-common-denominator choice. A BPE-tokenised firewall
  silently drifts whenever the tokeniser is retrained.
- **Fingerprinting at `N < 10`.** Low `N` catches boilerplate as
  "contamination" and floods the review queue. GPT-3's published
  default (`N = 13`) is the safe starting point; lower `N` demands
  tuned thresholds and a documented false-positive budget.
- **Threshold `T > 1` without a measurement.** Thresholds > 1 are
  valid only when a false-positive-rate study on real training data
  justifies them. A hand-picked `T = 3` with no study is an invented
  parameter.
- **Only logging aggregate match counts.** The `per_source` block is
  the signal that catches a single contaminated source (e.g., a
  mirrored eval leaderboard) in an otherwise-clean slice.
- **Pre-signing `ai_risk_engineer_review`.** Firewall authors never
  own the sign-off. The default is `pending_review`; the peer signs
  after reviewing the quarantine bucket.
- **Letting the firewall silently delete docs.** The firewall *routes*;
  it never deletes. The quarantine bucket is the lineage record for
  `mod-109`.
- **Omitting the eval-suite content-addressed sidecar.** Without a
  sidecar hash on the eval shards, a swapped-out eval (common when
  benchmarks ship updates) invalidates the firewall silently.
- **Treating `suite_d`-style excluded evals as pipeline bugs.** The
  policy note is the place where deliberately-included evals are
  documented; excluding them from the firewall is a contract, not
  an omission.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Treats benchmark contamination as a subtype of training-data
  poisoning: the eval set has been poisoned into the training set.
- **MITRE ATLAS — AML.T0019 Publish Poisoned Datasets, AML.T0020
  Poison Training Data:** <https://atlas.mitre.org/>. The
  adversary-facing anchor for a poisoned-eval scenario where an
  attacker deliberately leaks benchmark items into open web dumps.
- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE 2.7:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 ("AI system performance or assurance criteria … are measured
  qualitatively or quantitatively as intended") is the control the
  firewall preserves: a contaminated eval violates "as intended".
  MEASURE 2.7 anchors the security side.
- **GPT-3 (Brown et al. 2020), §C "Details of Test Set Contamination
  Studies":** the public source for the `N = 13` n-gram contamination
  convention this firewall adopts. Cite the specific section when
  defending the choice of `N`.
- **lm-evaluation-harness decontamination tooling:** EleutherAI's
  `lm-evaluation-harness` ships a reference decontamination recipe
  that Exercise 02 runs end-to-end. The firewall in this exercise
  implements the same idea at pipeline-ingest time, not at
  evaluation time.
- **`xxhash` reference implementation:** <https://github.com/Cyan4973/xxHash>.
  xxh3-128 is the n-gram fingerprint convention; its 128-bit output
  makes collisions negligible at eval-suite scale.
- **`pybloom_live`:** <https://github.com/joseph-fox/python-bloomfilter>.
  Sizes the bit array and number of hash functions from
  `(capacity, error_rate)`.
- **Local anchors:** `mod-105` exercise-01 (canonical form and the
  `NORMALISATION_VERSION` constant this firewall reuses); `mod-105`
  exercise-05 (`pending_review` sign-off handoff convention);
  `mod-109` (Sigstore signing of the quarantine manifest);
  `mod-111` (how the firewall feeds the quality-regression monitor).
