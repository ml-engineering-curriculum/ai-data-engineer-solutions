# Exercise 03 — Dolma or RedPajama-Data v2 Pipeline Reproduction (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner runs a scoped subset of a
published reference pipeline (Dolma `cc` or RedPajama-Data v2 signal
computation) against 2–3 WARC files, using the published toolkit with
published parameters, and writes a comparison report against the published
reference's aggregate statistics.

The exercise is **not** about rebuilding the corpus. It is about using the
published operators as-is so the learner internalises:

- how the reference pipeline stages its work,
- which parameters are hard-coded in the published config,
- and where their small slice behaves differently from the published
  aggregate — and why.

The worked answer below walks **Path A (Dolma)** end-to-end because the
Dolma toolkit is YAML-driven and more faithful to "read the published
config, run it". **Path B (RedPajama-Data v2)** is covered in §2.7 with the
material differences called out. Either is a passing solution.

Governance framing: this pipeline, regardless of which path is chosen,
feeds training data. The acquire-stage opt-out check from Chapter 06 runs
*before* any downstream operator, mapping to NIST AI RMF `MAP 4.1` and
`MEASURE 2.10`; ignoring it reproduces exactly the posture OWASP ML Top 10
**ML02 Data Poisoning Attack** warns against.

## 2. Implementation — worked answer (Path A: Dolma)

### 2.1 Project layout

```
exercise-03/
├── README.md
├── inputs.yaml
├── requirements.lock
├── configs/
│   ├── tagging_config.yaml
│   ├── mixing_config.yaml
│   └── dedup_config.yaml
├── acquire_evidence/              # Chapter-06 decisions, one dir per host
├── code/
│   ├── 01_pin_warcs.py
│   ├── 02_acquire_check.py
│   ├── 03_run_dolma.sh
│   ├── 04_compute_stats.py
│   └── 05_build_comparison.py
├── runs/
│   └── run_YYYY-MM-DD/
│       ├── documents/             # Dolma input shards (JSONL.gz)
│       ├── attributes/            # Dolma per-tagger attribute files
│       └── logs/
├── stats/
│   ├── your_stats.yaml
│   └── reference_stats.yaml
└── comparison_report.md
```

### 2.2 `inputs.yaml` — pin the slice

```yaml
# inputs.yaml
pipeline_path: "dolma"
dolma_version: "1.0.x"                # resolved from requirements.lock
snapshot_id: "CC-MAIN-2024-30"
selection_predicate_sha256: "<sha256 of selection_predicate.sql>"
warcs:
  - key: "crawl-data/CC-MAIN-2024-30/segments/.../warc/CC-MAIN-20240730xxx.warc.gz"
    observed_sha256: "<fill at fetch time>"
    bytes: 1234567890
  - key: "crawl-data/CC-MAIN-2024-30/segments/.../warc/CC-MAIN-20240730yyy.warc.gz"
    observed_sha256: "<fill at fetch time>"
    bytes: 1200000000
  - key: "crawl-data/CC-MAIN-2024-30/segments/.../warc/CC-MAIN-20240730zzz.warc.gz"
    observed_sha256: "<fill at fetch time>"
    bytes: 1180000000
notes:
  bucket: "commoncrawl (public)"
  acquire_stage_evidence_dir: "acquire_evidence/"
  fail_open_on_missing_robots_txt: true
```

`observed_sha256` is captured by `01_pin_warcs.py` after downloading; it is
how a reviewer confirms they ran against the same bytes.

### 2.3 `code/01_pin_warcs.py`

```python
# code/01_pin_warcs.py — download three WARCs and record their SHA-256s.
import hashlib
from pathlib import Path

import boto3
import yaml

INPUTS = Path("inputs.yaml")
LOCAL = Path("runs/_inputs")

def main() -> None:
    cfg = yaml.safe_load(INPUTS.read_text())
    s3 = boto3.client("s3")  # unsigned read works for the public CC bucket;
                             # configure via the standard AWS env / profile
    LOCAL.mkdir(parents=True, exist_ok=True)
    for entry in cfg["warcs"]:
        dst = LOCAL / Path(entry["key"]).name
        if not dst.exists():
            s3.download_file("commoncrawl", entry["key"], str(dst))
        h = hashlib.sha256(dst.read_bytes()).hexdigest()
        entry["observed_sha256"] = h
        entry["bytes"] = dst.stat().st_size
    INPUTS.write_text(yaml.safe_dump(cfg, sort_keys=False))

if __name__ == "__main__":
    main()
```

### 2.4 Acquire-stage check (Chapter 06) — reuse Exercise 01

The acquire-stage check must run **before** any Dolma stage sees the
content. The implementation is the same as Exercise 01, operating on URLs
extracted from each WARC. The output is a per-URL decision file under
`acquire_evidence/`, keyed by host; URLs with `acquire_allowed=false` are
dropped from the Dolma input stage.

Why layer it on? Dolma does not enforce AI-use opt-outs by default (verify
against the current Dolma release); honouring the opt-outs is a project
responsibility. The published Dolma recipe filters URLs through
`url_blocklist` + `cc_blocklist` only — compliance-grade opt-out honouring
is a bolt-on, not a toolkit feature.

### 2.5 `configs/tagging_config.yaml` — the operator chain

The Dolma toolkit is a "tagger + mixer" model: taggers compute per-document
attributes; the mixer applies keep/drop rules against those attributes. The
published configs for Dolma v1 live under `configs/` in the
`allenai/dolma` repo; pin the file + commit SHA you're starting from.

```yaml
# configs/tagging_config.yaml
documents:
  - "runs/run_YYYY-MM-DD/documents/*.jsonl.gz"

taggers:
  - cc_re                              # URL filter (regex blocklists)
  - domain_blocklist                   # host-level blocklist
  - gopher_v2                          # Gopher-style quality signals
  - fasttext_lid_v3                    # language-ID
  - jigsaw_hatespeech_sentence_v2     # safety — comment out if skipped
  - jigsaw_nsfw_sentence_v2           # NSFW — comment out if skipped
  - pii_detection_v1                  # PII attributes (not redaction)
  # - decontamination_<name>          # benchmark decontam; configured below
processes: 16
work_dir:
  input: "runs/run_YYYY-MM-DD/documents"
  output: "runs/run_YYYY-MM-DD/attributes"
```

```yaml
# configs/mixing_config.yaml
streams:
  - name: cc_en_v1
    documents:
      - "runs/run_YYYY-MM-DD/documents/*.jsonl.gz"
    attributes:
      - cc_re
      - domain_blocklist
      - gopher_v2
      - fasttext_lid_v3
    output:
      path: "runs/run_YYYY-MM-DD/mixed/cc_en_v1"
      max_size_in_bytes: 1_073_741_824
    filter:
      include:
        - "$.attributes.fasttext_lid_v3__en[0].score >= 0.5"
      exclude:
        - "$.attributes.gopher_v2__repetition_ratio[0] >= 0.3"
        - "$.attributes.gopher_v2__mean_word_length[0] < 3"
        - "$.attributes.gopher_v2__mean_word_length[0] > 10"
        - "$.attributes.cc_re__blocked[0] == 1"
        - "$.attributes.domain_blocklist__blocked[0] == 1"
```

Rules of thumb that the learner must document:

- **Every tagger named above must exist at the pinned `dolma` version.** Some
  taggers drift (names change, versions are appended). Mark any that are
  missing at your pinned version with `<!-- unverified: ... -->` and
  substitute the nearest equivalent, naming the substitution.
- **The Gopher rules above are the ones Rae et al. 2021 describe.** Dolma
  implements them but exposes the thresholds via config; the thresholds in
  the mixing config must match the paper or be flagged as a deliberate
  deviation.
- **Benchmark decontamination** (Dolma's decontam tagger) runs against a
  pinned list of benchmarks. For this exercise, cite the published Dolma
  decontam target list and either reuse it as-is or justify any deletions.

### 2.6 `configs/dedup_config.yaml` — MinHash

```yaml
# configs/dedup_config.yaml — document-level MinHash (near-dedup)
documents:
  - "runs/run_YYYY-MM-DD/mixed/cc_en_v1/*.jsonl.gz"
dedupe:
  name: minhash_lsh_v1
  skip_empty: true
  paragraphs: false
  # The published Dolma config uses these (verify at your pinned version):
  num_perm: 128
  threshold: 0.8                  # Jaccard; paper reports 0.8 for cc
  ngram_size: 5
  seed: 1
output_dir: "runs/run_YYYY-MM-DD/deduped"
processes: 16
```

### 2.7 Path B — RedPajama-Data v2 differences

The RedPajama-Data v2 pipeline is script-driven. Replace §2.5 / §2.6 with:

```
# In RedPajama-Data checkout:
#   pretraining/CC-v2/
#     1-wet2json.py                  # produces documents.jsonl.gz
#     2-cc_net.py                    # perplexity signal, language-ID
#     3-quality_signals.py           # ~20 rps_doc_* signals
#     4-minhash.py                   # MinHash sigs at Jaccard {0.7, 0.8, 0.9}
# You keep your acquire-stage check as step 0, then run 1→4 on the WARCs
# you pinned in inputs.yaml.
```

Key behavioural difference to capture in the comparison report: RedPajama-v2
**emits signals for every input document**; it does not filter. The
"reproduction" is of signal correctness, not corpus construction. The
comparison then compares the **signal distributions** (e.g. `rps_doc_frac_lines_end_with_ellipsis`)
to the published distributions, not the filtered counts.

### 2.8 `code/04_compute_stats.py` — per-stage statistics

```python
# code/04_compute_stats.py
import gzip
import json
from pathlib import Path

import polars as pl
import yaml

RUN = Path("runs/run_YYYY-MM-DD")
OUT = Path("stats/your_stats.yaml")

def count_jsonl(glob: str) -> int:
    n = 0
    for p in sorted(RUN.glob(glob)):
        with gzip.open(p, "rt", encoding="utf-8") as fh:
            n += sum(1 for _ in fh)
    return n

def tok_count(glob: str) -> int:
    # Approximation: whitespace-split tokens. The real Dolma stats use GPT-2
    # BPE; cite that discrepancy in the comparison report.
    total = 0
    for p in sorted(RUN.glob(glob)):
        with gzip.open(p, "rt", encoding="utf-8") as fh:
            for line in fh:
                doc = json.loads(line)
                total += len((doc.get("text") or "").split())
    return total

def main() -> None:
    stats = {
        "pipeline": "dolma",
        "slice": "ex-03",
        "stages": {
            "input": {
                "docs": count_jsonl("documents/*.jsonl.gz"),
                "approx_whitespace_tokens": tok_count("documents/*.jsonl.gz"),
            },
            "mixed_cc_en_v1": {
                "docs": count_jsonl("mixed/cc_en_v1/*.jsonl.gz"),
                "approx_whitespace_tokens": tok_count("mixed/cc_en_v1/*.jsonl.gz"),
            },
            "deduped": {
                "docs": count_jsonl("deduped/*.jsonl.gz"),
                "approx_whitespace_tokens": tok_count("deduped/*.jsonl.gz"),
            },
        },
    }
    inp = stats["stages"]["input"]["docs"]
    mix = stats["stages"]["mixed_cc_en_v1"]["docs"]
    ded = stats["stages"]["deduped"]["docs"]
    stats["ratios"] = {
        "mix_keep_fraction": None if inp == 0 else mix / inp,
        "dedup_keep_fraction": None if mix == 0 else ded / mix,
    }
    OUT.write_text(yaml.safe_dump(stats, sort_keys=False))

if __name__ == "__main__":
    main()
```

### 2.9 `stats/reference_stats.yaml` — published values

The published reference is the Dolma paper (Soldaini et al., "Dolma: An Open
Corpus of Three Trillion Tokens..."); some values also live in the dataset
card on Hugging Face. Record only the numbers you can cite to a specific
section, and leave the rest `<!-- unverified: ... -->`.

```yaml
# stats/reference_stats.yaml
reference: "Dolma paper (Soldaini et al., 2024) — https://arxiv.org/abs/2402.00159"
stages:
  input:
    source: "Common Crawl WET + WARC subset"
    docs: "<!-- unverified: cite paper section -->"
  after_url_filter_and_quality:
    docs: "<!-- unverified: cite paper section -->"
  after_minhash:
    keep_fraction: "<!-- unverified: cite paper section -->"
notes:
  - "Dolma reports token counts using its tokenizer; your stats use whitespace
    tokens. Cite the discrepancy in the comparison."
  - "The paper reports aggregate numbers over the full CC subset (multi-TB);
    compare by ratios, not absolutes."
```

### 2.10 `comparison_report.md` — worked skeleton

```markdown
# Dolma `cc` reproduction — comparison report

## Pinned inputs

- Snapshot: CC-MAIN-2024-30
- WARCs: 3 (see `inputs.yaml`, with observed SHA-256s)
- Dolma version: 1.0.x (see `requirements.lock`)
- Acquire-stage evidence: `acquire_evidence/` (host-keyed, SHA-256'd robots.txt)

## Ratios comparison

| Stage                                   | Your ratio | Ref ratio      | Delta | Plausible explanation                                                          |
|-----------------------------------------|-----------:|---------------:|------:|--------------------------------------------------------------------------------|
| URL + domain blocklist keep fraction    | <fill>     | <ref>          | <δ>   | Our slice is TLD-restricted (`.com/.net/.org/.edu/.gov`), skewing blocklist hits |
| Language-ID (en, ≥0.5) keep fraction    | <fill>     | <ref>          | <δ>   | Our slice is already pre-filtered on declared language in the CC index          |
| Gopher-style quality keep fraction      | <fill>     | <ref>          | <δ>   | Small-sample noise; Gopher thresholds unchanged                                 |
| MinHash dedup keep fraction (0.8)       | <fill>     | <ref>          | <δ>   | Three WARCs give limited cross-doc duplicate exposure vs full snapshot          |

## Qualitative observations

- Operators present at pinned Dolma version but absent from the paper config:
  `<...>` — these are post-paper additions (verify against the repo CHANGELOG).
- Operators cited in the paper but renamed at pinned version: `<...>`.
- Acquire-stage opt-out rejected ~<N>% of candidate URLs — this slice sees
  opt-outs at a rate the published Dolma corpus does not (Dolma pre-dates
  wide `GPTBot`/`Google-Extended` adoption; the deltas are expected to grow
  over time).
```

## 3. Validation steps

1. **Toolkit is actually used.** `code/03_run_dolma.sh` invokes the published
   Dolma CLI, not a custom Python rewrite. The point of the exercise is to
   run the published operator chain.
2. **Parameters are traceable.** Each non-default value in the YAML configs
   is either present in the published Dolma config at the pinned commit SHA,
   or carries a comment naming the deliberate deviation.
3. **Acquire-stage check runs first.** The timestamps on
   `acquire_evidence/*` predate the Dolma stage logs in `runs/.../logs/`.
   Reviewers can diff timestamps to confirm order.
4. **Every WARC is pinned by SHA-256.** `inputs.yaml` has a non-empty
   `observed_sha256` for each entry.
5. **At least four stage-level ratios in the comparison, each cited.** The
   published reference citation is a section number of the paper or a URL
   to the dataset card, not a general "see Dolma paper".
6. **`plausible explanation` is actually plausible.** "unknown" is only
   acceptable when it is followed by a `<!-- unverified: ... -->` note
   naming what would resolve it.
7. **Token comparison discloses the tokenizer discrepancy.** If your stats
   use whitespace tokens but the paper uses a BPE tokenizer, the report
   says so; otherwise the delta is uninterpretable.
8. **Scaling disclaimer present.** The comparison is about ratios, not
   absolutes. A paragraph in the report says so and why.
9. **Reproduction instructions actually work.** `make reproduce` (or the
   documented equivalent) re-runs stages 0 → N on a fresh checkout.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
Pass at ≥ 20/30 with zero zeros on **critical** rows.

| # | Dimension                                                                                    | Critical? |
|---|----------------------------------------------------------------------------------------------|-----------|
| 1 | Chosen path (A or B) is documented with a one-paragraph justification                        | Yes       |
| 2 | WARC inputs are pinned by SHA-256 in `inputs.yaml`                                           | Yes       |
| 3 | Published toolkit is used end-to-end; no reimplemented operators                             | Yes       |
| 4 | Toolkit version is pinned in `requirements.lock`                                             | Yes       |
| 5 | All toolkit configs committed under `configs/`; parameters are traceable to the reference    | Yes       |
| 6 | Acquire-stage opt-out check runs **before** any downstream operator                          | Yes       |
| 7 | Per-stage statistics captured (docs in/out, drop fraction, wall-clock)                        | Yes       |
| 8 | Comparison report contains ≥4 stage-level ratios with citations                              | Yes       |
| 9 | "Plausible explanation" column is substantive, not "unknown"                                 | Yes       |
| 10 | Ratios, not absolutes, drive the comparison                                                  | Yes       |
| 11 | Tokenizer / tokenization discrepancy, if any, is disclosed                                   | No        |
| 12 | Governance anchors (OWASP ML Top 10 / NIST AI RMF) are cited for the acquire-stage policy    | No        |
| 13 | Unverifiable claims use `<!-- unverified: ... -->`                                       | Yes       |
| 14 | `make reproduce` (or equivalent) actually works on a fresh checkout                          | Yes       |
| 15 | For Path B: `rps_doc_*` signal distributions compared to published distributions              | No        |

## 5. Common mistakes

- **Reimplementing operators from the paper instead of using the toolkit.**
  The exercise's point is to *use* the published operator chain, warts and
  all. A from-scratch rewrite answers a different question.
- **Comparing absolute counts.** Your slice is 0.001% of the published
  corpus; absolute counts tell you nothing. Ratios and distributions do.
- **Running without the acquire-stage check.** Dolma's URL filter is not
  an AI-use opt-out honoring stage; stacking the two is the project's
  responsibility.
- **Pinning `dolma` by latest.** `dolma==latest` will drift. Pin a version
  and record it in `requirements.lock`.
- **Mixing paths (part Dolma, part RedPajama-v2).** Pick one. The two
  pipelines have different I/O shapes and you will end up with neither
  reproducibly reproduced.
- **Treating Dolma's decontam as "benchmark decontam, done".** Dolma's
  decontam targets a specific benchmark list from its paper; your project's
  internal benchmarks may not be covered. The gap is mod-106 territory and
  must be named in the report.
- **No deviations column.** A clean reproduction is rare; what the report
  really documents is deviations and their causes. Zero deviations
  reported almost always means one was missed.
- **"We don't know" with no follow-up.** Deltas without plausible
  explanations become folklore. Either explain it or `<!-- unverified -->`
  it so the audit knows it's open.

## 6. References

- **Dolma** — Soldaini et al., "Dolma: An Open Corpus of Three Trillion
  Tokens for Language Model Pretraining Research" (2024),
  <https://arxiv.org/abs/2402.00159>. Toolkit:
  <https://github.com/allenai/dolma>. Dataset:
  <https://huggingface.co/datasets/allenai/dolma>.
- **RedPajama-Data v2** — Together AI blog and repo,
  <https://www.together.ai/blog/redpajama-data-v2>,
  <https://github.com/togethercomputer/RedPajama-Data>. Dataset:
  <https://huggingface.co/datasets/togethercomputer/RedPajama-Data-V2>.
- **CCNet** — Wenzek et al., "CCNet: Extracting High Quality Monolingual
  Datasets from Web Crawl Data" (2019),
  <https://arxiv.org/abs/1911.00359>. The pipeline shape Dolma and
  RedPajama-Data both build on.
- **Gopher data-quality rules** — Rae et al., "Scaling Language Models:
  Methods, Analysis & Insights from Training Gopher" (2021),
  <https://arxiv.org/abs/2112.11446>, §A1.
- **DataComp-LM** (ablation framing for the stretch goal) — Li et al.
  (2024), <https://arxiv.org/abs/2406.11794>.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02
  grounds the "no acquire-stage check" risk.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MAP 4.1` and
  `MEASURE 2.10` are the governance anchors for the acquire-stage policy.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. Browse the matrix for the
  current technique IDs under the "Poison Training Data" tactic at run
  time.
- **Local anchors:** Chapter 06 for the Chapter-06 acquire-stage check
  reused here; Chapter 07 for the slice manifest shape; mod-106 for
  benchmark decontamination beyond Dolma's built-in target list; mod-109
  for Sigstore signing and SLSA attestations (stretch goal).
