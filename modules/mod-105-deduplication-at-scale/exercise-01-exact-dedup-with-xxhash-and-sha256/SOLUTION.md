# Exercise 01 — Exact Deduplication With xxhash and SHA-256 (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds the exact-dedup
stage that every subsequent stage in Chapter 03–05 treats as a solved
sub-problem: a documented canonicalisation policy, an xxh128 fingerprint
per document, a SHA-256 audit hash at the slice-artifact level, a
longest-survivor cluster resolver, and the first three sections of the
Chapter 08 `dedup_report.yaml`. The exercise ends with a head-to-head
comparison between a Python-`set` deduplicator and a Bloom-filter
deduplicator on the same input.

The point is not to produce a smaller file. It is to produce a
**reproducible, auditable primitive**:

- The normalisation policy (Unicode form, whitespace rule, case rule,
  encoding) is written down **before any code runs** and lives in
  `§2.normalization` of the dedup report.
- Every document carries an xxh128 fingerprint computed over the same
  canonical bytes.
- The slice artifact carries one SHA-256 — the audit hash — over the
  sorted fingerprint stream, so the slice is signable at the
  Sigstore / in-toto layer `mod-109` will install.
- Cluster resolution is **documented** (longest-survivor), not implicit.
- Per-source removal rates are mandatory (Chapter 08 §3).

Governance framing: exact dedup is one of the compensating controls for
the OWASP ML Top 10 **ML02:2023 — Data Poisoning Attack** family, because
mass-copied adversarial content is cheap for an attacker and
disproportionately surfaces in the long tail of duplicate counts
(Kandpal 2022, picked up in Exercise 02 §4). NIST AI RMF **MEASURE 2.7**
("AI system security and resilience are measured") covers the evidence
side: the dedup report plus signed audit hash is the measurement.
MITRE ATLAS techniques **AML.T0019** (Publish Poisoned Datasets) and
**AML.T0020** (Poison Training Data) are the attacker-side anchors.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── policy_note.md               # the normalisation decision, pre-code
├── comparison.md                # set vs Bloom head-to-head
├── code/
│   ├── canonicalise.py
│   ├── fingerprint.py
│   ├── dedup_set.py             # Implementation A (ground-truth oracle)
│   ├── dedup_bloom.py           # Implementation B (compared to A)
│   ├── cluster_resolve.py       # longest-survivor rule
│   └── emit_report.py           # writes dedup_report.yaml §1–§3
└── results/
    ├── fingerprints.parquet
    ├── fingerprints.parquet.sha256
    ├── removed_docs.parquet
    ├── dedup_slice.parquet
    ├── dedup_slice.sha256
    └── dedup_report.yaml        # §1 + §2 + §3 exact-dedup populated
```

### 2.2 `policy_note.md` — write this before any code

```markdown
# Normalisation policy for slice <slice-id>

**Decision date:** 2026-11-04
**Author:** <operator email>
**Scope:** This policy applies to every document in slice <slice-id> for
exact dedup. Any change invalidates every fingerprint in
`results/fingerprints.parquet` and requires a full re-fingerprint pass
(Chapter 02 §"Exact dedup is not idempotent across normalisation changes").

| Layer        | Choice                                   | Rationale                                                                                                      |
|--------------|------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Unicode form | NFC                                      | Mixed-source corpus; NFKC would collapse full-width / ligature forms we want to keep distinct in English text. |
| Whitespace   | strip + collapse runs to single space    | Catches CDN-injected whitespace and extractor-trailing newlines; acceptable because slice has no code blocks.  |
| Case         | preserve                                 | Casefolding collapses `US` (country) and `us` (pronoun). Default per Chapter 02.                               |
| Encoding     | utf-8 (explicit)                         | Lock the encoding so re-runs on different Python / platform combinations hash the same bytes.                  |

**Fingerprint algorithm:** `xxhash.xxh3_128_intdigest` (128-bit).
**Audit algorithm:** SHA-256 over the sorted-fingerprint stream of the
slice (one hash per slice, not per document — Chapter 02 §"Which hash").
```

Commit this file in its own commit before anything else. `git log --follow
policy_note.md` is the audit trail that the policy pre-dates the code.

### 2.3 `code/canonicalise.py`

```python
# code/canonicalise.py
# Canonical form that every fingerprint is computed over.
# Policy is pinned in policy_note.md and §2 of the dedup report.
from __future__ import annotations

import unicodedata

NORMALISATION_VERSION = "nfc-ws-collapse-v1"


def canonicalise(text: str) -> bytes:
    """Return the canonical byte sequence a fingerprint is computed over.

    Policy (policy_note.md):
      - Unicode NFC
      - strip, then collapse internal whitespace to one space
      - preserve case
      - UTF-8
    """
    collapsed = " ".join(text.strip().split())
    return unicodedata.normalize("NFC", collapsed).encode("utf-8")
```

The function is pure and deterministic. The constant
`NORMALISATION_VERSION` is written into `§2.normalization.version` of the
dedup report so the fingerprint table's schema is explicit about which
canonical form it belongs to.

### 2.4 `code/fingerprint.py`

```python
# code/fingerprint.py
# xxh128 fingerprint over the canonical form (per Chapter 02).
from __future__ import annotations

import hashlib
import struct
from typing import Iterable

import xxhash

from canonicalise import canonicalise


def fingerprint(text: str) -> int:
    """128-bit document fingerprint used as the primary dedup key."""
    return xxhash.xxh3_128_intdigest(canonicalise(text))


def audit_sha(sorted_fingerprints: Iterable[int]) -> str:
    """SHA-256 of the big-endian-packed sorted fingerprint stream.

    Slice-level audit hash. Matches the Chapter 08 §1
    `post_dedup_slice_sha256` field. One hash per slice, NOT per document.
    """
    h = hashlib.sha256()
    for fp in sorted_fingerprints:
        h.update(struct.pack(">QQ", fp >> 64, fp & ((1 << 64) - 1)))
    return h.hexdigest()
```

Two design notes worth defending at review:

- **Sorted fingerprints.** The audit hash must be order-insensitive w.r.t.
  the row order of the input slice; otherwise re-running the pipeline on
  a re-sharded slice produces a different hash for the same logical
  content. Sorting before hashing makes the audit hash a function of the
  **set** of fingerprints, which is the right semantic.
- **Big-endian pack.** A fixed, documented byte layout is required for
  the audit hash to be reproducible across machines. Do not rely on
  Python's default `int.to_bytes` without specifying endianness.

### 2.5 `code/dedup_set.py` — Implementation A (oracle)

```python
# code/dedup_set.py
# Implementation A: Python `set` as the "already-seen" structure.
# Acts as the oracle for Implementation B's false-positive measurement.
from __future__ import annotations

import argparse
import json
import resource
import time
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq

from fingerprint import fingerprint


def dedup_with_set(input_parquet: Path, output_dir: Path) -> dict:
    output_dir.mkdir(parents=True, exist_ok=True)
    seen: set[int] = set()
    rows: list[tuple[str, int, int]] = []  # (doc_id, fingerprint, text_len)

    t0 = time.perf_counter()
    reader = pq.ParquetFile(input_parquet)
    docs_seen = 0

    for batch in reader.iter_batches(batch_size=50_000, columns=["doc_id", "text"]):
        for doc_id, text in zip(
            batch.column("doc_id").to_pylist(),
            batch.column("text").to_pylist(),
        ):
            fp = fingerprint(text)
            rows.append((doc_id, fp, len(text)))
            seen.add(fp)
            docs_seen += 1

    elapsed = time.perf_counter() - t0
    peak_rss_kb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

    out_table = pa.table(
        {
            "doc_id": [r[0] for r in rows],
            "fingerprint_xxh128_hi": [r[1] >> 64 for r in rows],
            "fingerprint_xxh128_lo": [r[1] & ((1 << 64) - 1) for r in rows],
            "text_len": [r[2] for r in rows],
        }
    )
    pq.write_table(out_table, output_dir / "fingerprints.parquet")

    return {
        "impl": "set",
        "docs_seen": docs_seen,
        "unique_fingerprints": len(seen),
        "elapsed_sec": elapsed,
        "docs_per_sec": docs_seen / elapsed,
        "peak_rss_mb": peak_rss_kb / 1024,
    }


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--input", required=True, type=Path)
    ap.add_argument("--output-dir", required=True, type=Path)
    ap.add_argument("--metrics-out", required=True, type=Path)
    args = ap.parse_args()
    metrics = dedup_with_set(args.input, args.output_dir)
    args.metrics_out.write_text(json.dumps(metrics, indent=2, sort_keys=True))
    print(json.dumps(metrics, indent=2, sort_keys=True))
```

Notes:

- The set is the **oracle** against which the Bloom filter's false
  positives are measured. The cost of reading `seen` is negligible
  compared to `fingerprint()`.
- `peak_rss_mb` is intentionally measured from the whole process, not
  just the set, so comparisons with Implementation B are apples-to-apples
  (both include parquet reader buffers).
- Parquet is written in a fixed column layout that Implementation B must
  match so `cluster_resolve.py` reads either output without special
  cases.

### 2.6 `code/dedup_bloom.py` — Implementation B

```python
# code/dedup_bloom.py
# Implementation B: Bloom filter sized from (N, target_fp_rate).
# Produces a stream of "already-seen" verdicts whose false-positive rate
# is bounded by the Chapter 02 `ε = 10^-4` target.
from __future__ import annotations

import argparse
import json
import resource
import time
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
from pybloom_live import BloomFilter

from fingerprint import fingerprint


def dedup_with_bloom(
    input_parquet: Path,
    output_dir: Path,
    oracle_fingerprints: set[int],
    expected_n: int,
    fp_rate: float = 1e-4,
) -> dict:
    output_dir.mkdir(parents=True, exist_ok=True)
    bf = BloomFilter(capacity=expected_n, error_rate=fp_rate)

    rows: list[tuple[str, int, int, bool]] = []  # + bloom_said_seen
    false_positives = 0
    docs_seen = 0
    t0 = time.perf_counter()

    reader = pq.ParquetFile(input_parquet)
    for batch in reader.iter_batches(batch_size=50_000, columns=["doc_id", "text"]):
        for doc_id, text in zip(
            batch.column("doc_id").to_pylist(),
            batch.column("text").to_pylist(),
        ):
            fp = fingerprint(text)
            bloom_said_seen = fp in bf
            # Bloom claims "seen" but oracle has not seen it yet — that is a
            # false positive. We compute oracle membership BEFORE inserting
            # fp into the oracle in a parallel pass; here the oracle is
            # pre-computed and passed in.
            if bloom_said_seen and fp not in oracle_fingerprints_seen_so_far(
                rows, fp
            ):
                false_positives += 1
            bf.add(fp)
            rows.append((doc_id, fp, len(text), bloom_said_seen))
            docs_seen += 1

    elapsed = time.perf_counter() - t0
    peak_rss_kb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

    out_table = pa.table(
        {
            "doc_id": [r[0] for r in rows],
            "fingerprint_xxh128_hi": [r[1] >> 64 for r in rows],
            "fingerprint_xxh128_lo": [r[1] & ((1 << 64) - 1) for r in rows],
            "text_len": [r[2] for r in rows],
            "bloom_said_seen": [r[3] for r in rows],
        }
    )
    pq.write_table(out_table, output_dir / "fingerprints_bloom.parquet")

    return {
        "impl": "bloom",
        "docs_seen": docs_seen,
        "elapsed_sec": elapsed,
        "docs_per_sec": docs_seen / elapsed,
        "peak_rss_mb": peak_rss_kb / 1024,
        "target_fp_rate": fp_rate,
        "observed_false_positives": false_positives,
        "observed_fp_rate": false_positives / max(docs_seen, 1),
    }


def oracle_fingerprints_seen_so_far(rows: list, fp: int) -> set[int]:
    """Running oracle set from the rows seen so far.

    For a production implementation, maintain the running set incrementally
    instead of recomputing. This function is a correctness illustration.
    """
    return {r[1] for r in rows}
```

Design notes worth defending:

- **Sizing the filter.** The constructor needs `capacity=N` and
  `error_rate=ε`. `pybloom_live` sizes the bit array and the number of
  hash functions from those two. Per Chapter 02, 19 bits per element at
  `ε = 10^-4` is the rule of thumb; the library handles that.
- **Oracle semantics.** A "false positive" is: Bloom says "seen", but the
  oracle set does not yet contain this fingerprint. A `(seen already-by-
  oracle, seen already-by-Bloom)` pair is a true positive, not a false
  positive. The naive `oracle_fingerprints_seen_so_far` is explicit for
  clarity; a production implementation maintains a running set
  incrementally.
- **Writing `bloom_said_seen` as a column** is intentional — the
  `cluster_resolve.py` step can diff against the oracle fingerprint table
  and reconstruct the false-positive set without a second pass.

### 2.7 `code/cluster_resolve.py` — longest-survivor rule

```python
# code/cluster_resolve.py
# Resolve fingerprint-collision clusters with the longest-survivor rule.
from __future__ import annotations

import argparse
import hashlib
import struct
from pathlib import Path

import pyarrow as pa
import pyarrow.compute as pc
import pyarrow.parquet as pq


def resolve(fingerprints_parquet: Path, slice_parquet: Path, output_dir: Path) -> None:
    output_dir.mkdir(parents=True, exist_ok=True)
    fps = pq.read_table(fingerprints_parquet)
    slice_tbl = pq.read_table(slice_parquet)

    # (doc_id, fingerprint, text_len) joined with the raw text / source columns
    joined = slice_tbl.join(fps, keys=["doc_id"])

    # Rank by text_len within each fingerprint cluster; keep rank 1.
    df = joined.to_pandas()
    df["cluster_rank"] = (
        df.groupby(["fingerprint_xxh128_hi", "fingerprint_xxh128_lo"])["text_len"]
        .rank(method="first", ascending=False)
        .astype(int)
    )

    survivors = df[df["cluster_rank"] == 1].drop(columns=["cluster_rank"])
    removed = df[df["cluster_rank"] > 1].copy()

    # removed_docs schema: doc_id, cluster_representative_doc_id, cluster_size
    reps = (
        survivors.set_index(["fingerprint_xxh128_hi", "fingerprint_xxh128_lo"])["doc_id"]
        .to_dict()
    )
    removed["cluster_representative_doc_id"] = removed.apply(
        lambda r: reps[(r["fingerprint_xxh128_hi"], r["fingerprint_xxh128_lo"])],
        axis=1,
    )
    cluster_sizes = (
        df.groupby(["fingerprint_xxh128_hi", "fingerprint_xxh128_lo"])["doc_id"]
        .count()
        .to_dict()
    )
    removed["cluster_size"] = removed.apply(
        lambda r: cluster_sizes[(r["fingerprint_xxh128_hi"], r["fingerprint_xxh128_lo"])],
        axis=1,
    )

    pa.Table.from_pandas(
        removed[["doc_id", "cluster_representative_doc_id", "cluster_size"]]
    ).to_parquet(output_dir / "removed_docs.parquet")
    pa.Table.from_pandas(survivors).to_parquet(output_dir / "dedup_slice.parquet")

    # Slice-level SHA-256 (sorted fingerprint stream).
    fps_sorted = sorted(
        zip(
            survivors["fingerprint_xxh128_hi"].tolist(),
            survivors["fingerprint_xxh128_lo"].tolist(),
        )
    )
    h = hashlib.sha256()
    for hi, lo in fps_sorted:
        h.update(struct.pack(">QQ", hi, lo))
    (output_dir / "dedup_slice.sha256").write_text(h.hexdigest() + "\n")
```

**Why longest-survivor?** Chapter 05 discusses survivor rules at length;
the exact-dedup default is "keep the longest observed text". Reasoning:
in a fingerprint collision cluster, the longest document is the one
least likely to be a truncated or stub version of the underlying content
and is the safest representative for downstream tokenisation.

### 2.8 `code/emit_report.py` — write `dedup_report.yaml` §1–§3

```python
# code/emit_report.py
# Emit §1 (identity), §2 (fingerprint policy), and §3 exact_dedup fields.
# Leaves near-dup / paragraph / sentence fields as explicit null — those
# are Exercise 02 and Exercise 03's job.
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

from canonicalise import NORMALISATION_VERSION


def sha256_file(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--slice-id", required=True)
    ap.add_argument("--slice-version", required=True)
    ap.add_argument("--operator", required=True)
    ap.add_argument("--pre-dedup-slice", required=True, type=Path)
    ap.add_argument("--post-dedup-slice", required=True, type=Path)
    ap.add_argument("--removed-docs", required=True, type=Path)
    ap.add_argument("--fingerprints", required=True, type=Path)
    ap.add_argument("--config", required=True, type=Path)
    ap.add_argument("--run-start-utc", required=True)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    pre = pq.read_table(args.pre_dedup_slice)
    post = pq.read_table(args.post_dedup_slice)
    removed = pq.read_table(args.removed_docs)

    pre_n = pre.num_rows
    post_n = post.num_rows
    removed_n = removed.num_rows
    assert pre_n == post_n + removed_n, (
        f"row conservation violated: pre={pre_n} post={post_n} removed={removed_n}"
    )

    # Per-source removal rate (requires a 'source' column on the slice).
    per_source: dict[str, float] = {}
    if "source" in pre.column_names:
        pre_by_source = pre.group_by("source").aggregate([("doc_id", "count")])
        post_by_source = post.group_by("source").aggregate([("doc_id", "count")])
        pre_counts = dict(zip(pre_by_source["source"].to_pylist(),
                              pre_by_source["doc_id_count"].to_pylist()))
        post_counts = dict(zip(post_by_source["source"].to_pylist(),
                               post_by_source["doc_id_count"].to_pylist()))
        for src, n in pre_counts.items():
            per_source[src] = round(1 - post_counts.get(src, 0) / n, 6)

    git_sha = subprocess.check_output(["git", "rev-parse", "HEAD"]).decode().strip()

    report = {
        "slice_id": args.slice_id,
        "slice_version": args.slice_version,
        "toolchain": {
            "name": "datasketch+hand-rolled",
            "version": "exercise-01",
            "git_sha": git_sha,
            "config_path": str(args.config),
            "config_sha256": sha256_file(args.config),
        },
        "operator": args.operator,
        "run_start_utc": args.run_start_utc,
        "run_end_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "compute_shape": {
            "host": socket.gethostname(),
            "cpu_cores": None,  # fill from /proc/cpuinfo or `os.cpu_count()`
            "memory_gb": None,
            "gpus": 0,
        },
        "pre_dedup_slice_sha256": sha256_file(args.pre_dedup_slice),
        "post_dedup_slice_sha256": sha256_file(args.post_dedup_slice),
        "normalization": {
            "unicode_form": "NFC",
            "whitespace": "collapse_and_strip",
            "case": "preserve",
            "encoding": "utf-8",
            "version": NORMALISATION_VERSION,
        },
        "hash": {
            "fingerprint_algo": "xxh3_128",
            "audit_algo": "sha256",
        },
        "granularities": [
            {
                "level": "document",
                "shingle_tokenization": None,
                "shingle_width": None,
                "minhash_num_perm": None,
                "lsh_bands": None,
                "lsh_rows_per_band": None,
                "jaccard_threshold": None,
                "survivor_rule": "longest",
            }
        ],
        "exact_dedup": {
            "pre_doc_count": pre_n,
            "post_doc_count": post_n,
            "removal_rate": round(removed_n / pre_n, 6) if pre_n else 0.0,
            "per_source_removal_rate": per_source,
        },
        "near_dedup_document": None,
        "near_dedup_paragraph": None,
    }

    args.output.write_text(yaml.safe_dump(report, sort_keys=False))
    print(json.dumps({"report_path": str(args.output), "report_sha256":
                      sha256_file(args.output)}, indent=2))


if __name__ == "__main__":
    main()
```

Two determinism safeguards worth defending at review:

- `yaml.safe_dump(..., sort_keys=False)` is paired with an intentionally
  ordered Python dict so byte-identical output requires no
  post-processing. Re-running the script on the same inputs must produce
  a byte-identical file; the test in §3 verifies this.
- Any field the exercise does not populate is written as explicit `null`,
  not omitted. Downstream readers (`mod-109`, `mod-112`) can tell the
  difference between "this field was considered and left empty" and
  "this field did not exist at author time".

### 2.9 `comparison.md` — set vs Bloom

The learner's `comparison.md` should contain, at minimum:

```markdown
# Set vs Bloom — Implementation A vs B

| metric                      | set (A) | bloom (B) |
|-----------------------------|---------|-----------|
| docs/sec                    | …       | …         |
| peak RSS (MB)               | …       | …         |
| false positives observed    | 0 (oracle) | <n> |
| target FP rate              | 0       | 1e-4      |
| observed FP rate            | 0       | <obs>     |

## Verifying Chapter 02 §"Scaling guide summary"

Chapter 02 claims: Python `set` ≤ 10^7 fits on a workstation
(~60-80 bytes per 128-bit int × 10^7 ≈ 600-800 MB). At N = <your N> my
set used <measured> MB, which is within a factor of <ratio> of the
prediction. If the ratio is far off — explain.

Bloom at ε = 10^-4 predicts ~19 bits/element; for N = <your N> that is
<predicted> MB. Measured: <measured> MB.
```

The intent is not to beat the baseline — the two implementations serve
different scaling regimes. The comparison shows the learner has operated
both, measured both, and can recommend one over the other for a given
deployment.

### 2.10 `Makefile`

```makefile
.PHONY: all build dedup report clean

all: dedup report

dedup:
	python code/dedup_set.py --input data/slice.parquet \
		--output-dir results/ --metrics-out results/metrics_set.json
	python code/dedup_bloom.py --input data/slice.parquet \
		--output-dir results/ --metrics-out results/metrics_bloom.json
	python code/cluster_resolve.py \
		results/fingerprints.parquet data/slice.parquet results/

report:
	python code/emit_report.py \
		--slice-id slice-cc-en-small-v1 --slice-version 1.0.0 \
		--operator $$(git config user.email) \
		--pre-dedup-slice data/slice.parquet \
		--post-dedup-slice results/dedup_slice.parquet \
		--removed-docs results/removed_docs.parquet \
		--fingerprints results/fingerprints.parquet \
		--config policy_note.md \
		--run-start-utc $$(date -u +%Y-%m-%dT%H:%M:%SZ) \
		--output results/dedup_report.yaml

clean:
	rm -rf results/fingerprints*.parquet results/removed_docs.parquet \
	       results/dedup_slice.parquet results/dedup_slice.sha256 \
	       results/dedup_report.yaml
```

## 3. Validation steps

1. **`policy_note.md` pre-dates the code.** `git log --follow policy_note.md
   | tail -1` returns a commit older than any `code/*.py` commit on this
   exercise. If not, the policy was retro-fitted and the review fails.
2. **Row conservation.** `pre_doc_count == post_doc_count + removed_docs
   row count`. The assertion in `emit_report.py` enforces this; a
   mismatch indicates a bug in `cluster_resolve.py`.
3. **Report determinism.** `make clean && make report && sha256sum
   results/dedup_report.yaml` twice; the two hashes match. If they don't,
   `emit_report.py` is leaking non-deterministic data (a timestamp other
   than `run_end_utc`, a dict-order change, etc.).
4. **Sidecar hashes match.** `sha256sum results/fingerprints.parquet`
   matches `results/fingerprints.parquet.sha256`.
5. **Slice audit hash matches the sorted-fingerprint stream.** Compute
   `audit_sha(sorted_fingerprints)` from `dedup_slice.parquet` and match
   against `results/dedup_slice.sha256`.
6. **Per-source removal rates present.** `yq
   '.exact_dedup.per_source_removal_rate | keys | length'
   results/dedup_report.yaml` ≥ 5.
7. **Bloom FP rate is in the right neighbourhood.** `observed_fp_rate`
   in `metrics_bloom.json` is within one order of magnitude of the
   `target_fp_rate = 1e-4`. If observed is >> 10× target, the capacity
   was under-sized relative to actual `N`.
8. **Normalisation audit.** Pick a known near-duplicate pair (same text
   with trailing whitespace) that your policy should collapse; run the
   `canonicalise()` function on both inputs; byte-compare. They must be
   equal under NFC-and-collapse-whitespace.
9. **No invented metrics.** If a cell in the `per_source_removal_rate`
   table is `null`, the source had zero pre-dedup documents, not a
   missing computation. Document this convention in the README.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible
with evidence). Pass threshold: ≥ 20/28 with zero zeros on critical rows.

| #  | Dimension                                                                                                  | Critical? |
|----|------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `policy_note.md` exists and is older than any `code/*.py` commit                                           | Yes       |
| 2  | `canonicalise()` is a pure function pinned by `NORMALISATION_VERSION`                                      | Yes       |
| 3  | `fingerprint()` uses `xxh3_128` and operates on the canonical bytes, not raw text                          | Yes       |
| 4  | Slice-level `audit_sha` is order-insensitive (sorts fingerprints before hashing)                           | Yes       |
| 5  | Both Implementation A and Implementation B measure peak RSS and docs/sec                                   | Yes       |
| 6  | Bloom filter is sized from `(N, ε = 10^-4)`, not from a magic constant                                     | Yes       |
| 7  | `cluster_resolve.py` implements the longest-survivor rule and writes `removed_docs.parquet`                | Yes       |
| 8  | `dedup_report.yaml` populates §1, §2, and §3 exact-dedup fields; near-dup / paragraph fields are null       | Yes       |
| 9  | `per_source_removal_rate` covers ≥ 5 distinct sources                                                      | Yes       |
| 10 | Report determinism check (two runs → byte-identical YAML) passes                                           | Yes       |
| 11 | Sidecar `.sha256` files match the computed artifact hashes                                                 | Yes       |
| 12 | Governance anchor (OWASP ML Top 10, MITRE ATLAS, NIST AI RMF) cited in README                              | No        |
| 13 | `comparison.md` reports both throughput and memory and interrogates Chapter 02's rule-of-thumb              | No        |
| 14 | Any unverifiable claim is flagged with `<!-- followup-research: ... -->`                                   | Yes       |

## 5. Common mistakes

- **Fingerprinting the raw text.** A single trailing newline becomes a
  different fingerprint. Canonicalise first, then hash, every time.
- **Using SHA-256 per-document.** SHA-256 is correct for the slice-level
  audit hash; using it per document costs 10–30× vs xxh128 for the same
  collision safety at document scale. Chapter 02 §"Which hash" is
  explicit: xxh128 for fingerprint, SHA-256 for audit.
- **Lowercasing by default.** Casefolding collapses `US` and `us`, which
  is almost always a correctness bug for English pretraining. Only apply
  casefolding in domain-specific contexts, and declare it in
  `policy_note.md`.
- **Bloom filter sized to a round number.** `BloomFilter(capacity=1_000_000,
  error_rate=0.01)` is not Chapter 02's recipe. Size from the actual
  expected `N` and the stated target `ε = 10^-4`; the library computes
  bits and hash functions from those two.
- **"Just dropping duplicates" instead of writing `removed_docs.parquet`.**
  A silent drop is a lineage gap; `mod-109` cannot reconstruct what the
  pipeline removed. Keep the removal manifest with the cluster
  representative and cluster size.
- **Treating the audit hash as a function of row order.** If the slice is
  re-sharded, the hash must not change. Sort before hashing.
- **Reporting only aggregate `removal_rate`.** Chapter 08 §3 mandates
  `per_source_removal_rate`. An aggregate hides credibility-weighted
  shifts and adversarial-perturbation signals.
- **Omitting `null` fields.** `dedup_report.yaml` must be a complete,
  stable schema even when a section is empty — downstream readers
  interpret missing keys differently from explicit nulls.
- **Baking `run_end_utc` into deterministic re-run checks.** The
  determinism check in §3 should exclude `run_end_utc` (and
  `run_start_utc`) or run both back-to-back with the same wall clock;
  otherwise the determinism test is trivially broken.

## 6. References

- **xxHash** — xxhash reference implementation and algorithm notes:
  <https://github.com/Cyan4973/xxHash>. xxh3-128 is the fingerprint
  convention used by every major pretraining pipeline cited in Chapter
  02.
- **Python `hashlib` SHA-256** — <https://docs.python.org/3/library/hashlib.html>.
  Use SHA-256 for the slice-level audit hash only.
- **pybloom_live** — <https://github.com/joseph-fox/python-bloomfilter>.
  Sizes the bit array and hash-function count from `(capacity, error_rate)`.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02:2023 (Data Poisoning Attack) is the governance anchor for why
  exact dedup is a compensating control, not optional hygiene.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The current matrix lists
  AML.T0019 (Publish Poisoned Datasets) and AML.T0020 (Poison Training
  Data) under adversary-facing techniques your dedup pipeline weakens.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE 2.7
  (security and resilience are measured) and MEASURE 2.10 (trustworthy
  characteristics are measured) are the control anchors the dedup report
  satisfies.
- **Local anchors:** Chapter 02 of this module (this exercise's primary
  content); Chapter 08 (the dedup report schema); Exercise 02 (extends
  the report to near-dup); `mod-103` Exercise 01 (canonical input
  slice); `mod-109` (Sigstore signing of the slice-level audit hash).
