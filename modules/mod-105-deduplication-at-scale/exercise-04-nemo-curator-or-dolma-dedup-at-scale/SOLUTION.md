# Exercise 04 — Dedup at Scale with NeMo Curator, dolma, or data-juicer (Solution)

## 1. Solution overview

This is an **implementation + comparison** exercise. The learner picks
**one** production toolchain — `dolma`, `NeMo Curator`, or
`data-juicer` — reproduces the Exercise 02 `(K=128, b=16, r=8, J=0.8)`
MinHash+LSH document-level pipeline with it, measures throughput /
memory / wall-clock / output size, computes candidate-pair agreement
against the Exercise 02 baseline, and writes a one-page `choice.md`
defending the choice against the two toolchains that were **not**
picked.

The deliverables are:

- A toolchain-specific config file (`dedup.toml`, `dedup.yaml`, or
  `FuzzyDuplicatesConfig(...)` Python snippet) that encodes the
  Exercise 02 parameters.
- A `run_pipeline.sh` that launches the toolchain against the slice.
- `metrics.json` with `docs_per_sec`, `peak_memory_gb`,
  `wall_clock_hours`, and `gpu_memory_gb` where applicable.
- `candidate_pairs.parquet` and a `pair_agreement.json` reporting
  Jaccard-of-pair-sets vs Exercise 02.
- `dedup_report.yaml` with §1 `toolchain` fields updated and §6
  `artifacts` SHAs populated.
- A `choice.md` that defends the chosen toolchain against the two
  alternatives using Chapter 06's decision axes.

The reference worked solution below uses `dolma` as the primary
toolchain and spot-checks the comparison against `data-juicer` and
`NeMo Curator`. The structure is identical across the three choices;
only the config file / pipeline code differs. Both of the other paths
are included as `Alternatives` below.

Governance framing: §1 of the dedup report is the **provenance
primary key** (Chapter 08). A dedup artifact without
`toolchain.{name, version, git_sha, config_sha256}` is unauditable;
`mod-109` cannot sign it, and `mod-112` cannot reference it in the
dataset card. The config SHA is the one field most often missing from
real-world dedup artifacts.

## 2. Implementation — worked answer (primary: `dolma`)

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── Makefile
├── toolchain: dolma
├── choice.md                               # one-pager comparison
├── config/
│   ├── dedup.toml                          # the dolma config
│   └── dedup.toml.sha256
├── code/
│   ├── run_pipeline.sh                     # launches dolma
│   ├── collect_metrics.py                  # scrapes top / psutil / logs
│   ├── export_pairs.py                     # dolma attributes → pair list
│   ├── compare_pairs.py                    # agreement with Exercise 02
│   └── emit_report.py                      # updates §1 + §6 of dedup_report.yaml
└── results/
    ├── pipeline_log.txt
    ├── metrics.json
    ├── candidate_pairs.parquet
    ├── candidate_pairs.parquet.sha256
    ├── pair_agreement.json
    └── dedup_report.yaml
```

### 2.2 `config/dedup.toml`

```toml
# config/dedup.toml
# Reproduce Exercise 02 under dolma.
# Reference: https://github.com/allenai/dolma/blob/main/docs/deduplication.md
# Pin a dolma release (`pip show dolma | grep Version`) and the git SHA.

streams = [
  { name = "my_slice",
    documents = ["./data/my_slice/*.jsonl.gz"],
    attributes = ["doc_dedup"] }
]

# -- Exact dedup — Bloom-filter backed (Chapter 02 Implementation B) --
[dedupers.my_slice_exact]
name = "document_dedup"
documents.key = "text"
documents.attribute_name = "doc_exact_dedup"
bloom_filter.file = "./results/bloom.bin"
bloom_filter.size_in_bytes = 1073741824          # 1 GiB
bloom_filter.estimated_doc_count = 10_000_000
bloom_filter.desired_false_positive_rate = 1e-4
bloom_filter.read_only = false

# -- Fuzzy dedup — MinHash + LSH; matches Exercise 02 --
[dedupers.my_slice_fuzzy]
name = "fuzzy_dedup"
documents.key = "text"
documents.attribute_name = "doc_fuzzy_dedup"
num_permutations = 128
num_bands = 16
rows_per_band = 8
jaccard_threshold = 0.80
ngram_length = 5                                 # word-level 5-shingles
# ^^ NOTE: dolma field names have shifted across releases; the TOML keys
# above reflect `allenai/dolma` main as of 2026-11 and should be
# re-verified against `docs/deduplication.md` at the pinned commit SHA.
```

Three design calls worth defending:

- **Bloom filter at `ε = 1e-4`.** Matches the Exercise 01 target. The
  trade-off (1-in-10k false positive) is acceptable for pretraining; for
  a forensic slice (e.g. legal-data pipeline) you would use dolma's
  exact-set mode and pay the memory cost.
- **`ngram_length = 5`** for word shingles, identical to Exercise 02.
  The shingle-width field name may differ by dolma release; verify
  against `docs/deduplication.md` at the pinned commit.
- **Config SHA** is in `§1.toolchain.config_sha256` and is the one
  thing a change to this file will propagate into the dedup report.
  `sha256sum config/dedup.toml > config/dedup.toml.sha256`.

### 2.3 `code/run_pipeline.sh`

```bash
#!/usr/bin/env bash
# code/run_pipeline.sh
# Launch the dolma pipeline, capture logs + wall clock.
set -euo pipefail

CONFIG=${CONFIG:-config/dedup.toml}
LOG=${LOG:-results/pipeline_log.txt}
START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "run_start_utc=$START" | tee "$LOG"
echo "config_sha256=$(sha256sum "$CONFIG" | cut -d' ' -f1)" | tee -a "$LOG"
echo "dolma_version=$(pip show dolma 2>/dev/null | awk '/Version/{print $2}')" | tee -a "$LOG"

/usr/bin/time -v dolma dedupe --config "$CONFIG" 2>&1 | tee -a "$LOG"

END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "run_end_utc=$END" | tee -a "$LOG"
```

`/usr/bin/time -v` captures peak RSS, wall-clock, and user+system CPU
time. On a GPU box the `nvidia-smi` output is piped alongside it.

### 2.4 `code/collect_metrics.py`

```python
# code/collect_metrics.py
# Parse pipeline_log.txt + dolma's attribute outputs into metrics.json.
from __future__ import annotations

import argparse
import datetime as dt
import json
import re
from pathlib import Path


def parse_time_verbose(log: str) -> dict[str, float]:
    m_wall = re.search(r"Elapsed \(wall clock\) time .*: (.+)", log)
    m_rss = re.search(r"Maximum resident set size \(kbytes\): (\d+)", log)
    wall_clock_hours = None
    if m_wall:
        parts = [float(x) for x in m_wall.group(1).replace(":", " ").split()]
        if len(parts) == 2:
            wall_clock_hours = (parts[0] * 60 + parts[1]) / 3600
        elif len(parts) == 3:
            wall_clock_hours = (parts[0] * 3600 + parts[1] * 60 + parts[2]) / 3600
    peak_rss_gb = int(m_rss.group(1)) / 1024 / 1024 if m_rss else None
    return {"wall_clock_hours": wall_clock_hours, "peak_memory_gb": peak_rss_gb}


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--log", required=True, type=Path)
    ap.add_argument("--input-doc-count", required=True, type=int)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    log = args.log.read_text()
    m = parse_time_verbose(log)
    docs_per_sec = (
        args.input_doc_count / (m["wall_clock_hours"] * 3600)
        if m["wall_clock_hours"]
        else None
    )
    m["docs_per_sec"] = docs_per_sec
    m["gpu_memory_gb"] = None            # dolma is CPU-only
    args.output.write_text(json.dumps(m, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

### 2.5 `code/export_pairs.py`

Dolma emits per-document attribute files alongside the source JSONL.
For `fuzzy_dedup`, the attribute records each document's cluster
assignment. Derive the pair list:

```python
# code/export_pairs.py
# Join dolma's fuzzy_dedup attribute column into a pair list for
# comparison against Exercise 02.
from __future__ import annotations

import argparse
import gzip
import itertools
import json
from collections import defaultdict
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--attr-dir", required=True, type=Path,
                    help="dolma attributes dir for doc_fuzzy_dedup")
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    clusters: dict[str, list[str]] = defaultdict(list)
    for p in sorted(args.attr_dir.glob("*.jsonl.gz")):
        with gzip.open(p, "rt") as fh:
            for line in fh:
                row = json.loads(line)
                doc_id = row["id"]
                for attr in row.get("attributes", {}).get("doc_fuzzy_dedup", []):
                    # dolma attribute shape: [start, end, cluster_id]
                    cluster_id = attr[2]
                    clusters[cluster_id].append(doc_id)

    pairs = []
    for members in clusters.values():
        members.sort()
        for a, b in itertools.combinations(members, 2):
            pairs.append((a, b))

    pq.write_table(
        pa.table({"doc_a": [p[0] for p in pairs], "doc_b": [p[1] for p in pairs]}),
        args.output,
    )


if __name__ == "__main__":
    main()
```

**Pair derivation note.** Dolma emits cluster assignments; the pair
comparison against Exercise 02 is on the derived pairs (one pair per
unique unordered combination of cluster members). This gives you a
strict within-cluster pair set that is directly comparable to the
`verified_pairs.parquet` produced by Exercise 02's
`MinHashLSH.query(m) → verification` chain.

### 2.6 `code/compare_pairs.py`

```python
# code/compare_pairs.py
# Jaccard-of-pair-sets between Exercise 02 and this toolchain's output.
from __future__ import annotations

import argparse
import json
import random
from pathlib import Path

import pyarrow.parquet as pq


def pairs_to_set(parquet_path: Path) -> set[tuple[str, str]]:
    t = pq.read_table(parquet_path, columns=["doc_a", "doc_b"]).to_pandas()
    return {
        (min(a, b), max(a, b))
        for a, b in zip(t["doc_a"], t["doc_b"])
    }


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--baseline", required=True, type=Path,
                    help="Exercise 02 verified_pairs.parquet")
    ap.add_argument("--candidate", required=True, type=Path,
                    help="This exercise's exported pairs")
    ap.add_argument("--output", required=True, type=Path)
    ap.add_argument("--sample-size", type=int, default=1_000)
    ap.add_argument("--seed", type=int, default=42)
    args = ap.parse_args()

    a = pairs_to_set(args.baseline)
    b = pairs_to_set(args.candidate)
    inter = a & b
    uni = a | b
    agreement = len(inter) / len(uni) if uni else 1.0

    # 1,000-pair sample for a manual spot-check, per Exercise 04 §4.
    rng = random.Random(args.seed)
    sample_from_union = rng.sample(sorted(uni), min(args.sample_size, len(uni)))
    sample_agreement = sum(
        1 for p in sample_from_union if (p in a) == (p in b)
    ) / len(sample_from_union)

    out = {
        "jaccard_of_pair_sets": agreement,
        "intersection_size": len(inter),
        "union_size": len(uni),
        "baseline_size": len(a),
        "candidate_size": len(b),
        "sample_size": len(sample_from_union),
        "sample_agreement": sample_agreement,
    }
    args.output.write_text(json.dumps(out, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

### 2.7 `code/emit_report.py`

```python
# code/emit_report.py
# Extend Exercise 02's dedup_report.yaml with §1 toolchain updates and
# §6 artifact SHAs for this toolchain's outputs.
from __future__ import annotations

import argparse
import hashlib
from pathlib import Path

import yaml


def sha256_file(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--baseline-report", required=True, type=Path)
    ap.add_argument("--toolchain-name", required=True,
                    choices=["dolma", "nemo-curator", "data-juicer"])
    ap.add_argument("--toolchain-version", required=True)
    ap.add_argument("--toolchain-git-sha", required=True)
    ap.add_argument("--config-path", required=True, type=Path)
    ap.add_argument("--candidate-pairs", required=True, type=Path)
    ap.add_argument("--run-log", required=True, type=Path)
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    report = yaml.safe_load(args.baseline_report.read_text())
    report["toolchain"] = {
        "name": args.toolchain_name,
        "version": args.toolchain_version,
        "git_sha": args.toolchain_git_sha,
        "config_path": str(args.config_path),
        "config_sha256": sha256_file(args.config_path),
    }
    report.setdefault("artifacts", {})
    report["artifacts"].update(
        {
            "candidate_pairs_path": str(args.candidate_pairs),
            "candidate_pairs_sha256": sha256_file(args.candidate_pairs),
            "pipeline_log_path": str(args.run_log),
            "pipeline_log_sha256": sha256_file(args.run_log),
        }
    )
    args.output.write_text(yaml.safe_dump(report, sort_keys=False))


if __name__ == "__main__":
    main()
```

### 2.8 `choice.md` — defend the choice

The one-page structure the reviewer expects:

```markdown
# Toolchain choice for slice <slice-id>: dolma

**Slice:** <slice-id>, <N> docs, ~<tokens> tokens.
**Team shape:** <n-engineer data team>, CPU-only cluster, no managed Ray.
**Deployment target:** bare-metal CPU box with 512 GiB RAM.

## Numbers

| metric                  | datasketch (Ex 02) | dolma   | delta                 |
|-------------------------|---------------------|---------|-----------------------|
| wall clock              | <t1>                | <t2>    | <...>                 |
| peak memory             | <m1>                | <m2>    | <...>                 |
| docs/sec                | <d1>                | <d2>    | <...>                 |
| pair agreement          | 1.00 (baseline)     | <a>     | expected ≥ 0.75       |

## Why dolma over NeMo Curator and data-juicer

**dolma:** chosen. Config-driven, CPU-only, used in a public pretraining
release (Dolma). Bloom-filter exact dedup is first-class. No GPU
dependency and no Ray / Spark buy-in required. The team has JSONL+gzip
inputs already.

**NeMo Curator:** rejected. GPU dependency is a strict pessimisation
here — the team has no GPU capacity. NeMo Curator would be the right
choice if the slice were ≥ 10^9 docs on an existing A100 cluster.

**data-juicer:** rejected. Ray cluster not available and no multimodal
dedup is needed for this slice. If the next slice is multimodal, revisit.

## Open questions

- Pair agreement is <a>; investigate if < 0.75 (likely a config
  mismatch — recheck ngram width and threshold).
- Bloom-filter false-positive rate observed is <obs>; target is 1e-4.
  Within one order of magnitude is acceptable.

<!-- followup-research: confirm dolma main ngram field name (ngram_length vs shingle_length) at the pinned commit SHA -->
```

### 2.9 Alternatives — the same shape in NeMo Curator and data-juicer

**NeMo Curator** (GPU box; Dask-CUDA):

```python
# nemo-curator.py (alternative to dedup.toml)
from nemo_curator import DocumentDataset
from nemo_curator.modules.fuzzy_dedup import FuzzyDuplicates, FuzzyDuplicatesConfig

config = FuzzyDuplicatesConfig(
    cache_dir="./nemo_cache",
    id_field="doc_id",
    text_field="text",
    seed=0xC0FFEE,
    num_hashes=128,           # == num_perm in Exercise 02
    num_buckets=16,           # == bands
    hashes_per_bucket=8,      # == rows
    jaccard_threshold=0.80,
    char_ngrams=24,           # ≈ 5-word-shingle for English prose
    use_64_bit_hash=False,
)

ds = DocumentDataset.read_parquet("s3://.../my_slice/*.parquet")
dedup = FuzzyDuplicates(config=config)
duplicates = dedup(ds)
duplicates.to_parquet("results/candidate_pairs.parquet")
```

Document the `char_ngrams=24` approximation in `choice.md` — it is
**not** a word-5-shingle, which Exercise 02 used; it is character
n-grams. Approximation rationale: 24-char windows roughly span the
5-word boundary for mean English word length ~5 chars. Expect pair
agreement to drop to the 0.6–0.8 range as a result. If agreement < 0.5,
it is almost certainly a different shingling choice, not a bug.

**data-juicer** (Ray cluster):

```yaml
# data-juicer.yaml (alternative to dedup.toml)
project_name: my_slice_dedup
dataset_path: ./data/my_slice.jsonl
export_path: ./data/my_slice_dedup.jsonl
np: 32

process:
  - document_deduplicator:               # exact (Chapter 02); MD5 hash
      lowercase: false
      ignore_non_character: false
  - document_minhash_deduplicator:
      tokenization: space
      window_size: 5                     # word 5-shingle (matches Ex 02)
      num_permutations: 128
      jaccard_threshold: 0.80
      num_bands: 16
      num_rows_per_band: 8
      lowercase: false
```

Pin `data-juicer` tightly. Operator names and keyword arguments shift
across releases; verify at the pinned commit.

### 2.10 `Makefile`

```makefile
.PHONY: all pipeline metrics compare report clean

all: pipeline metrics compare report

pipeline:
	bash code/run_pipeline.sh

metrics:
	python code/collect_metrics.py --log results/pipeline_log.txt \
		--input-doc-count $$(cat data/input_doc_count.txt) \
		--output results/metrics.json

compare:
	python code/export_pairs.py --attr-dir data/my_slice/attributes/doc_fuzzy_dedup \
		--output results/candidate_pairs.parquet
	python code/compare_pairs.py --baseline ../exercise-02/results/verified_pairs.parquet \
		--candidate results/candidate_pairs.parquet \
		--output results/pair_agreement.json

report:
	python code/emit_report.py \
		--baseline-report ../exercise-02/results/dedup_report.yaml \
		--toolchain-name dolma \
		--toolchain-version $$(pip show dolma | awk '/Version/{print $$2}') \
		--toolchain-git-sha $$(git -C /opt/dolma rev-parse HEAD) \
		--config-path config/dedup.toml \
		--candidate-pairs results/candidate_pairs.parquet \
		--run-log results/pipeline_log.txt \
		--output results/dedup_report.yaml

clean:
	rm -rf results/*.parquet results/metrics.json results/pair_agreement.json \
	       results/dedup_report.yaml results/pipeline_log.txt
```

## 3. Validation steps

1. **Config SHA is in the report.** `yq '.toolchain.config_sha256'
   results/dedup_report.yaml` equals `sha256sum config/dedup.toml`.
2. **Toolchain version pinned.** `yq '.toolchain.version'` is not
   empty; `yq '.toolchain.git_sha'` is a 40-hex string.
3. **Metrics sanity.** `metrics.json` has `docs_per_sec > 0`,
   `peak_memory_gb > 0`, `wall_clock_hours > 0`. GPU memory is `null`
   for dolma / data-juicer on CPU; a positive number for NeMo Curator.
4. **Pair-set agreement is ≥ 0.75.** `jaccard_of_pair_sets` in
   `pair_agreement.json` ≥ 0.75. Below 0.5 is a configuration
   mismatch — investigate (shingle width, threshold, canonicalisation).
5. **Sample agreement audit.** `sample_agreement` in
   `pair_agreement.json` is close to `jaccard_of_pair_sets` (within
   ±0.05); large divergence means the pair-set distribution is skewed.
6. **`choice.md` compares against both alternatives.** Each of the
   non-selected toolchains has a rejection reason tied to Chapter 06's
   decision axes.
7. **Bloom-filter FP rate observed.** If the chosen path uses dolma's
   Bloom-filter exact dedup, the observed FP rate is within one order
   of magnitude of the `desired_false_positive_rate` in the TOML.
8. **No GPU metric for a CPU-only toolchain.** `gpu_memory_gb` is
   `null` for dolma / data-juicer; present and non-null for NeMo
   Curator on a GPU box.
9. **Approximation flags.** If the toolchain forced a deviation
   (character n-grams vs word shingles for NeMo Curator; Bloom-filter
   vs exact set for dolma), `choice.md` records it explicitly.
10. **Any unverifiable claim is tagged.** `grep "followup-research"
    choice.md README.md` surfaces every pending verification.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible
with evidence). Pass threshold: ≥ 20/28 with zero zeros on critical rows.

| #  | Dimension                                                                                           | Critical? |
|----|------------------------------------------------------------------------------------------------------|-----------|
| 1  | Toolchain config file encodes `K=128, b=16, r=8, J=0.8` (or documents an approximation)              | Yes       |
| 2  | Config file is SHA-256-pinned and the SHA is in the dedup report                                     | Yes       |
| 3  | Toolchain version, git SHA, and config path are in `§1.toolchain.*`                                  | Yes       |
| 4  | `metrics.json` has docs/sec, peak RSS, wall clock; GPU memory if applicable                          | Yes       |
| 5  | Pair-set agreement vs Exercise 02 is ≥ 0.75                                                          | Yes       |
| 6  | 1,000-pair sample agreement is within ±0.05 of the pair-set agreement                                | No        |
| 7  | `choice.md` compares against both non-selected toolchains on Chapter 06's axes                       | Yes       |
| 8  | Approximations (character-ngram vs word-shingle, bloom vs set) are called out explicitly             | Yes       |
| 9  | §6 `artifacts.*_sha256` populated for candidate_pairs and pipeline_log                               | Yes       |
| 10 | `pipeline_log.txt` is committed (or its SHA is committed and the log is archived)                    | No        |
| 11 | `Makefile` reproduces the full pipeline from clean clone                                             | No        |
| 12 | Governance anchor (OWASP, MITRE ATLAS, NIST AI RMF) cited in the README                              | No        |
| 13 | Any unverifiable claim flagged with `<!-- followup-research: ... -->`                                | Yes       |
| 14 | `Makefile` target for `report` is parameter-free (no learner-specific paths hard-coded in code)      | No        |

## 5. Common mistakes

- **Trusting the toolchain's defaults.** Each toolchain's defaults
  have moved across releases: `datasketch` ships at `(b, r)` optimised
  for a fixed threshold; dolma's `ngram_length` default has changed;
  NeMo Curator's `num_buckets` default has changed to 20/13 in recent
  releases. Pin everything and re-verify at the commit SHA you run.
- **Comparing word-shingle vs character-n-gram outputs as if they are
  equivalent.** They are not. NeMo Curator uses character n-grams; a
  `char_ngrams=24` setting is an *approximation* of word 5-shingles.
  Pair agreement will be lower; that is expected and must be
  documented, not treated as a bug.
- **Omitting `config_sha256` from `§1`.** The single most common gap in
  real-world dedup artifacts. A config change that silently rolls out
  into a new pipeline run cannot be audited without a config SHA.
- **Measuring wall-clock without peak RSS.** Throughput without memory
  is uninterpretable. A pipeline at 10k docs/sec peaking at 480 GiB is
  a different operational story than one at 3k docs/sec peaking at
  60 GiB.
- **Not pinning the toolchain version.** `pip show <toolchain>` into
  `requirements.lock` and the git SHA of the toolchain's repo into the
  dedup report. Both. "Latest" is not a version.
- **Writing `choice.md` without rejecting the other two toolchains
  concretely.** Chapter 06 is organised around decision axes
  (deployment, operator model, accelerator, scale sweet spot). Each
  non-selected toolchain should be rejected against those axes, not
  with vague prose.
- **Treating pair agreement < 0.75 as acceptable.** Chapter 04's rule
  of thumb is that two correctly-configured MinHash+LSH implementations
  at the same `(K, b, r, J)` on the same shingles should agree on at
  least ~75% of near-duplicate pairs. Lower agreement usually means a
  shingle-width mismatch, a threshold mismatch, or a canonicalisation
  mismatch — investigate, do not accept.
- **Running the pipeline on a resharded version of the input and
  comparing against Exercise 02's pair list.** Doc IDs must be stable
  across the two runs; otherwise you are comparing apples to oranges.

## 6. References

- **`dolma`** — <https://github.com/allenai/dolma>. Primary toolchain
  for the worked solution; `docs/deduplication.md` at the pinned
  commit is the field reference. Pin the version aggressively.
- **Soldaini, L., Kinney, R., Bhagia, A. et al., "Dolma: an Open
  Corpus of Three Trillion Tokens for Language Model Pretraining
  Research", ACL 2024** —
  <https://aclanthology.org/2024.acl-long.840/>. Methodology paper
  describing the dolma dedup toolchain's design.
- **`NeMo Curator`** — <https://github.com/NVIDIA/NeMo-Curator>. NVIDIA's
  GPU-accelerated curator. `FuzzyDuplicatesConfig` is the API surface;
  defaults have shifted across releases (`num_buckets=20,
  hashes_per_bucket=13` as of 2026-11; verify at the pinned release).
- **`data-juicer`** — <https://github.com/modelscope/data-juicer>.
  Alibaba's Ray-based operator framework;
  `document_minhash_deduplicator` and `document_deduplicator` are the
  two relevant operators.
- **Chen, D., Dou, Y., Li, Y., Deng, Y., Shen, Y. et al., "Data-Juicer:
  A One-Stop Data Processing System for Large Language Models", arXiv
  2309.02033, 2023** — <https://arxiv.org/abs/2309.02033>.
- **FineWeb / `datatrove`** —
  <https://github.com/huggingface/datatrove>. Stretch-goal fourth
  toolchain; shares shape with dolma. HuggingFace's FineWeb pipeline
  runs on it.
- **SlimPajama processing scripts** —
  <https://github.com/Cerebras/modelzoo/tree/main/modelzoo/transformers/data_processing/slimpajama>.
  Stretch-goal reproduction target. Parameters have drifted across
  branches; pin the commit SHA you reference. <!-- followup-research: confirm current SlimPajama (b, r, J) defaults at the commit SHA the learner actually consults -->
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 anchor for why a reproducible, SHA-pinned toolchain run matters.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. AML.T0019 / AML.T0020
  for the adversary model this pipeline weakens.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN 1.7
  ("Processes and procedures are in place for decommissioning and
  phasing out of AI systems safely"), MEASURE 2.7, and MANAGE 4.1 all
  touch the toolchain-provenance surface.
- **Local anchors:** Chapter 06 (this exercise's primary content);
  Exercise 02 (the datasketch baseline this exercise compares against);
  Exercise 05 (adversarial-perturbation review on top of the pipeline
  output); Chapter 08 (the report schema `§1.toolchain.*` populates).
