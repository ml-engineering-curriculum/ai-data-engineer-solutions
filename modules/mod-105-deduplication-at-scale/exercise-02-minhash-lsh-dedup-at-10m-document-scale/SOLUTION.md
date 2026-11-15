# Exercise 02 — MinHash + LSH Deduplication at 10M-Document Scale (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner stands up a
`datasketch`-backed MinHash + LSH pipeline on a document slice at the
10M scale, calibrates the Jaccard threshold against a labelled sample,
and extends the `dedup_report.yaml` from Exercise 01 with §3's
`near_dedup_document` block and §4's `memorisation_risk` block.

The key deliverables are:

- A parameterised `(K=128, b, r)` LSH pipeline that **passes the `(b, r)`
  tuple explicitly** to `MinHashLSH`, not via threshold-only
  initialisation. This forces the learner to understand the S-curve
  rather than trust `datasketch`'s automatic sizing.
- A labelled calibration sample of ≥ 500 candidate pairs with
  ground-truth Jaccard and human (or heuristic) labels.
- A union-find pass that reduces verified pairs to clusters, a
  survivor-rule resolver (longest-document per cluster), and a removal
  manifest extending Exercise 01's.
- A pre-dedup / post-dedup duplicate-count distribution (p50 / p90 / p99
  / p99.9) that is the input to the Chapter 08 §4 memorisation-risk
  block.
- An extended `dedup_report.yaml` with §3 `near_dedup_document` and §4
  `memorisation_risk` populated.

Governance framing: the memorisation-risk delta is the bridge from
`mod-105` to `mod-110` (privacy risk). Kandpal 2022 reports that the
rate at which language models regenerate training sequences is
**superlinear** in a sequence's count in the training set — collapsing
the duplicate-count tail is directly a **training-data extraction
mitigation** (OWASP LLM Top 10 **LLM02: Sensitive Information Disclosure**
in the 2025 edition; **LLM06** in the 2023 edition — cite the edition in
your report). MITRE ATLAS **AML.T0048** ("Reconstruct Training Data")
and **AML.T0024** ("Exfiltrate via ML Inference API") frame the
adversary side.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── requirements.lock
├── Makefile
├── s_curve.md                           # calibration write-up
├── code/
│   ├── shingle.py
│   ├── build_signatures.py              # MinHash(num_perm=128), xxhash hashfunc
│   ├── build_lsh.py                     # MinHashLSH(params=(BANDS, ROWS))
│   ├── verify_pairs.py                  # MinHash.jaccard() verification
│   ├── union_find.py
│   ├── cluster_resolve.py               # longest-doc survivor
│   ├── plot_s_curve.py
│   └── emit_report.py                   # extends Exercise 01's report
├── data/
│   └── calibration_sample.jsonl         # 500 pairs, GT Jaccard + labels
└── results/
    ├── signatures.parquet
    ├── signatures.parquet.sha256
    ├── candidate_pairs.parquet
    ├── verified_pairs.parquet
    ├── clusters.parquet
    ├── clusters.parquet.sha256
    ├── removed_docs.parquet             # extends Exercise 01
    ├── dedup_report.yaml                # §3 near-dup + §4 populated
    └── plots/
        ├── s_curve.png
        ├── calibration.png
        └── cluster_size_hist.png
```

### 2.2 Threshold calibration — do this first

The calibration sample has the shape:

```json
{"doc_a_id": "...", "doc_b_id": "...", "jaccard_true": 0.83,
 "shingle_width": 5, "label": "duplicate", "labeller": "jf", "note": "..."}
```

Five hundred rows minimum, biased toward true near-duplicates via a
coarse prefix-match pre-filter. Two plots produced:

- `plots/calibration.png`: duplicate-rate vs ground-truth Jaccard; the
  "step" is your `J*`.
- `plots/s_curve.png`: three candidate `(b, r)` curves at `K = 128`
  overlaid. The chosen configuration's soft threshold `s*` must sit
  below `J*` by a cushion of 0.05–0.10.

A worked choice for a prose web slice with human labellers:

- Observed empirical step at `J ≈ 0.82`.
- Candidate configurations at `K = 128`:
  - `(b=16, r=8) → s* ≈ 0.68` — safe cushion.
  - `(b=20, r=6) → s* ≈ 0.57` — too wide; many false positives.
  - `(b=10, r=13) → s* ≈ 0.80` — too close; recall at risk.
- Chosen: `(b=16, r=8)` with `J* = 0.80`. Cushion ≈ 0.12.

`s_curve.md` must contain the three S-curves, the calibration plot, and
one paragraph defending the chosen tuple.

### 2.3 `code/shingle.py`

```python
# code/shingle.py
# Word-level 5-shingles over the Chapter 02 canonical form.
from __future__ import annotations

import unicodedata
from collections.abc import Iterator

SHINGLE_WIDTH = 5


def canonical_tokens(text: str) -> list[str]:
    collapsed = " ".join(text.strip().split())
    return unicodedata.normalize("NFC", collapsed).split()


def word_shingles(text: str, k: int = SHINGLE_WIDTH) -> Iterator[bytes]:
    toks = canonical_tokens(text)
    if len(toks) < k:
        yield " ".join(toks).encode("utf-8")
        return
    for i in range(len(toks) - k + 1):
        yield " ".join(toks[i : i + k]).encode("utf-8")
```

Shingles are yielded as `bytes`, matching `MinHash.update(bytes)`.
Keeping the same canonical form as Exercise 01 means an exact duplicate
and a near duplicate are measured against the same primitive — the two
reports are comparable row-by-row.

### 2.4 `code/build_signatures.py`

```python
# code/build_signatures.py
# One MinHash signature per document. 128 permutations. xxhash as hashfunc.
# Deterministic: a fixed seed + identical input → bit-identical signatures.
from __future__ import annotations

import argparse
from pathlib import Path

import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
import xxhash
from datasketch import MinHash

from shingle import word_shingles

NUM_PERM = 128


def _xx64(data: bytes) -> int:
    return xxhash.xxh64_intdigest(data)


def build_signature(text: str, seed: int) -> np.ndarray:
    m = MinHash(num_perm=NUM_PERM, seed=seed, hashfunc=_xx64)
    for shingle in word_shingles(text):
        m.update(shingle)
    return m.hashvalues.astype(np.uint64)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--input", required=True, type=Path)
    ap.add_argument("--output", required=True, type=Path)
    ap.add_argument("--seed", type=int, default=0xC0FFEE)
    args = ap.parse_args()

    reader = pq.ParquetFile(args.input)
    doc_ids: list[str] = []
    sigs: list[np.ndarray] = []
    for batch in reader.iter_batches(batch_size=10_000, columns=["doc_id", "text"]):
        for doc_id, text in zip(
            batch.column("doc_id").to_pylist(),
            batch.column("text").to_pylist(),
        ):
            doc_ids.append(doc_id)
            sigs.append(build_signature(text, args.seed))

    sig_bytes = np.stack(sigs).tobytes()  # fixed-width 128 * 8 bytes per row
    out = pa.table(
        {
            "doc_id": doc_ids,
            "signature": pa.array(
                [sig_bytes[i * NUM_PERM * 8 : (i + 1) * NUM_PERM * 8] for i in range(len(doc_ids))],
                type=pa.binary(NUM_PERM * 8),
            ),
        }
    )
    pq.write_table(out, args.output)


if __name__ == "__main__":
    main()
```

Determinism notes:

- `datasketch.MinHash(seed=…)` makes the permutation tables
  reproducible. **Pass the seed explicitly.** Omitting the seed gives
  you a run-local random permutation table and the signatures will not
  match on re-run.
- `hashfunc=xxhash.xxh64_intdigest` replaces `datasketch`'s default
  `sha1`, which is slower and non-critical for the fingerprinting role.
- Fixed-width binary columns (`pa.binary(NUM_PERM * 8)`) keep the
  Parquet file compact and random-accessible.

### 2.5 `code/build_lsh.py`

```python
# code/build_lsh.py
# MinHashLSH with (BANDS, ROWS) passed EXPLICITLY.
# The exercise forbids threshold-only initialisation.
from __future__ import annotations

import argparse
from pathlib import Path

import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
from datasketch import MinHash, MinHashLSH

NUM_PERM = 128
BANDS = 16
ROWS = 8
assert BANDS * ROWS == NUM_PERM, "K = b * r must hold"


def materialise_minhash(sig_bytes: bytes) -> MinHash:
    m = MinHash(num_perm=NUM_PERM)
    m.hashvalues = np.frombuffer(sig_bytes, dtype=np.uint64).copy()
    return m


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--signatures", required=True, type=Path)
    ap.add_argument("--jaccard-threshold", type=float, default=0.80)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    lsh = MinHashLSH(
        num_perm=NUM_PERM,
        params=(BANDS, ROWS),            # explicit — NOT threshold=0.8
    )

    sigs = pq.read_table(args.signatures)
    minhashes: dict[str, MinHash] = {}
    with lsh.insertion_session() as session:
        for doc_id, sig_bytes in zip(
            sigs.column("doc_id").to_pylist(),
            sigs.column("signature").to_pylist(),
        ):
            m = materialise_minhash(sig_bytes)
            session.insert(doc_id, m)
            minhashes[doc_id] = m

    pairs: list[tuple[str, str, float]] = []
    for doc_id, m in minhashes.items():
        for candidate in lsh.query(m):
            if candidate <= doc_id:
                continue
            est = m.jaccard(minhashes[candidate])
            if est >= args.jaccard_threshold:
                pairs.append((doc_id, candidate, est))

    pq.write_table(
        pa.table(
            {
                "doc_a": [p[0] for p in pairs],
                "doc_b": [p[1] for p in pairs],
                "jaccard_est": [p[2] for p in pairs],
            }
        ),
        args.output,
    )


if __name__ == "__main__":
    main()
```

The `params=(BANDS, ROWS)` argument is the acceptance criterion. Reading
the `datasketch` source will show that threshold-only initialisation
calls `optimal_param` to pick `(b, r)` automatically; for a learning
exercise that elides the whole point of the S-curve. Pin it explicitly.

### 2.6 `code/union_find.py`

```python
# code/union_find.py
# Rank + path-compression union-find over verified pairs.
from __future__ import annotations

import argparse
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


class UnionFind:
    def __init__(self) -> None:
        self.parent: dict[str, str] = {}
        self.rank: dict[str, int] = {}

    def find(self, x: str) -> str:
        p = self.parent.setdefault(x, x)
        while p != x:
            self.parent[x] = self.parent[p]  # path compression
            x, p = p, self.parent[p]
        return x

    def union(self, a: str, b: str) -> None:
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return
        ra_rank = self.rank.get(ra, 0)
        rb_rank = self.rank.get(rb, 0)
        if ra_rank < rb_rank:
            ra, rb = rb, ra
        self.parent[rb] = ra
        if ra_rank == rb_rank:
            self.rank[ra] = ra_rank + 1


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--verified-pairs", required=True, type=Path)
    ap.add_argument("--all-doc-ids", required=True, type=Path)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    uf = UnionFind()
    pairs = pq.read_table(args.verified_pairs).to_pandas()
    for a, b in zip(pairs["doc_a"], pairs["doc_b"]):
        uf.union(a, b)

    all_ids = pq.read_table(args.all_doc_ids).column("doc_id").to_pylist()
    clusters: dict[str, list[str]] = {}
    for d in all_ids:
        root = uf.find(d) if d in uf.parent else d
        clusters.setdefault(root, []).append(d)

    rows = [
        (root, members, len(members))
        for root, members in sorted(clusters.items())
        if len(members) > 1
    ]
    pq.write_table(
        pa.table(
            {
                "cluster_id": [r[0] for r in rows],
                "members": [r[1] for r in rows],
                "cluster_size": [r[2] for r in rows],
            }
        ),
        args.output,
    )


if __name__ == "__main__":
    main()
```

Two design calls to defend:

- **Singletons are not written.** `cluster_size > 1` is the filter. A
  singleton is semantically "no duplicate found for this document"; it
  would bloat the clusters file by ~95% without carrying useful
  information.
- **Clusters are sorted by root doc_id.** Determinism — re-runs produce
  the same byte stream. The Chapter 08 schema does not mandate an order;
  we pick one.

### 2.7 `code/cluster_resolve.py`

Uses the longest-survivor rule from Exercise 01 at the cluster level.
Extend `removed_docs.parquet` with:

```python
# removed_docs schema now carries both exact-dedup and near-dedup rows.
# dedup_stage ∈ {"exact", "near_doc"} distinguishes the two.
schema = pa.schema(
    [
        ("doc_id", pa.string()),
        ("cluster_representative_doc_id", pa.string()),
        ("cluster_size", pa.int64()),
        ("dedup_stage", pa.string()),
        ("jaccard_to_representative", pa.float64()),  # null for exact
    ]
)
```

### 2.8 Duplicate-count distribution — the §4 input

```python
# code/duplicate_count_distribution.py
from __future__ import annotations

import pyarrow.parquet as pq


def duplicate_count_distribution(clusters_parquet, all_doc_ids_parquet):
    clusters = pq.read_table(clusters_parquet).to_pandas()
    all_ids = pq.read_table(all_doc_ids_parquet).to_pandas()

    # pre-dedup duplicate count = cluster size each doc belongs to
    doc_to_cluster = {}
    for _, row in clusters.iterrows():
        for member in row["members"]:
            doc_to_cluster[member] = row["cluster_size"]
    pre = [doc_to_cluster.get(d, 1) for d in all_ids["doc_id"]]
    # post-dedup duplicate count = 1 for every surviving doc
    post = [1] * len(pre)

    import numpy as np
    return {
        "pre": {
            "p50": int(np.percentile(pre, 50)),
            "p90": int(np.percentile(pre, 90)),
            "p99": int(np.percentile(pre, 99)),
            "p99_9": int(np.percentile(pre, 99.9)),
        },
        "post": {"p50": 1, "p90": 1, "p99": 1, "p99_9": 1},
    }
```

Post is trivially all 1s by construction — survivor-rule removal leaves
one representative per cluster. The pre-distribution is the interesting
half; its long-tail shape is the memorisation-risk anchor.

### 2.9 `emit_report.py` — extend Exercise 01's YAML

```python
# code/emit_report.py
# Reads Exercise 01's dedup_report.yaml, adds §3 near_dedup_document and
# §4 memorisation_risk, writes a new report.
from __future__ import annotations

import argparse
import hashlib
from collections import Counter
from pathlib import Path

import pyarrow.parquet as pq
import yaml


def sha256_file(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--exercise-01-report", required=True, type=Path)
    ap.add_argument("--clusters", required=True, type=Path)
    ap.add_argument("--removed-docs-near", required=True, type=Path)
    ap.add_argument("--pre-doc-count", required=True, type=int)
    ap.add_argument("--duplicate-count-json", required=True, type=Path)
    ap.add_argument("--jaccard-threshold", type=float, default=0.80)
    ap.add_argument("--bands", type=int, default=16)
    ap.add_argument("--rows", type=int, default=8)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    base = yaml.safe_load(args.exercise_01_report.read_text())
    clusters = pq.read_table(args.clusters).to_pandas()
    removed = pq.read_table(args.removed_docs_near).to_pandas()

    n_removed = len(removed)
    n_clusters = len(clusters)
    mean_cluster_size = float(clusters["cluster_size"].mean()) if n_clusters else 0.0
    max_cluster_size = int(clusters["cluster_size"].max()) if n_clusters else 0

    per_source: dict[str, float] = {}
    if "source" in removed.columns:
        per_source_count = Counter(removed["source"])
        pre_per_source = Counter(removed.get("pre_source_count", {}))
        for src, removed_n in per_source_count.items():
            pre_n = pre_per_source.get(src, 0) or removed_n
            per_source[src] = round(removed_n / pre_n, 6) if pre_n else 0.0

    dup_dist = yaml.safe_load(args.duplicate_count_json.read_text())

    import json
    dup_dist = json.loads(args.duplicate_count_json.read_text())

    base.setdefault("granularities", [])
    base["granularities"].append(
        {
            "level": "document",
            "shingle_tokenization": "whitespace",
            "shingle_width": 5,
            "minhash_num_perm": 128,
            "lsh_bands": args.bands,
            "lsh_rows_per_band": args.rows,
            "jaccard_threshold": args.jaccard_threshold,
            "survivor_rule": "longest",
        }
    )

    base["near_dedup_document"] = {
        "pre_doc_count": args.pre_doc_count,
        "post_doc_count": args.pre_doc_count - n_removed,
        "removal_rate": round(n_removed / args.pre_doc_count, 6),
        "clusters_formed": n_clusters,
        "mean_cluster_size": round(mean_cluster_size, 4),
        "max_cluster_size": max_cluster_size,
        "per_source_removal_rate": per_source,
    }

    base["memorisation_risk"] = {
        "methodology": "kandpal_2022_frequency_curve",
        "pre_dedup_duplicate_count_distribution": dup_dist["pre"],
        "post_dedup_duplicate_count_distribution": dup_dist["post"],
        # Honest, not fabricated: Kandpal 2022 supports an order-of-magnitude
        # claim at the slice's modal duplicate count.
        "extraction_risk_delta_estimate": (
            "Roughly an order of magnitude reduction in the regeneration "
            "rate at this slice's modal duplicate count, consistent with "
            "Kandpal 2022; exact multiplier depends on model and task "
            "and is deferred to the mod-110 privacy-risk report."
        ),
        "method_sha": sha256_file(args.duplicate_count_json),
    }

    args.output.write_text(yaml.safe_dump(base, sort_keys=False))


if __name__ == "__main__":
    main()
```

The `extraction_risk_delta_estimate` is **deliberately qualitative**.
Kandpal 2022 reports a superlinear relationship between training
frequency and regeneration rate; it does not endorse a single "×10"
figure for a specific slice. Fabricating a precise multiplier is an
acceptance-criterion failure.

## 3. Validation steps

1. **Plot determinism.** `make clean && make build-signatures` twice;
   `sha256sum results/signatures.parquet` matches both times. If not,
   the `seed` is not pinned or the `hashfunc` is non-deterministic.
2. **Calibration sample size.** `wc -l data/calibration_sample.jsonl`
   ≥ 500. `jq -s '[.[].jaccard_true] | length'` matches. Every row has
   a `label` field.
3. **Chosen `(b, r)` justified.** In `s_curve.md`, the chosen cushion
   between `s*` and `J*` is 0.05–0.10. `s*` computed by hand from
   `(1/b)^(1/r)` matches the plotted curve.
4. **`build_lsh.py` passes `params=` explicitly.** `grep "params=" code/build_lsh.py`
   must return a line. `grep "threshold=" code/build_lsh.py` must NOT
   appear in the `MinHashLSH(...)` call.
5. **Verification pass does work.** `wc -l results/candidate_pairs.parquet`
   > `wc -l results/verified_pairs.parquet`. If they are equal, the
   verification is a no-op.
6. **Cluster size histogram matches report.** `max(cluster_size)` in the
   histogram equals `near_dedup_document.max_cluster_size` in the YAML.
7. **Per-source rows ≥ 5.** `yq
   '.near_dedup_document.per_source_removal_rate | keys | length'`
   results/dedup_report.yaml` ≥ 5.
8. **Memorisation-risk sentence does not fabricate a number.**
   `grep -E "[0-9]+x" results/dedup_report.yaml` on the
   `extraction_risk_delta_estimate` field returns at most a qualified
   phrase ("order of magnitude", "~10x"), never a precise multiplier
   without a Kandpal 2022-supported citation.
9. **Row conservation.** `pre_doc_count == post_doc_count +
   len(removed_docs_near)`.
10. **Report schema conformance.** The report validates against the
    Chapter 08 schema (fields in §1, §2, §3, §4 present and typed).

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible
with evidence). Pass threshold: ≥ 22/30 with zero zeros on critical rows.

| #  | Dimension                                                                                           | Critical? |
|----|------------------------------------------------------------------------------------------------------|-----------|
| 1  | S-curve plot shows ≥ 3 `(b, r)` configurations at `K = 128`                                          | Yes       |
| 2  | Labelled calibration sample has ≥ 500 pairs with ground-truth Jaccard + label                        | Yes       |
| 3  | Chosen cushion between `s*` and `J*` is in [0.05, 0.10]                                              | Yes       |
| 4  | `MinHashLSH` instantiated with `params=(BANDS, ROWS)`, not `threshold=`                              | Yes       |
| 5  | Signatures are deterministic under a fixed seed                                                      | Yes       |
| 6  | Verified pairs are a strict subset of candidate pairs                                                | Yes       |
| 7  | Union-find pass runs and produces cluster-sized-≥-2 rows only                                        | Yes       |
| 8  | Survivor-rule choice is documented (longest-doc default or justified alternative)                    | Yes       |
| 9  | `removed_docs.parquet` extends Exercise 01's schema with `dedup_stage` and `jaccard_to_representative` | Yes       |
| 10 | `dedup_report.yaml` §3 `near_dedup_document` has `per_source_removal_rate` with ≥ 5 sources          | Yes       |
| 11 | `dedup_report.yaml` §4 `memorisation_risk` present with p50/p90/p99/p99.9 pre and post               | Yes       |
| 12 | `extraction_risk_delta_estimate` is qualitative and cites Kandpal 2022; no fabricated multiplier     | Yes       |
| 13 | Cluster-size histogram matches `max_cluster_size` in the YAML                                        | No        |
| 14 | OWASP LLM Top 10 training-data-extraction item and MITRE ATLAS techniques cited in README            | No        |
| 15 | Any unverifiable claim flagged with `<!-- followup-research: ... -->`                                | Yes       |

## 5. Common mistakes

- **Threshold-only `MinHashLSH` initialisation.** Convenient, but then
  `datasketch`'s `optimal_param` picks `(b, r)` for you and you cannot
  defend the choice. The exercise explicitly requires `params=`.
- **No calibration sample.** Choosing `J* = 0.8` because it "feels
  right" or because SlimPajama used it is not a defence for your slice.
  A labelled sample makes the choice specific to the slice at hand.
- **Not seeding the MinHash.** `datasketch.MinHash()` picks permutation
  tables stochastically unless `seed=` is passed. Re-runs then produce
  different fingerprints, and the determinism acceptance check fails.
- **Comparing full signatures without hashing shingles with a stable
  hash function.** `datasketch` defaults to SHA-1 on shingles; switch
  to `xxhash.xxh64_intdigest` for both speed and stability across
  Python versions.
- **Walking the full `O(N^2)` pair graph after LSH.** LSH's whole
  purpose is to give you an `O(N × candidate_density)` pair stream.
  Build the pairs from `lsh.query(m)` per document, deduplicate by
  `doc_a ≤ doc_b`, and verify only those.
- **Union-find without path compression.** At 10M docs you hit worst-
  case `O(N × depth)` behaviour. Use both path compression and union
  by rank.
- **Writing every singleton to `clusters.parquet`.** 95% of docs are
  singletons; the file balloons without carrying information. Filter
  `cluster_size > 1`.
- **Fabricating a memorisation-risk multiplier.** Kandpal 2022 shows a
  superlinear relationship between frequency and regeneration rate.
  It does not endorse "we reduced extraction risk by 47.3x". A
  qualitative, Kandpal-anchored sentence is the correct deliverable.
- **Dropping `per_source_removal_rate`.** Aggregate removal rate hides
  the per-source credibility shifts the dataset-card author
  (`mod-112`) needs to disclose.
- **Reusing Exercise 01's survivor rule without declaring it.** The
  `survivor_rule` field in `§2.granularities` is mandatory; a reader
  cannot infer it from the cluster file alone.

## 6. References

- **`datasketch` documentation** — <https://ekzhu.github.io/datasketch/>.
  `MinHashLSH.insertion_session`, `optimal_param`, and the storage-config
  backends are the API surface you must know.
- **`datasketch` source** — <https://github.com/ekzhu/datasketch>.
  Read `datasketch/lsh.py` to see that `threshold=` is converted to
  `(b, r)` via an optimisation routine; pass `params=` to skip it.
- **xxHash** — <https://github.com/Cyan4973/xxHash>. The stable,
  fast hash function to swap in as `MinHash(..., hashfunc=…)`.
- **Kandpal, N., Wallace, E., Raffel, C., "Deduplicating Training Data
  Mitigates Privacy Risks in Language Models", 2022** —
  <https://arxiv.org/abs/2202.06539>. The anchor paper for the
  memorisation-risk section of the dedup report. The regeneration rate
  as a function of training frequency is superlinear; the deduplication
  reduction in the tail is the headline mitigation.
- **SlimPajama processing scripts** — <https://github.com/Cerebras/modelzoo/tree/main/modelzoo/transformers/data_processing/slimpajama>.
  Use as a practitioner reference for `(K, b, r, J)` on Common Crawl.
  Pin the commit SHA you reference; parameters have drifted across
  branches. <!-- followup-research: confirm current default parameters at the commit SHA the learner actually consults -->
- **OWASP Large Language Model Top 10** —
  <https://genai.owasp.org/llm-top-10/>. LLM02 (Sensitive Information
  Disclosure) in the 2025 edition covers the training-data extraction
  threat; the 2023 edition uses LLM06. Cite the edition you reference.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. AML.T0048 (Reconstruct
  Training Data) and AML.T0024 (Exfiltrate via ML Inference API) are
  the adversary-side techniques this exercise's pipeline weakens.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE 2.10
  (trustworthy characteristics are measured) is the control anchor.
- **Local anchors:** Chapter 03 and Chapter 04 of this module; Exercise
  01 (the exact-dedup primitive that feeds this exercise); Exercise 03
  (paragraph granularity on top of document dedup); `mod-110`
  (privacy-risk report consuming the §4 memorisation-risk block).
