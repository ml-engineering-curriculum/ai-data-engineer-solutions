# Exercise 03 — Paragraph vs Document vs Sentence Dedup Trade-Off (Solution)

## 1. Solution overview

This is a **design-with-implementation** exercise. The learner runs the
same slice through three dedup pipelines — document, document +
paragraph, document + paragraph + targeted sentence — and produces a
side-by-side comparison showing *what each granularity removes, what it
keeps, and which downstream risk it addresses*. The output is a
judgement call defensible with evidence: ship pipeline B or pipeline C
for this slice, and here is why.

The three deliverable variants:

- **Pipeline A — document only** (= Exercise 02 output). `K=128`,
  `(b, r) = (16, 8)`, `J = 0.8`.
- **Pipeline B — document + paragraph.** On top of A, split surviving
  documents into paragraphs, drop paragraphs < 100 chars, MinHash+LSH
  at `K=128`, `(b, r) = (16, 8)`, `J = 0.85`, keep earliest-crawl-date
  paragraph per cluster, re-assemble.
- **Pipeline C — document + paragraph + targeted sentence.** On top of
  B, run **exact** dedup at the sentence level, but **only on
  sentences that match a benchmark-n-gram set or a PII heuristic.**
  Keep first occurrence, drop subsequent.

The design choice that most distinguishes B from C: **sentence-level
dedup is targeted, not blanket.** Blanket sentence dedup is actively
wrong on prose slices — it removes polite closings, standard legal
phrasing, and conversational idioms that carry linguistic signal. The
comparison report must contain evidence of that harm (the 20 idiomatic
sentences that blanket dedup would have removed).

Governance framing: granularity is a **threat-model choice**, not a
technical knob. Paragraph dedup is the Chapter 05 answer to Kandpal
2022's boilerplate-memorisation finding; targeted sentence dedup is the
bridge to `mod-106` (benchmark decontamination) and `mod-110` (PII
appearance counts). OWASP LLM Top 10 **LLM02 Sensitive Information
Disclosure** (2025 edition; **LLM06** in 2023) and OWASP ML Top 10
**ML02 Data Poisoning Attack** are the two anchors; MITRE ATLAS
**AML.T0048 Reconstruct Training Data** is the adversary technique
paragraph dedup weakens.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── requirements.lock
├── Makefile
├── comparison.md                                # the deliverable write-up
├── code/
│   ├── paragraph_split.py
│   ├── paragraph_dedup.py                       # MinHash + LSH at paragraph
│   ├── sentence_target_set.py                   # benchmark n-grams + PII regexes
│   ├── sentence_dedup_targeted.py               # exact dedup on target sentences
│   ├── reassemble.py
│   ├── emit_report.py                           # extends Exercise 02's YAML
│   └── comparison_samples.py                    # 20 boilerplate + 20 idiom samples
├── data/
│   ├── benchmark_ngrams.txt
│   └── pii_regex.yaml
└── results/
    ├── doc_only.parquet                         # = Exercise 02 output
    ├── doc_plus_paragraph.parquet
    ├── doc_plus_paragraph_plus_sentence.parquet
    ├── paragraph_clusters.parquet
    ├── paragraph_clusters.parquet.sha256
    ├── targeted_sentence_clusters.parquet
    └── dedup_report.yaml                        # §3 paragraph + targeted sentence
```

### 2.2 `code/paragraph_split.py`

```python
# code/paragraph_split.py
# HTML <p> if source HTML is present; fall back to double-newline.
# Chapter 02 canonicalisation is applied per paragraph (not per document).
from __future__ import annotations

import unicodedata
from dataclasses import dataclass

from bs4 import BeautifulSoup


@dataclass(slots=True, frozen=True)
class Paragraph:
    doc_id: str
    paragraph_idx: int                 # 0-based position inside the doc
    text: str                          # canonical form, same policy as Ex 01
    source_html: bool                  # True if HTML-boundary-extracted


MIN_CHARS = 100


def canonicalise_paragraph(s: str) -> str:
    return unicodedata.normalize("NFC", " ".join(s.strip().split()))


def split_html(doc_id: str, html: str) -> list[Paragraph]:
    soup = BeautifulSoup(html, "lxml")
    blocks = soup.find_all(["p", "li", "blockquote"])
    out: list[Paragraph] = []
    for i, blk in enumerate(blocks):
        text = canonicalise_paragraph(blk.get_text(" ", strip=True))
        if len(text) >= MIN_CHARS:
            out.append(Paragraph(doc_id, i, text, source_html=True))
    return out


def split_text(doc_id: str, text: str) -> list[Paragraph]:
    chunks = [c for c in text.split("\n\n") if c.strip()]
    out: list[Paragraph] = []
    for i, c in enumerate(chunks):
        p = canonicalise_paragraph(c)
        if len(p) >= MIN_CHARS:
            out.append(Paragraph(doc_id, i, p, source_html=False))
    return out
```

Two defensible design calls:

- **Keep the paragraph_idx.** Re-assembly must restore order; the index
  is the join key. Without it, a document's paragraph order is
  wall-clock dependent.
- **Short-paragraph floor at 100 chars.** The Chapter 03 "short
  document" problem applies to paragraphs: a MinHash over a 20-shingle
  set is high-variance. Treat 100 chars as the operational floor;
  revisit per corpus if needed and document the choice in
  `comparison.md`.

### 2.3 `code/paragraph_dedup.py`

```python
# code/paragraph_dedup.py
# MinHash+LSH at paragraph granularity. Reuses the Exercise 02 code with
# a tighter Jaccard threshold (0.85) because paragraphs are shorter.
from __future__ import annotations

# Imports and helpers reused from Exercise 02:
#   build_signature(text, seed) and materialise_minhash(sig_bytes)
from datasketch import MinHash, MinHashLSH

NUM_PERM = 128
BANDS = 16
ROWS = 8
JACCARD_THRESHOLD = 0.85


def dedup_paragraphs(paragraphs, seed=0xC0FFEE):
    # paragraphs: iterable of (paragraph_id, Paragraph) where paragraph_id
    # is something globally unique like f"{doc_id}:{paragraph_idx}".
    lsh = MinHashLSH(num_perm=NUM_PERM, params=(BANDS, ROWS))
    signatures: dict[str, MinHash] = {}
    with lsh.insertion_session() as session:
        for pid, p in paragraphs:
            m = build_signature(p.text, seed)
            session.insert(pid, m)
            signatures[pid] = m

    pairs = []
    for pid, m in signatures.items():
        for candidate in lsh.query(m):
            if candidate <= pid:
                continue
            if m.jaccard(signatures[candidate]) >= JACCARD_THRESHOLD:
                pairs.append((pid, candidate))
    return pairs
```

Justification for `J = 0.85` (vs document-level `0.80`):

- The `datasketch` MinHash Jaccard estimator has variance `~1/K`. At
  `K = 128` the standard deviation is about `0.09` when true `J = 0.5`
  and falls at both tails. On short paragraphs with small shingle sets
  the estimator is noisier — raising the threshold compensates.
- The empirical Chapter 05 rule of thumb used by `dolma`'s paragraph
  dedup is "paragraph threshold = document threshold + 0.05". We follow
  it.

### 2.4 Survivor rule — earliest crawl date

```python
# cluster_resolve_paragraph.py
def resolve_cluster(members, metadata):
    # members: list of paragraph_id strings in the same cluster
    # metadata: dict paragraph_id → {crawl_date, text, doc_id, paragraph_idx}
    survivors = sorted(members, key=lambda pid: (
        metadata[pid]["crawl_date"], pid))  # pid breaks ties deterministically
    return survivors[0], survivors[1:]
```

Chapter 05 recommends the **earliest-crawl-date paragraph** as
representative for paragraph dedup, because provenance (where did this
paragraph *first* appear in the pipeline's view of the web?) is more
valuable than length for the downstream dataset card. The exercise's
acceptance criteria accept "longest" as a fallback **only if justified
in `comparison.md`**.

### 2.5 `code/reassemble.py`

```python
# code/reassemble.py
# Replace duplicate paragraphs with a canonical token OR drop them.
# The choice is recorded in comparison.md; both have semantic costs.
from __future__ import annotations

import pyarrow as pa
import pyarrow.parquet as pq

DUPLICATE_MARKER = "<DUPLICATE_PARAGRAPH>"


def reassemble(
    paragraphs_parquet,
    removed_paragraph_ids_set,
    policy: str = "mark",         # "mark" | "drop"
):
    assert policy in {"mark", "drop"}, policy
    table = pq.read_table(paragraphs_parquet).to_pandas()
    docs: dict[str, list[tuple[int, str]]] = {}
    for _, r in table.iterrows():
        pid = f"{r['doc_id']}:{r['paragraph_idx']}"
        if pid in removed_paragraph_ids_set:
            if policy == "drop":
                continue
            docs.setdefault(r["doc_id"], []).append(
                (r["paragraph_idx"], DUPLICATE_MARKER)
            )
        else:
            docs.setdefault(r["doc_id"], []).append(
                (r["paragraph_idx"], r["text"])
            )
    reassembled = []
    for doc_id, parts in docs.items():
        parts.sort()
        reassembled.append(
            {"doc_id": doc_id, "text": "\n\n".join(p[1] for p in parts)}
        )
    return reassembled
```

**`mark` vs `drop` is an audit decision.** `mark` preserves
document-level provenance (a reader can see something was removed and
where); `drop` yields cleaner training text but silently erases
evidence. The exercise asks you to pick one and record it; a slice that
feeds a GRC audit flow usually wants `mark`.

### 2.6 `code/sentence_target_set.py`

```python
# code/sentence_target_set.py
# Build the sentence target set: benchmark n-grams + toy PII regexes.
# Both are explicit, version-controlled files.
from __future__ import annotations

import re
from pathlib import Path

import yaml


def load_benchmark_ngrams(path: Path) -> set[str]:
    # The Chapter 02 canonical form is applied so the sentence comparison
    # matches the same normalisation as the rest of the pipeline.
    import unicodedata
    out: set[str] = set()
    for line in path.read_text().splitlines():
        s = line.strip()
        if not s or s.startswith("#"):
            continue
        out.add(unicodedata.normalize("NFC", " ".join(s.split())))
    return out


def load_pii_regexes(path: Path) -> list[re.Pattern]:
    cfg = yaml.safe_load(path.read_text())
    return [re.compile(p) for p in cfg["patterns"]]


def is_target(sentence: str, ngrams: set[str], regexes: list[re.Pattern]) -> str | None:
    """Return the reason-code if this sentence should be targeted, else None."""
    if sentence in ngrams:
        return "benchmark_ngram"
    for r in regexes:
        if r.search(sentence):
            return "pii_heuristic"
    return None
```

Reasonable minimal `data/pii_regex.yaml`:

```yaml
# data/pii_regex.yaml
# Toy heuristic — not a real PII detector. mod-110 ships the real one.
patterns:
  - "(?i)(?<![a-z0-9._%+-])[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}(?![a-z])"   # email
  - "(?<!\\d)(?:\\+?\\d{1,3}[\\s-])?\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}(?!\\d)"  # US phone
```

Reasonable minimal `data/benchmark_ngrams.txt`:

```
# Prefix sentences of 20 famous benchmark questions. One per line.
# This is a learning-exercise preview of mod-106 benchmark decontamination
# (expanded with a real contamination set there).
What is the capital of France?
Which planet is closest to the Sun?
...
```

Both files are version-controlled; the `targeted_sentence` block of the
dedup report cites the SHA-256 of both, so a reader can tell what was
in scope at report time.

### 2.7 `code/sentence_dedup_targeted.py`

```python
# code/sentence_dedup_targeted.py
# Sentence-level EXACT dedup, applied ONLY to target sentences.
# Keep first occurrence, drop subsequent.
from __future__ import annotations

from pathlib import Path
import unicodedata

import pyarrow as pa
import pyarrow.parquet as pq
import pysbd

from sentence_target_set import (
    is_target, load_benchmark_ngrams, load_pii_regexes
)


def canonical_sentence(s: str) -> str:
    return unicodedata.normalize("NFC", " ".join(s.strip().split()))


def dedup_targeted(input_parquet: Path,
                   ngrams_path: Path,
                   regex_path: Path,
                   output_parquet: Path,
                   clusters_out: Path):
    seg = pysbd.Segmenter(language="en", clean=False)
    ngrams = load_benchmark_ngrams(ngrams_path)
    regexes = load_pii_regexes(regex_path)

    seen: dict[str, tuple[str, int]] = {}  # canonical sentence -> (doc_id, idx)
    cluster_rows: list[tuple[str, str, str, int, str]] = []
    reassembled: list[dict] = []

    for row in pq.read_table(input_parquet).to_pylist():
        doc_id = row["doc_id"]
        kept_sentences: list[str] = []
        for i, sentence in enumerate(seg.segment(row["text"])):
            canon = canonical_sentence(sentence)
            reason = is_target(canon, ngrams, regexes)
            if reason is None:
                kept_sentences.append(sentence)
                continue
            if canon in seen:
                cluster_rows.append(
                    (canon, seen[canon][0], doc_id, i, reason)
                )
                continue
            seen[canon] = (doc_id, i)
            kept_sentences.append(sentence)
        reassembled.append(
            {"doc_id": doc_id, "text": " ".join(kept_sentences)}
        )

    pq.write_table(pa.table(reassembled), output_parquet)
    pq.write_table(
        pa.table(
            {
                "sentence_canonical": [r[0] for r in cluster_rows],
                "representative_doc_id": [r[1] for r in cluster_rows],
                "removed_doc_id": [r[2] for r in cluster_rows],
                "removed_sentence_idx": [r[3] for r in cluster_rows],
                "reason_code": [r[4] for r in cluster_rows],
            }
        ),
        clusters_out,
    )
```

Two things worth defending at review:

- **Exact dedup on the canonical sentence form, not MinHash.** At
  sentence granularity on a *targeted* set, exact-match is the right
  primitive: the target set is small, the sentences are short, and the
  Jaccard-noise argument from §2.3 cuts even harder at sentence scale.
- **`clean=False` on `pysbd`.** The Chapter 02 canonicalisation is the
  single source of truth for whitespace / Unicode; we do not let the
  segmenter apply a second normalisation.

### 2.8 Comparison samples

```python
# code/comparison_samples.py
# 20 boilerplate paragraphs removed by B but not A — with sources + counts.
# 20 idiomatic sentences that BLANKET sentence dedup WOULD have removed.
from __future__ import annotations

import pyarrow.parquet as pq
from collections import Counter


def sample_boilerplate_paragraphs(paragraph_clusters_parquet,
                                   top_n: int = 20,
                                   min_sources: int = 5):
    # Boilerplate = paragraphs appearing across many sources.
    df = pq.read_table(paragraph_clusters_parquet).to_pandas()
    grouped = (
        df.groupby("cluster_id")
          .agg(
              size=("paragraph_id", "count"),
              distinct_sources=("source", "nunique"),
              sample_text=("text", "first"),
          )
          .reset_index()
    )
    boilerplate = grouped[grouped["distinct_sources"] >= min_sources]
    return boilerplate.sort_values("size", ascending=False).head(top_n)


def sample_idiomatic_sentences(sampled_blanket_dedup_parquet, top_n: int = 20):
    # A side-run of BLANKET sentence dedup on a 10k-doc sample tells us
    # what common idioms WOULD have been removed if C had been "untargeted".
    df = pq.read_table(sampled_blanket_dedup_parquet).to_pandas()
    counts = Counter(df["sentence_canonical"])
    # Idioms = short, high-frequency sentences
    idioms = [
        (s, n) for s, n in counts.most_common(100)
        if len(s) <= 80 and n >= 50
    ]
    return idioms[:top_n]
```

### 2.9 Extending the dedup report — the paragraph + targeted-sentence
### blocks

The Chapter 08 schema fields to populate (extending Exercise 02's YAML):

```yaml
granularities:
  - level: document
    # (populated by Exercise 02)
  - level: paragraph
    paragraph_splitter: html_block
    min_chars: 100
    shingle_tokenization: whitespace
    shingle_width: 5
    minhash_num_perm: 128
    lsh_bands: 16
    lsh_rows_per_band: 8
    jaccard_threshold: 0.85
    survivor_rule: earliest_crawl
    reassembly_policy: mark          # or drop — document the choice
  - level: sentence
    scope: targeted                   # NOT blanket
    sentence_splitter: pysbd
    target_set:
      benchmark_ngrams_path: data/benchmark_ngrams.txt
      benchmark_ngrams_sha256: <sha256>
      pii_regex_path: data/pii_regex.yaml
      pii_regex_sha256: <sha256>
    dedup_primitive: exact            # xxh128 on canonical sentence form
    survivor_rule: first_occurrence

near_dedup_paragraph:
  pre_paragraph_count: <N>
  post_paragraph_count: <N - removed>
  removal_rate: <...>
  clusters_formed: <...>
  mean_cluster_size: <...>
  max_cluster_size: <...>
  per_source_removal_rate:
    <source>: <rate>
    # ≥ 5 sources
  reassembly_policy: mark
```

And the targeted-sentence block:

```yaml
targeted_sentence_dedup:
  reason_codes_removed:
    benchmark_ngram: <count>
    pii_heuristic: <count>
  benchmark_ngrams_sha256: <sha256>
  pii_regex_sha256: <sha256>
  # NOTE: this is NOT a benchmark decontamination pass; mod-106 ships that.
  caveat: "preview of mod-106; not a decontamination guarantee"
```

### 2.10 `comparison.md` — the headline deliverable

Four mandatory sections:

**(1) Removal-rate table.** Per pipeline per granularity.

| pipeline | doc removal rate | paragraph removal rate | targeted sentence removal rate |
|---|---|---|---|
| A (doc only) | <x>% | — | — |
| B (doc + para) | <x>% | <y>% | — |
| C (doc + para + targeted sentence) | <x>% | <y>% | <z>% |

**(2) 20 boilerplate paragraphs removed by B but not A.**
Each row: canonical paragraph text (first 80 chars), cluster size,
number of distinct sources, three representative source URLs.

**(3) 20 idiomatic sentences blanket sentence dedup *would have*
removed.** Produced by a side-run of blanket sentence dedup on a 10k-doc
sample of C's input. Each row: sentence (canonical form), frequency,
sentence length.

**(4) Judgement call: B or C for production?** One paragraph. The
defensible answer for a Common-Crawl-style web slice is "B, plus
targeted-sentence dedup gated on `mod-106`'s contamination list and
`mod-110`'s PII appearance counts". Pure B leaves benchmark overlap on
the table; pure C (blanket) destroys idiom distributions. The targeted
flavour of C is the shippable answer.

## 3. Validation steps

1. **Three variant files produced.** `ls results/*.parquet` includes
   `doc_only.parquet`, `doc_plus_paragraph.parquet`, and
   `doc_plus_paragraph_plus_sentence.parquet`.
2. **Sidecar SHA-256 files match.** For each variant parquet, `sha256sum`
   matches the `.sha256` sidecar.
3. **Removal-rate table conservation.** At each granularity, pre − post
   = removed-count. The row counts in `removed_docs.parquet` /
   `paragraph_clusters.parquet` / `targeted_sentence_clusters.parquet`
   must match the report's `removal_rate × pre_count`.
4. **≥ 5 distinct sources in paragraph `per_source_removal_rate`.**
   `yq '.near_dedup_paragraph.per_source_removal_rate | keys | length'
   results/dedup_report.yaml` ≥ 5.
5. **20 boilerplate paragraphs with sources.** `comparison.md` contains
   a table of 20 rows; each row has canonical text, cluster size,
   distinct sources, and at least one cited URL.
6. **20 idiomatic sentences.** `comparison.md` contains a 20-row table
   from the blanket-dedup side-run.
7. **Benchmark n-gram list and PII regex file are SHA-pinned in the
   report.** `yq '.granularities[] | select(.level == "sentence")
   .target_set.benchmark_ngrams_sha256'` returns a 64-hex string;
   same for `pii_regex_sha256`.
8. **At least one re-assembly boundary case called out.** The
   comparison.md explicitly notes one document where `<DUPLICATE_PARAGRAPH>`
   vs drop changes semantic flow (e.g. a Q&A doc whose answer is
   boilerplate; dropping it collapses two questions together).
9. **Targeted-sentence caveat.** The `targeted_sentence_dedup.caveat`
   field in the report explicitly says this is **not** a full
   `mod-106` decontamination.
10. **Determinism.** Re-running the emitter on the same inputs produces
    a byte-identical `dedup_report.yaml` (modulo `run_end_utc`).

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
Pass threshold: ≥ 22/30 with zero zeros on critical rows.

| #  | Dimension                                                                                           | Critical? |
|----|------------------------------------------------------------------------------------------------------|-----------|
| 1  | Three variant slices (A/B/C) produced; sidecar SHA-256 files match                                   | Yes       |
| 2  | Paragraph splitter is HTML-aware when source HTML is present; falls back to `\n\n` otherwise          | Yes       |
| 3  | Paragraph min-length gate (≥ 100 chars) is applied and justified                                     | Yes       |
| 4  | Paragraph MinHash+LSH uses `(b, r)` explicitly and `J* = 0.85`                                       | Yes       |
| 5  | Paragraph survivor rule is `earliest_crawl` by default; alternative is justified                     | Yes       |
| 6  | Re-assembly policy (`mark` vs `drop`) is declared in the report and defended in `comparison.md`      | Yes       |
| 7  | Sentence target set comes from a **versioned** benchmark-n-gram file + PII regex file (both SHA-pinned) | Yes    |
| 8  | Targeted sentence dedup is **exact**, not MinHash (correct primitive at this scale)                  | Yes       |
| 9  | `comparison.md` has the 20-boilerplate table with sources + counts                                   | Yes       |
| 10 | `comparison.md` has the 20-idiomatic-sentence table from a blanket-dedup side-run                    | Yes       |
| 11 | `comparison.md` has a judgement paragraph choosing B or C and defending it                           | Yes       |
| 12 | `dedup_report.yaml` populates `near_dedup_paragraph` and `targeted_sentence_dedup` per Chapter 08    | Yes       |
| 13 | Report explicitly notes "not a full mod-106 decontamination"                                         | Yes       |
| 14 | At least one re-assembly edge case called out in `comparison.md`                                     | No        |
| 15 | Any unverifiable claim flagged with `<!-- followup-research: ... -->`                                | Yes       |

## 5. Common mistakes

- **Blanket sentence dedup "to be thorough".** Catastrophic for prose
  registers. Chapter 05 is explicit: blanket sentence dedup destroys
  polite closings and legal boilerplate, which is *signal* in prose
  slices. Target the set.
- **Paragraph threshold reused from document (J = 0.8).** Too loose —
  short paragraphs with small shingle sets are noisy. Raise to 0.85
  per Chapter 05 rule of thumb.
- **No short-paragraph floor.** Paragraphs under 100 chars are
  high-variance noise under MinHash. Either skip them or use exact
  dedup instead.
- **Keeping the longest paragraph instead of earliest-crawl.** Longest
  is the Exercise 01 document default; at paragraph granularity
  earliest-crawl-date preserves provenance, which is what the dataset
  card (`mod-112`) needs.
- **Not recording the re-assembly policy.** `mark` and `drop` have
  different downstream costs (training data noise vs audit opacity).
  The report must say which was chosen.
- **Treating targeted sentence dedup as a benchmark-decontamination
  pass.** It is not; `mod-106` does that. The caveat is mandatory.
- **Omitting the SHA of the benchmark n-gram list or the PII regex
  file.** A reader cannot reproduce what was in scope without the
  files' SHAs.
- **Only counting the removal rate at document granularity.** The
  report requires per-granularity, per-source removal rates. Per
  granularity because the three pipelines are three different
  interventions; per source because the dataset card consumer
  (`mod-112`) needs it.
- **No judgement call.** The exercise ends in a shippable
  recommendation. "Depends on the slice" without further argument is a
  review failure.

## 6. References

- **`datasketch` documentation** —
  <https://ekzhu.github.io/datasketch/>. Same `MinHashLSH` API as
  Exercise 02; the parameters change, not the surface.
- **`pysbd`** — <https://github.com/nipunsadvilkar/pySBD>. Sentence
  boundary disambiguation library used in many public pretraining
  pipelines; `clean=False` keeps normalisation in the Chapter 02
  canonicaliser.
- **Beautiful Soup** —
  <https://www.crummy.com/software/BeautifulSoup/bs4/doc/>. HTML-block
  parser for paragraph splitting.
- **`dolma`** — <https://github.com/allenai/dolma>. Practitioner
  reference for paragraph-level deduplication as the mid-tier recipe
  on Common Crawl. Pin the commit SHA of the dedup config you consult.
- **Lee, K. et al., "Deduplicating Training Data Makes Language Models
  Better", 2021 (NearDup)** — <https://arxiv.org/abs/2107.06499>.
  Document-level MinHash dedup is their contribution; they explicitly
  frame paragraph-level as a future extension.
- **Kandpal, N. et al., "Deduplicating Training Data Mitigates Privacy
  Risks in Language Models", 2022** —
  <https://arxiv.org/abs/2202.06539>. Cited for the
  boilerplate-memorisation claim.
- **OWASP Large Language Model Top 10** —
  <https://genai.owasp.org/llm-top-10/>. LLM02 Sensitive Information
  Disclosure (2025); LLM06 in 2023.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 Data Poisoning Attack is the anchor for why paragraph-level
  boilerplate is adversarially relevant.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. AML.T0048 Reconstruct
  Training Data is the adversary technique paragraph dedup weakens.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE 2.10
  is the control anchor.
- **Local anchors:** Chapter 05 (this exercise's primary content);
  Exercise 02 (document-level pipeline that feeds pipeline A); Exercise
  04 (production toolchain comparison); `mod-106` (benchmark
  decontamination — the home for the full-fidelity sentence-level
  decon work); `mod-110` (PII appearance counts).
