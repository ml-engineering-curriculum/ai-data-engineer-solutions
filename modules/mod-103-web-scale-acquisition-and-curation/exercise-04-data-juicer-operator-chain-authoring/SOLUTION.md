# Exercise 04 — Data-Juicer Operator-Chain Authoring (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner wires up a CCNet-style
curation pipeline using only Alibaba Data-Juicer's library operators,
instruments every operator with a uniform observability shape, and writes a
"custom-code gap" report naming which pipeline responsibilities still have
to be covered outside the library.

A good solution has four parts:

1. A **`recipe.yaml`** that chains ≥10 Data-Juicer operators — URL / host
   filter, length / special-character / word-repetition filters, language-ID,
   line-level dedup, perplexity filter, MinHash near-dedup, basic PII
   mappers — with every operator's config documented.
2. A **uniform instrumentation layer** that emits one JSONL record per
   operator with `name`, `version`, `config_sha256`, `docs_in`, `docs_out`,
   `drop_fraction`, `wall_clock_seconds`, `sample_dropped` (10 records with
   reasons).
3. A **slice manifest emitter** that reads the instrumentation output and
   populates the Chapter-07 slice manifest's `operators` list.
4. A **custom-code gap report** that names ≥3 responsibilities Data-Juicer
   did not cover (minimally: the acquire-stage opt-out check, project-
   specific benchmark decontamination, and the manifest emission itself)
   with an explanation for each.

The exercise's deeper point: the "which library do I pick" question is
answered not by the operator list but by the **gap list**. Every operator
library leaves a non-trivial custom-code surface. Making the learner
enumerate that surface is more useful than optimising an operator chain.

The operator names in the recipe below are the ones in Data-Juicer at
authoring time; the catalog evolves, so every operator must be verified
against the Data-Juicer repo's operator table at the pinned version. Any
drift is marked `<!-- unverified: ... -->` with a substitution rather
than silently renamed.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── requirements.lock
├── recipe.yaml
├── code/
│   ├── instrumentation.py
│   ├── manifest_emitter.py
│   └── run.sh
├── outputs/                    # curated shards (JSONL.gz)
├── instrumentation/            # one JSONL per operator
│   ├── url_filter.jsonl
│   ├── language_id_score_filter.jsonl
│   └── ...
├── manifest/
│   └── slice_manifest.json
└── custom_code_gap_report.md
```

### 2.2 `recipe.yaml` — the operator chain

```yaml
# recipe.yaml — Data-Juicer operator chain for the Exercise-01 slice.
# Pin every operator at the Data-Juicer version in requirements.lock.
# Operator names are from the Data-Juicer catalog; verify at pinned version.

project_name: cc_en_small_v1_curation
dataset_path: ../exercise-01/output/slice-00000.jsonl.gz
export_path: outputs/slice-curated.jsonl.gz
executor_type: default           # switch to `ray` for the cluster stretch goal
np: 16                           # processes

# Data-Juicer emits per-operator traces under the work_dir when enabled.
open_tracer: true
trace_num: 10                    # sample up to 10 dropped records per op
work_dir: ./outputs/_dj_work

process:
  # 1. URL / host filter (if your JSONL carries URLs)
  - url_filter:
      blocklist_type: "domain"
      # Point at the UT1 adult blocklist (public, widely used).
      # URL evolves; verify. Fallback: ship a local snapshot.
      blocklist_path: "resources/ut1_adult_domains.txt"

  # 2. Text length — drop stubs and giants
  - text_length_filter:
      min_len: 50
      max_len: 100000

  # 3. Language-ID score — English only
  - language_id_score_filter:
      lang: "en"
      min_score: 0.5
      # lid.176.bin SHA pinned in README; path resolved via env.
      model_path: "${LID_176_PATH}"

  # 4. Special-character ratio — drop non-prose soup
  - special_characters_filter:
      min_ratio: 0.0
      max_ratio: 0.4

  # 5. Word-repetition — drop SEO spam
  - word_repetition_filter:
      rep_len: 10
      min_ratio: 0.0
      max_ratio: 0.3

  # 6. Alphanumeric ratio — drop code-pasted / symbol-heavy pages
  - alphanumeric_filter:
      tokenization: false
      min_ratio: 0.55
      max_ratio: 1.0

  # 7. Line-level exact dedup
  - document_deduplicator:
      lowercase: true
      ignore_non_character: true

  # 8. Perplexity filter — CCNet-style quality tier
  - perplexity_filter:
      lang: "en"
      max_ppl: 2000
      # KenLM 5-gram trained on English Wikipedia. Path + SHA pinned in README.
      model_path: "${KENLM_EN_PATH}"

  # 9. Document-level MinHash near-dedup
  - document_minhash_deduplicator:
      tokenization: "space"
      window_size: 5
      num_permutations: 128
      jaccard_threshold: 0.8
      num_bands: 9
      num_rows_per_band: 13

  # 10. PII: email + IP redaction (mappers, not filters)
  - clean_email_mapper: {}
  - clean_ip_mapper: {}
```

Config notes worth writing into `README.md`:

- Operator names have drifted over Data-Juicer releases (e.g. renames from
  `_deduplicator` → `_filter` + `_deduplicator` splits). Every operator above
  must be re-verified against the catalog at your pinned version.
- `url_filter` lists a UT1 domain blocklist. Loading UT1 is a Chapter-05
  choice; verify the UT1 source URL at run time. If the operator name at
  the pinned version is different (e.g. `domain_filter`), substitute and
  flag with `<!-- unverified: ... -->`.
- `perplexity_filter` requires a KenLM model. The path, the Wikipedia dump
  date it was trained on, and its SHA-256 belong in the README.
- MinHash params (`num_permutations=128`, `jaccard_threshold=0.8`,
  `9×13` banding) match the common CCNet / Dolma defaults; any deviation
  must be documented.
- The two PII mappers are *redaction stand-ins*, not production PII controls.
  mod-110 covers the context-aware recognisers that belong on a real pipeline.
  The gap report names this as a known gap, not a hidden one.

### 2.3 `code/instrumentation.py` — uniform per-operator records

Data-Juicer ships its own `TracerProvider` that emits per-operator traces
under `work_dir`. The wrapper below is a thin post-processor: it reads
those trace files and emits the standardised schema the exercise requires.
This is the most portable option — a learner can replace it later with a
fork of the tracer, or with an in-process hook, without changing the
downstream consumers.

```python
# code/instrumentation.py — standardise Data-Juicer trace output.
import hashlib
import json
import time
from pathlib import Path
from typing import Iterable

import yaml

RECIPE = Path("recipe.yaml")
TRACE_DIR = Path("outputs/_dj_work/trace")
OUT_DIR = Path("instrumentation")
OUT_DIR.mkdir(parents=True, exist_ok=True)

def _config_sha(op: dict) -> str:
    return hashlib.sha256(
        yaml.safe_dump(op, sort_keys=True).encode()
    ).hexdigest()

def _iter_operator_configs() -> Iterable[tuple[str, dict]]:
    recipe = yaml.safe_load(RECIPE.read_text())
    for entry in recipe["process"]:
        [(name, cfg)] = entry.items()
        yield name, cfg or {}

def _read_trace(name: str) -> dict:
    """Read Data-Juicer's trace files for one operator.

    DJ writes trace shards per operator under TRACE_DIR; the exact filename
    pattern has drifted between releases. Verify at pinned version; this
    helper assumes <op>-<shard>.jsonl is the current convention. Treat the
    returned dict as best-effort and fall back to counting input/output
    shards if trace is unavailable.
    """
    docs_in = docs_out = 0
    dropped_samples = []
    wall = 0.0
    for p in sorted(TRACE_DIR.glob(f"{name}-*.jsonl")):
        with p.open() as fh:
            for line in fh:
                rec = json.loads(line)
                docs_in += rec.get("docs_in", 0)
                docs_out += rec.get("docs_out", 0)
                wall += rec.get("wall_seconds", 0.0)
                if rec.get("dropped") and len(dropped_samples) < 10:
                    dropped_samples.append({
                        "text_prefix": (rec["dropped"].get("text") or "")[:200],
                        "reason": rec["dropped"].get("reason", "n/a"),
                    })
    return {
        "docs_in": docs_in,
        "docs_out": docs_out,
        "wall_clock_seconds": wall,
        "sample_dropped": dropped_samples,
    }

def main() -> None:
    dj_version = _dj_version()
    for name, cfg in _iter_operator_configs():
        trace = _read_trace(name)
        drop = (
            0.0 if trace["docs_in"] == 0
            else (trace["docs_in"] - trace["docs_out"]) / trace["docs_in"]
        )
        record = {
            "name": name,
            "version": dj_version,
            "config_sha256": _config_sha(cfg),
            "docs_in": trace["docs_in"],
            "docs_out": trace["docs_out"],
            "drop_fraction": drop,
            "wall_clock_seconds": trace["wall_clock_seconds"],
            "sample_dropped": trace["sample_dropped"],
            "emitted_at": int(time.time()),
        }
        with (OUT_DIR / f"{name}.jsonl").open("w") as fh:
            fh.write(json.dumps(record, ensure_ascii=False) + "\n")

def _dj_version() -> str:
    try:
        import data_juicer  # type: ignore
        return getattr(data_juicer, "__version__", "unknown")
    except Exception:
        return "unknown"

if __name__ == "__main__":
    main()
```

### 2.4 `code/manifest_emitter.py` — Chapter-07 slice manifest

```python
# code/manifest_emitter.py — read instrumentation; emit the slice manifest.
import datetime as dt
import hashlib
import json
from pathlib import Path

INSTR = Path("instrumentation")
OUT = Path("manifest/slice_manifest.json")
OUT.parent.mkdir(parents=True, exist_ok=True)

def _sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()

def main() -> None:
    operators = []
    for p in sorted(INSTR.glob("*.jsonl")):
        with p.open() as fh:
            rec = json.loads(fh.read().strip().splitlines()[0])
        operators.append({
            "name": rec["name"],
            "library": "data-juicer",
            "library_version": rec["version"],
            "config_sha256": rec["config_sha256"],
            "docs_in": rec["docs_in"],
            "docs_out": rec["docs_out"],
            "drop_fraction": rec["drop_fraction"],
            "wall_clock_seconds": rec["wall_clock_seconds"],
        })

    shards = sorted(Path("outputs").glob("slice-curated*.jsonl*"))
    manifest = {
        "slice_name": "slice-cc-en-curated-v1",
        "slice_version": "1.0.0",
        "upstream_slice": "slice-cc-en-small-v1 (Exercise 01)",
        "operators": operators,
        "shards": [
            {"path": str(s), "sha256": _sha256_file(s)}
            for s in shards
        ],
        "built_at": dt.datetime.utcnow().replace(microsecond=0).isoformat() + "Z",
        "builder": {"user": "<fill>", "git_sha": "<fill>"},
    }
    OUT.write_text(json.dumps(manifest, indent=2, sort_keys=True) + "\n")

if __name__ == "__main__":
    main()
```

### 2.5 `code/run.sh` — one-shot runner

```bash
#!/usr/bin/env bash
# code/run.sh — run the Data-Juicer chain + emit instrumentation + manifest.
set -euo pipefail
: "${LID_176_PATH:?point at lid.176.bin (Exercise-01 download)}"
: "${KENLM_EN_PATH:?point at your English KenLM 5-gram}"

python -m data_juicer.tools.process_data --config recipe.yaml
python code/instrumentation.py
python code/manifest_emitter.py
echo "[run] curated shards under outputs/, manifest under manifest/"
```

### 2.6 Custom-code gap report — worked example

```markdown
# Custom-code gap report — Data-Juicer @ v<pinned>

Data-Juicer covered the middle of the pipeline. The ends — governance in,
provenance out — are custom code. The gaps below are why "we use Data-Juicer"
is not the same as "we have a production pipeline".

## Gap 1 — Acquire-stage opt-out (Chapter 06)
**What the library gives you:** a `url_filter` operator that accepts a
blocklist. That is a **publisher**-side block (admin bans a domain), not a
**publisher-choice** opt-out (the host added `GPTBot: Disallow` to
`robots.txt`).
**What we wrote:** the Chapter-06 acquire-stage check (reused from
Exercise 01). One `robots.txt` fetch per host; evidence hashed and stored;
per-URL decision joined onto the JSONL before Data-Juicer runs.
**Why it is not in the library:** honouring publisher-choice opt-outs is a
compliance question (lawful basis, GDPR Art. 6, pipeline-wide policy), not an
operator. The library operator can block a host; the compliance decision —
which hosts to block, from which signals, with what evidence — is a project
choice. NIST AI RMF `MAP 4.1` is the governance anchor.

## Gap 2 — Project benchmark decontamination
**What the library gives you:** some decontamination operators, at various
levels of maturity (verify against the pinned version). They target a public
benchmark set.
**What we wrote:** a list of our internal benchmarks + their n-gram indices +
a wrapper operator that hashes a document against the index and sets an
attribute. Downstream Data-Juicer filters on the attribute.
**Why it is not in the library:** the benchmark list is organisation-
specific. The library cannot guess which evals we care about. mod-106 covers
the n-gram index + contamination-threshold policy; this operator is a thin
adapter around that.

## Gap 3 — Slice manifest emission (Chapter 07)
**What the library gives you:** per-operator traces in Data-Juicer's trace
format. Useful for debugging, not a manifest.
**What we wrote:** `manifest_emitter.py` (see §2.4) — reads the
instrumentation output and builds a Chapter-07 slice manifest with
`operators[]`, shards, builder context, and SHA-256s.
**Why it is not in the library:** the manifest shape is project-defined. A
future Data-Juicer release may ship a manifest exporter; until it emits the
Chapter-07 shape with SHA-256s and acquire-stage evidence pointers, it is a
seed, not a replacement.

## Gap 4 — Production PII (mod-110 scope)
**What the library gives you:** regex-based `clean_email_mapper` /
`clean_ip_mapper`.
**What we wrote:** nothing in this exercise. The gap is named because the
recipe explicitly uses the mappers as stand-ins; a production pipeline
layers a context-aware PII recogniser (see mod-110) on top of or in place
of the regex mappers. Treating the regex mappers as "PII, done" is the
anti-pattern.

## Gap 5 — Source signing / attestation (mod-109 scope)
**What the library gives you:** nothing explicit.
**What we wrote:** nothing in this exercise. The gap is named because the
slice manifest's `shards[].sha256` is the input to a Sigstore signing step
(and eventually a SLSA attestation); the signing step lives in mod-109 and
is out of scope here. The *absence* of signing is a real property of the
output slice, not a silent omission.
```

## 3. Validation steps

1. **Operator count.** `recipe.yaml` chains at least 10 Data-Juicer
   operators. Comments or stub entries do not count.
2. **Operators resolve at the pinned version.** `python -c "import
   data_juicer; data_juicer.op_registry.list()"` (or the current CLI
   equivalent) lists every operator in the recipe. Any absence is flagged
   `<!-- unverified: ... -->` with a substitution.
3. **Uniform instrumentation schema.** Every file in `instrumentation/`
   has the same keys: `name`, `version`, `config_sha256`, `docs_in`,
   `docs_out`, `drop_fraction`, `wall_clock_seconds`, `sample_dropped`,
   `emitted_at`. A schema check (`jq -s 'map(keys) | unique'`) returns
   one key set.
4. **`sample_dropped` has reasons.** At least one operator has a
   non-empty `sample_dropped` with a `reason` field. If every operator
   shows empty samples, the tracer is misconfigured or the sample size is
   wrong.
5. **Manifest validates against the Chapter-07 shape.** `operators[]`,
   `shards[]`, `built_at`, `builder`, `slice_name`, `slice_version` are
   present; every `shards[].sha256` matches a re-hash of the file on disk.
6. **Gap report names ≥3 responsibilities.** Each has a one-paragraph
   explanation that distinguishes a *library choice* from a *compliance
   choice*. "The library is missing X" without explaining who is responsible
   for X is a smell.
7. **Model paths and SHAs are pinned.** `lid.176.bin` and KenLM model
   SHAs are in `README.md`. If the exercise was run without KenLM, that
   operator is commented out with a `<!-- unverified: ... -->` note.
8. **`make reproduce` runs end-to-end.** Fresh checkout → operator chain →
   instrumentation → manifest.
9. **No invented numbers.** Where a wall-clock, throughput, or quality
   number is cited, it came from `instrumentation/*.jsonl`, not the author's
   estimate.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
Pass at ≥ 20/30 with zero zeros on **critical** rows.

| # | Dimension                                                                                  | Critical? |
|---|--------------------------------------------------------------------------------------------|-----------|
| 1 | `recipe.yaml` chains ≥10 Data-Juicer operators; each has documented config                 | Yes       |
| 2 | Operator names verified against pinned Data-Juicer version (or flagged)                    | Yes       |
| 3 | `data-juicer` version pinned in `requirements.lock`                                        | Yes       |
| 4 | Uniform instrumentation schema (`name`, `version`, `config_sha256`, `docs_in`, `docs_out`, `drop_fraction`, `wall_clock_seconds`, `sample_dropped`) | Yes       |
| 5 | `sample_dropped` includes a reason per sample, non-empty for ≥1 operator                   | Yes       |
| 6 | Slice manifest validates against the Chapter-07 shape; `operators` populated from traces   | Yes       |
| 7 | Gap report names ≥3 responsibilities with explanations                                      | Yes       |
| 8 | Gap report distinguishes library choice vs compliance choice                                | Yes       |
| 9 | `lid.176.bin` and KenLM model SHAs are pinned in README                                    | Yes       |
| 10 | Acquire-stage check runs **before** Data-Juicer sees the content                           | Yes       |
| 11 | Shard SHA-256s in the manifest match re-computed hashes                                    | Yes       |
| 12 | Governance anchors (NIST AI RMF, OWASP ML Top 10) cited at least once                       | No        |
| 13 | PII mappers flagged as stand-ins, not production controls                                   | No        |
| 14 | `make reproduce` works on fresh checkout                                                   | Yes       |
| 15 | No invented performance numbers                                                             | Yes       |

## 5. Common mistakes

- **Chaining 10 operators by including redundant filters.** Three
  near-duplicate length filters do not a 10-operator chain make. The
  operator set should cover distinct responsibilities: URL / length /
  content shape / language / perplexity / dedup / PII.
- **Trusting the operator name from an older tutorial.** Data-Juicer's
  catalog has drifted; always verify at your pinned version.
- **Treating `clean_email_mapper` as "PII, done".** Regex-based mappers
  are stand-ins. mod-110 is where the real controls live; this exercise
  names the gap rather than papering over it.
- **No `config_sha256` on operator records.** Without the config hash, a
  config edit that changes the operator's effect is invisible in the
  manifest.
- **One trace file per shard, not per operator.** The exercise requires
  one JSONL per operator; shard-level is not substitutable (you cannot
  compute per-operator drop fractions from shard-level traces).
- **Running the chain without the Chapter-06 check upstream.** The gap
  report is supposed to *name* the missing acquire-stage check; running
  without one also validates the gap report is wrong (the gap is covered
  outside the library, not ignored).
- **Pinning the operator chain without pinning the KenLM dump date.**
  Perplexity scores shift with the KenLM model; `max_ppl=2000` means
  nothing without the model's dump date and SHA.
- **Gap report that only lists "missing operators".** "Data-Juicer does
  not have an X operator" is not a gap report entry on its own; the entry
  must say what we wrote, who owns the gap, and why the library is not the
  right owner.

## 6. References

- **Data-Juicer** — Chen et al., "Data-Juicer: A One-Stop Data Processing
  System for Large Language Models" (2023),
  <https://arxiv.org/abs/2309.02033>. Source:
  <https://github.com/modelscope/data-juicer>. Operator catalog lives in
  the repo README at the pinned version.
- **CCNet** — Wenzek et al. (2019),
  <https://arxiv.org/abs/1911.00359>. The pipeline shape the recipe
  mirrors.
- **Gopher data-quality rules** — Rae et al. (2021),
  <https://arxiv.org/abs/2112.11446>, §A1. Thresholds for word repetition,
  mean word length, etc.
- **UT1 blocklists** — Université Toulouse 1 blocklists, widely used as a
  starting point for URL / domain denylists; verify the current
  distribution URL at run time.
- **FastText `lid.176`** —
  <https://fasttext.cc/docs/en/language-identification.html>.
- **KenLM** — Heafield, "KenLM: Faster and Smaller Language Model
  Queries" (2011), <https://kheafield.com/code/kenlm/>. Required for the
  `perplexity_filter` operator.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02
  anchors the "pipeline without an acquire-stage check" risk; the gap
  report names this.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MAP 4.1` and
  `MEASURE 2.10` anchor the compliance-vs-library split the gap report
  makes.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. Browse at run time for the
  current technique IDs under "Poison Training Data".
- **Local anchors:** Chapter 05 (operator libraries) for the catalog
  framing; Chapter 06 for the acquire-stage policy; Chapter 07 for the
  slice manifest shape; mod-106 for benchmark decontamination; mod-109
  for signing / attestation; mod-110 for context-aware PII.
