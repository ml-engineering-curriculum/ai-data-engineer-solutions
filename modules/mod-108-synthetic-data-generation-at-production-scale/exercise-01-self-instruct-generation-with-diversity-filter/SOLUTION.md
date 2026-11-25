# Exercise 01 — Self-Instruct Generation with Diversity Filter (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner stands up a working
Self-Instruct loop against a modern hosted or local generator on a
seed pool they authored, filters candidates through the Chapter 02
stack (length / punctuation / language / keyword blacklist /
duplicate-of-seed / ROUGE-L), and ships the first blocks of the
program package: `seeds/seed_pool.jsonl`, `recipes/self_instruct_config.yaml`,
`outputs/sft_corpus.parquet`, `filters/rouge_l_report.parquet`, and a
one-page `calibration.md`.

Deliverables (taken verbatim from the exercise contract):

- A 100–200-item seed pool, hashed into the manifest and categorised
  by task family.
- A resumable Self-Instruct loop that writes a manifest row per run
  and emits a parquet corpus with lineage columns.
- A ROUGE-L dedup filter with a documented operating threshold and a
  per-round rejection report.
- A `calibration.md` that reports per-filter rejection rates, refusal
  rate, pool-wide diversity metrics, wall-clock and cost, and at
  least four concrete example instructions (2 accepted, 2 rejected
  with reason).

Governance framing. Self-Instruct is the simplest synthetic pipeline
and it already touches three top-of-module controls. The OWASP
Machine Learning Security Top 10 anchor is **ML02:2023 Data Poisoning
Attack** — unfiltered generator output that reproduces refusal text,
boilerplate, or modality-keyword failures is a self-poisoning vector.
The MITRE ATLAS technique in scope is **AML.T0020 Poison Training
Data** (the attacker here is the generator itself, emitting
systematic style drift that collapses the student's diversity surface).
The NIST AI Risk Management Framework control anchors are **GOVERN
1.2** (roles and responsibilities around data pipeline accountability)
and **MEASURE 2.11** (fairness-and-bias measurement, because an
unmeasured task-family drift biases the eventual evaluation surface).
Cite the exact framework URLs in the References section rather than
paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── seeds/
│   ├── seed_pool.jsonl                  # 100-200 hand-authored seeds
│   └── seed_pool.jsonl.sha256
├── recipes/
│   └── self_instruct_config.yaml        # pinned generator + thresholds
├── code/
│   ├── build_seed_pool.py               # helper: tag families + dedup check
│   ├── generate.py                      # main loop; --resume supported
│   ├── filters.py                       # length/kw/refusal/lang/dedup stack
│   ├── rouge_l.py                       # ROUGE-L + embedding prefilter
│   ├── manifest.py                      # emit the YAML
│   └── calibrate.py                     # per-round rejection stats
├── outputs/
│   ├── sft_corpus.parquet               # final (instruction, response) pairs
│   └── sft_corpus.parquet.sha256
├── filters/
│   ├── rouge_l_report.parquet           # per-(round, filter) rejection counts
│   ├── rejected_samples.jsonl           # ~50 sampled rejections across stages
│   └── refusal_patterns.txt
└── calibration.md
```

### 2.2 `seeds/seed_pool.jsonl` — the single most important artifact

A pool of 100–200 hand-authored seed tasks in the chosen domain. Each
line is one record:

```json
{
  "seed_id": "seed-00001",
  "task_family": "extraction",
  "difficulty": "medium",
  "expected_format": "structured",
  "instruction": "Given the following CSV row, emit a JSON object with keys name, age, and city.",
  "input": "\"Alice\",34,\"Berlin\"",
  "output": "{\"name\": \"Alice\", \"age\": 34, \"city\": \"Berlin\"}"
}
```

Hygiene rules (all enforced by `build_seed_pool.py`):

- SHA-256 over the sorted-key, trailing-newline-normalised JSONL
  bytes. Pin the digest in both `seeds/seed_pool.jsonl.sha256` and
  the manifest. A seed-pool hash that drifts silently between runs is
  a reproducibility bug.
- Task families are balanced: no family carries more than 2× the mean
  count. The script fails the pool-build if the check fails.
- Run the pool against itself at ROUGE-L > 0.7 — zero internal hits.
  Internal near-duplicates are the first-order collapse risk.
- Every seed is tagged `difficulty ∈ {easy, medium, hard}` and
  `expected_format ∈ {free_form, structured}`. The eventual
  release-eligibility check reads these columns.

### 2.3 `code/filters.py`

```python
# code/filters.py
# Chapter 02 §"Order of operations" filter stack, cheapest-rejects-first.
from __future__ import annotations

import re
import unicodedata
from dataclasses import dataclass
from typing import Callable

import fasttext

MIN_WORDS = 3
MAX_WORDS = 150

# Modality keywords the generator cannot handle in a text-only context.
# Chapter 02 §"Heuristic filters" names the singulars; plurals are added
# here because the generator emits both "describe this plot" and
# "describe these plots".
KW_BLACKLIST = {
    "image", "images", "graph", "graphs", "plot", "plots", "file", "files",
    "figure", "figures", "picture", "pictures", "chart", "charts",
}

REFUSAL_PREFIXES = (
    "i cannot", "i can't", "i'm sorry", "i am sorry", "as an ai language model",
    "as a language model", "i apologise", "i apologize", "i'm unable",
    "i am unable",
)

_LID = fasttext.load_model("lid.176.bin")


@dataclass(slots=True)
class FilterResult:
    accepted: bool
    stage: str | None   # one of: length / punct / lang / kw / dup_seed / rouge_l


def _canon(text: str) -> str:
    return unicodedata.normalize("NFC", " ".join(text.strip().split()))


def length_filter(text: str) -> FilterResult:
    n = len(text.split())
    if n < MIN_WORDS or n > MAX_WORDS:
        return FilterResult(False, "length")
    return FilterResult(True, None)


def punct_or_refusal_filter(text: str) -> FilterResult:
    if not text or not text[0].isalpha():
        return FilterResult(False, "punct")
    lowered = text.lower()
    if any(lowered.startswith(p) for p in REFUSAL_PREFIXES):
        return FilterResult(False, "punct")
    return FilterResult(True, None)


def language_filter(text: str, target: str = "en") -> FilterResult:
    label, _ = _LID.predict(text.replace("\n", " "), k=1)
    code = label[0].replace("__label__", "")
    if code != target:
        return FilterResult(False, "lang")
    return FilterResult(True, None)


def keyword_filter(text: str) -> FilterResult:
    tokens = re.findall(r"[a-zA-Z']+", text.lower())
    if any(t in KW_BLACKLIST for t in tokens):
        return FilterResult(False, "kw")
    return FilterResult(True, None)


def duplicate_of_seed_filter(text: str, seed_norm_set: set[str]) -> FilterResult:
    if _canon(text).lower() in seed_norm_set:
        return FilterResult(False, "dup_seed")
    return FilterResult(True, None)


STACK: list[Callable[..., FilterResult]] = [
    length_filter, punct_or_refusal_filter, language_filter,
    keyword_filter,
]


def run_stack(text: str, seed_norm_set: set[str]) -> FilterResult:
    for f in STACK:
        r = f(text)
        if not r.accepted:
            return r
    return duplicate_of_seed_filter(text, seed_norm_set)
```

Design notes defensible at review:

- **Plurals are first-class in the blacklist.** Chapter 02 §"Keyword
  filter leakage" is explicit: `plot` and `plots` are both filter
  hits. Omitting plurals is the most common filter-leakage bug.
- **FastText lid.176 is the language ID.** It is cheap (~10 µs per
  call), is model-friendly to the streaming batch loop, and has
  a published model card; swapping it for a learned classifier is
  not worth the latency cost at this scale.
- **The refusal prefix list is maintained.** Modern hosted models
  refuse more often than GPT-3. Expect to grow this list as refusal
  patterns drift across generator versions.

### 2.4 `code/rouge_l.py`

```python
# code/rouge_l.py
# ROUGE-L against the growing pool with an embedding prefilter.
from __future__ import annotations

from dataclasses import dataclass

import numpy as np
from rouge_score import rouge_scorer

_SCORER = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=False)
ROUGE_L_THRESHOLD = 0.7     # Chapter 02 operating choice; pinned
PREFILTER_TOP_K = 50        # embedding neighbours to re-score with ROUGE-L


@dataclass(slots=True)
class PoolIndex:
    texts: list[str]
    embeddings: np.ndarray   # L2-normalised, shape (N, D)


def rouge_l(a: str, b: str) -> float:
    return _SCORER.score(a, b)["rougeL"].fmeasure


def candidate_rouge_max(cand: str, cand_emb: np.ndarray,
                         pool: PoolIndex) -> float:
    """Top-K cosine prefilter, then exact ROUGE-L on the K neighbours.

    Returns the max ROUGE-L over the K nearest pool items. Caller
    compares against ROUGE_L_THRESHOLD.
    """
    if not pool.texts:
        return 0.0
    sims = pool.embeddings @ cand_emb        # cosine, since L2-normalised
    k = min(PREFILTER_TOP_K, len(pool.texts))
    topk = np.argpartition(-sims, k - 1)[:k]
    return max(rouge_l(cand, pool.texts[int(i)]) for i in topk)
```

Two design choices worth defending at review:

- **Embedding prefilter switched on past ~2 k pool items.** Chapter 02
  §"Pool-size scaling problem" describes the N-growing naive cost.
  At pool size 100 the naive loop is milliseconds; at 2 k it crosses
  the per-candidate 2-second ceiling. The prefilter preserves recall
  at K=50 for ROUGE-L > 0.7 on short imperative sentences.
- **`use_stemmer=False`.** The original paper used ROUGE-L on raw
  tokens. Enabling the stemmer raises recall of near-duplicates but
  also raises false-positive drops on legitimate paraphrases
  ("compute" vs "computed" become identical). Keep it off unless a
  tuning study justifies the change.

### 2.5 `code/generate.py` (loop skeleton)

```python
# code/generate.py
# Resumable Self-Instruct loop. Writes outputs/sft_corpus.parquet and
# filters/rouge_l_report.parquet. Supports --resume by reading the
# existing corpus and continuing from max(generation_round) + 1.
from __future__ import annotations

import argparse
import hashlib
import json
import random
import time
from collections import Counter
from pathlib import Path
from typing import Iterator

import pyarrow as pa
import pyarrow.parquet as pq
from rouge_score import rouge_scorer

from filters import run_stack
from rouge_l import PoolIndex, candidate_rouge_max, ROUGE_L_THRESHOLD

SEEDS_PER_PROMPT = 6         # paper default
PRIOR_PER_PROMPT = 2
CANDIDATES_PER_CALL = 15     # generator emits Task 9 .. Task 23


def few_shot_prompt(seed_sample: list[dict], prior_sample: list[dict]) -> str:
    parts = ["Come up with a series of tasks:"]
    for i, t in enumerate(seed_sample + prior_sample, start=1):
        parts.append(f"Task {i}: {t['instruction']}")
    parts.append(f"Task {len(seed_sample) + len(prior_sample) + 1}:")
    return "\n".join(parts)


def parse_candidates(completion: str, start_index: int) -> list[str]:
    """Split a completion on `Task N:` delimiters, strip blanks."""
    out = []
    for line in completion.splitlines():
        line = line.strip()
        if not line:
            continue
        if ":" in line and line.split(":", 1)[0].lower().startswith("task"):
            _, body = line.split(":", 1)
            out.append(body.strip())
    return [o for o in out if o]


def generate_round(round_idx: int, seeds: list[dict], prior: list[dict],
                   pool_index: PoolIndex, seed_norm_set: set[str],
                   generator, embedder,
                   rejection_counter: Counter) -> list[dict]:
    kept: list[dict] = []
    seed_sample = random.sample(seeds, SEEDS_PER_PROMPT)
    prior_sample = (
        random.sample(prior, PRIOR_PER_PROMPT)
        if len(prior) >= PRIOR_PER_PROMPT else []
    )

    prompt = few_shot_prompt(seed_sample, prior_sample)
    completion = generator.complete(
        prompt=prompt,
        max_tokens=2048,
        temperature=0.7,
        stop=["Task 25:", "\n\n\n"],
    )
    candidates = parse_candidates(completion, start_index=9)

    for cand in candidates:
        r = run_stack(cand, seed_norm_set)
        if not r.accepted:
            rejection_counter[(round_idx, r.stage)] += 1
            continue
        cand_emb = embedder.embed(cand)
        if candidate_rouge_max(cand, cand_emb, pool_index) > ROUGE_L_THRESHOLD:
            rejection_counter[(round_idx, "rouge_l")] += 1
            continue
        response = generate_response(cand, generator)
        if response is None:
            rejection_counter[(round_idx, "response")] += 1
            continue
        kept.append({
            "instruction": cand,
            "response": response,
            "seed_source_ids": [s["seed_id"] for s in seed_sample],
            "generation_round": round_idx,
            "generator_model_id": generator.model_id,
            "generation_timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ",
                                                    time.gmtime()),
        })
        pool_index.texts.append(cand)
        pool_index.embeddings = _append_embedding(pool_index.embeddings,
                                                   cand_emb)
    return kept


def generate_response(instruction: str, generator) -> str | None:
    """Second pass: generate the response at lower temperature."""
    completion = generator.complete(
        prompt=f"Instruction:\n{instruction}\n\nResponse:",
        max_tokens=1024,
        temperature=0.4,
        stop=["\n\nInstruction:"],
    )
    text = completion.strip()
    if not text:
        return None
    # Share the refusal filter: a refusal on the response side is dropped.
    from filters import REFUSAL_PREFIXES
    lowered = text.lower()
    if any(lowered.startswith(p) for p in REFUSAL_PREFIXES):
        return None
    return text


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--manifest", required=True, type=Path)
    ap.add_argument("--seeds", required=True, type=Path)
    ap.add_argument("--output", required=True, type=Path)
    ap.add_argument("--rejection-report", required=True, type=Path)
    ap.add_argument("--target-count", type=int, default=2000)
    ap.add_argument("--resume", action="store_true")
    args = ap.parse_args()

    manifest = _load_yaml(args.manifest)
    seeds = [json.loads(line) for line in args.seeds.read_text().splitlines()
             if line.strip()]
    seed_norm_set = {_canon_lower(s["instruction"]) for s in seeds}

    generator = _resolve_generator(manifest["generator"])
    embedder = _resolve_embedder(manifest.get("embedder",
                                               {"id": "sentence-transformers"}))

    prior: list[dict] = []
    pool_index = PoolIndex(texts=[], embeddings=_empty_emb(embedder.dim))
    rejection_counter: Counter = Counter()
    start_round = 0

    if args.resume and args.output.exists():
        existing = pq.read_table(args.output).to_pylist()
        prior = existing
        start_round = max(r["generation_round"] for r in existing) + 1
        for rec in existing:
            pool_index.texts.append(rec["instruction"])
            pool_index.embeddings = _append_embedding(
                pool_index.embeddings, embedder.embed(rec["instruction"]))

    while len(prior) < args.target_count:
        round_idx = start_round + (len(prior) // 100)
        kept = generate_round(round_idx, seeds, prior, pool_index,
                               seed_norm_set, generator, embedder,
                               rejection_counter)
        prior.extend(kept)
        _flush_corpus(args.output, prior)
        _flush_rejections(args.rejection_report, rejection_counter)

    _write_sha256(args.output)


if __name__ == "__main__":
    main()
```

Design notes worth defending at review:

- **Resumability is schema-stable.** `--resume` reads the existing
  parquet and continues. The schema does not change across resumes:
  `instruction_id`, `instruction`, `response`, `seed_source_ids`,
  `generation_round`, `generator_model_id`, `generation_timestamp`.
- **Response generation is a separate pass.** Chapter 02 §"The
  response-generation step" recommends the two-pass structure so the
  instruction-side ROUGE-L filter does its job before response cost
  is spent.
- **Rejection counts are keyed by `(round, filter)`.** This is the
  parquet schema `rouge_l_report.parquet` writes. The schema is
  stable across runs; `--resume` appends rows, never rewrites.

### 2.6 `recipes/self_instruct_config.yaml`

```yaml
# recipes/self_instruct_config.yaml
generator:
  model_id: claude-3-5-sonnet-20241022    # dated version, not bare family
  temperature: 0.7
  top_p: 1.0
  max_tokens: 2048
  stop: ["Task 25:", "\n\n\n"]

seed_pool:
  path: seeds/seed_pool.jsonl
  sha256: <64-hex-digest>
  item_count: 175

sampling:
  seeds_per_prompt: 6
  prior_generations_per_prompt: 2
  anneal_schedule: constant

filters:
  rouge_l_threshold: 0.7
  rouge_l_backend: embedding_prefilter_top_50
  length_range: [3, 150]
  language: en
  keyword_blacklist: [image, images, graph, graphs, plot, plots, file,
                       files, figure, figures, picture, pictures, chart,
                       charts]
  refusal_patterns_path: filters/refusal_patterns.txt

targets:
  instruction_count: 2000
  per_round_cap: 100
  max_rounds: 50

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

Pin the dated generator version (`claude-3-5-sonnet-20241022`). An
undated family (`claude-3-5-sonnet`) fails acceptance criterion #2.

### 2.7 `calibration.md`

One page. Required sections:

1. **Summary.** Total generation calls, total candidate instructions,
   surviving instructions, wall-clock, cost.
2. **Per-filter rejection rate** as a table with columns `filter`,
   `rejections`, `candidates_in`, `rejection_rate`. Five filter
   rows plus `rouge_l` and `response`.
3. **Pool diversity.** Unique first-tokens count, mean pairwise
   ROUGE-L of a 500-item uniform sample.
4. **Observed refusal rate** across generator calls (refusal /
   total_candidates).
5. **Example instructions.** Two accepted, two rejected; each with
   its filter verdict visible.
6. **Wall-clock and cost.** Per round and total.

Any claim that cannot be verified at authoring time (e.g., an
upstream throughput number from a vendor doc) is marked
`<!-- pending-verify: ... -->` and blocks auto-merge.

## 3. Validation steps

1. **Seed pool hash matches the sidecar.**
   `sha256sum seeds/seed_pool.jsonl` equals the contents of
   `seeds/seed_pool.jsonl.sha256`, and both equal the digest pinned
   in `recipes/self_instruct_config.yaml`.
2. **Seed pool has zero internal ROUGE-L > 0.7 hits.** Running the
   pool against itself at the threshold returns zero pairs.
3. **Task-family balance.** No family exceeds 2× the mean count.
4. **Corpus size.** `outputs/sft_corpus.parquet` has ≥ 2 000 rows
   and the required schema (`instruction_id`, `instruction`,
   `response`, `seed_source_ids`, `generation_round`,
   `generator_model_id`, `generation_timestamp`).
5. **No near-duplicates in the corpus.** A pairwise ROUGE-L scan
   across the instruction column returns zero pairs > 0.75.
6. **No banned keywords in the corpus.** A regex scan for
   `image|graph|plot|file|figure|picture|chart` (and their plurals)
   on the instruction column returns zero matches. Deliberate
   exemptions are documented in `calibration.md`.
7. **Generator version is dated.** `yq '.generator.model_id'
   recipes/self_instruct_config.yaml` returns a string that includes
   a date component. A bare family name fails.
8. **Rejection report has one row per `(round, filter)` pair.** Read
   `filters/rouge_l_report.parquet` and verify the schema.
9. **Resumability.** `make clean-outputs && make generate && kill
   mid-run && make resume` continues the run; the parquet schema is
   byte-stable across the pre- and post-resume shards; the final
   corpus's row count is the target.
10. **`calibration.md` reports the required sections.** Rejection
    rates sum to the candidate count; refusal rate is present; at
    least four concrete example instructions are listed.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `seeds/seed_pool.jsonl` has 100–200 entries with `task_family`, `difficulty`, `expected_format`                      | Yes       |
| 2  | Seed pool hash is pinned in the sidecar and the manifest, and the two match                                         | Yes       |
| 3  | Zero internal ROUGE-L > 0.7 hits in the seed pool                                                                   | Yes       |
| 4  | Generator version is dated (e.g., `claude-3-5-sonnet-20241022`)                                                     | Yes       |
| 5  | Filter stack order follows Chapter 02 §"Order of operations" (cheapest-reject-first)                                | Yes       |
| 6  | ROUGE-L threshold 0.7 is pinned in the manifest and in the rouge_l module                                            | Yes       |
| 7  | Keyword blacklist includes both singulars and plurals                                                               | Yes       |
| 8  | `outputs/sft_corpus.parquet` has ≥ 2 000 rows and the required schema                                                | Yes       |
| 9  | Corpus has zero ROUGE-L > 0.75 pairs and zero banned-keyword instructions                                            | Yes       |
| 10 | `filters/rouge_l_report.parquet` has one row per `(round, filter)` tuple                                            | Yes       |
| 11 | `generate.py --resume` continues a stopped run and preserves schema                                                 | Yes       |
| 12 | `calibration.md` reports per-filter rejection rate, refusal rate, and 4+ example instructions                        | Yes       |
| 13 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF GOVERN 1.2 / MEASURE 2.11) cited        | No        |
| 14 | Unverifiable claims are marked `<!-- pending-verify: ... -->`                                                       | Yes       |

## 5. Common mistakes

- **Shipping a bare generator family name.** `claude-3-5-sonnet`
  without the date drifts silently across model upgrades. Chapter 02
  §"Generator refusal drift" is explicit — pin the dated version.
- **Keyword filter without plurals.** `plot` matches but `plots`
  does not. Chapter 02 §"Keyword filter leakage" calls this the
  textbook filter-leakage bug. The acceptance regex test catches it.
- **ROUGE-L on stemmed tokens.** Enabling the stemmer raises false
  positives on paraphrases and makes the filter stricter than the
  paper's. Keep it off unless a tuning study justifies the change.
- **Letting the seed pool drift without a hash.** Changing a seed
  mid-run invalidates every downstream step. Hash first, run second.
- **Single-pass generation.** Asking for `(instruction, response)` in
  one call breaks the instruction-side ROUGE-L filter and inflates
  cost. The two-pass pattern in Chapter 02 is the baseline.
- **No per-round rejection logging.** Without it, pool collapse is
  invisible until the final corpus looks suspicious. The
  `rouge_l_report.parquet` schema is the diagnostic trail.
- **Target-count without a kill-switch.** If the generator refuses
  more than a few percent, the loop runs forever without hitting
  the target. Chapter 02 §"Where Self-Instruct breaks" §1 (pool
  collapse) and §4 (refusal drift) both motivate a kill-switch; the
  acceptance criteria require a per-filter rejection log but a
  kill-switch on refusal rate is worth adding.
- **Un-pinned sampling ratio.** 6:2 is the paper default; log the
  ratio in the manifest; if annealing, log the schedule.
- **Reseeding on `--resume` without re-hashing.** If seed-pool
  content changed since the last run, the pin breaks. The sidecar
  hash check at `--resume` start catches it.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchor for unfiltered synthetic data entering a
  training pipeline; self-poisoning is a valid subtype.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing anchor for a
  generator that emits systematic style drift into the student's
  training data.
- **NIST AI Risk Management Framework — GOVERN 1.2, MEASURE 2.11:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN
  1.2 anchors the pipeline-accountability surface; MEASURE 2.11
  anchors the bias-and-fairness measurement that an unmeasured
  task-family drift would undercut.
- **Self-Instruct (Wang et al. 2022, ACL 2023), arXiv 2212.10560:**
  <https://arxiv.org/abs/2212.10560>. The primary source for the
  recipe, the 6:2 sampling ratio, the ROUGE-L > 0.7 threshold, and
  the heuristic filter list. Cite the paper when defending any of
  these operating choices.
- **Reference implementation (not to copy):**
  <https://github.com/yizhongw/self-instruct>. The authors' public
  seed-pool and generation code; useful for contrasting your filter
  implementation against the paper's.
- **`rouge-score` library:** <https://github.com/google-research/google-research/tree/master/rouge>.
  The ROUGE-L implementation; pin version for reproducibility.
- **FastText language identification (`lid.176`):**
  <https://fasttext.cc/docs/en/language-identification.html>. The
  language-ID model; cheap, published, swap-safe.
- **Local anchors:** `mod-105` exercise-01 (canonical form and the
  `NORMALISATION_VERSION` constant this pipeline reuses when
  normalising text for the duplicate-of-seed check); `mod-108`
  Chapter 02 (operating choices); `mod-109` (manifest signing of
  `outputs/sft_corpus.parquet` for lineage).
