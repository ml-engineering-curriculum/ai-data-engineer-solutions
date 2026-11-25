# Exercise 01 — Presidio PII Pipeline at Corpus Scale (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner stands up a working
Microsoft Presidio-based detection + redaction pipeline over one slice
of the training corpus, produces per-document findings as a parquet
table, emits a per-run manifest with hashes, and writes a short
calibration note that quantifies precision / recall on a labelled
micro-benchmark.

Deliverables (taken verbatim from the exercise contract):

- `recipes/presidio_pipeline.yaml` — the pinned configuration: analyzer
  recognizer set, confidence threshold, language list, anonymizer
  operator table, and target slice hash.
- `code/` — a runnable, resumable pipeline that reads the slice's
  documents, runs the Presidio analyzer + anonymizer, and emits
  `outputs/findings.parquet`, `outputs/redacted/` and
  `outputs/pipeline_manifest.yaml`.
- `calibration/calibration.md` — precision / recall by entity type on
  a held-out, hand-labelled micro-benchmark, plus wall-clock and cost.
- `calibration/micro_benchmark.jsonl` — the labelled micro-benchmark
  the calibration report references.
- `tests/` — pytest cases that fail on the common regressions in
  §5 (plural leakage, confidence threshold drift, resume-schema drift).

Governance framing. The pipeline is the first-line compensating
control for personal-data exposure in a training corpus. The OWASP
Machine Learning Security Top 10 anchor is **ML02:2023 Data Poisoning
Attack** (personal data flowing into training is a self-poisoning
vector that materialises later as memorised regurgitation) and
**ML04:2023 Membership Inference Attack** (unredacted personal data
is the surface area a later membership-inference attack probes). The
MITRE ATLAS technique in scope is **AML.T0024 Exfiltration via ML
Inference API** — the extraction attack this control is designed to
blunt. The NIST AI Risk Management Framework anchors are **MEASURE
2.10** (privacy risk is characterised across the lifecycle) and
**MANAGE 2.2** (mechanisms are in place to supersede, disengage, or
deactivate AI systems that demonstrate performance or outcomes
inconsistent with intended use). Cite the exact framework URLs in
the References section rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── presidio_pipeline.yaml       # pinned recognizers + thresholds
├── code/
│   ├── pipeline.py                  # main runner; --resume supported
│   ├── recognizers.py               # custom + registry overrides
│   ├── anonymize.py                 # operator mapping
│   ├── manifest.py                  # emit per-run YAML
│   └── calibrate.py                 # precision/recall against micro-bench
├── outputs/
│   ├── findings.parquet             # one row per (doc, span)
│   ├── findings.parquet.sha256
│   ├── redacted/                    # anonymized documents
│   └── pipeline_manifest.yaml
├── calibration/
│   ├── micro_benchmark.jsonl        # ≥ 200 hand-labelled spans
│   └── calibration.md
└── tests/
    ├── test_recognizer_registry.py
    ├── test_confidence_threshold.py
    └── test_resume_schema.py
```

### 2.2 `recipes/presidio_pipeline.yaml`

```yaml
# recipes/presidio_pipeline.yaml
# Presidio configuration pinned for reproducibility.
# Every knob the pipeline reads is declared here; no CLI overrides.

slice:
  id: slice-example-v1
  sha256: <64-hex-digest-of-the-jsonl-slice>
  item_count: 1000000

analyzer:
  # Languages the analyzer registers. A doc whose lang_id != any of
  # these is routed to the `unsupported_language` channel and NOT
  # scanned. Chapter 10 §"Language coverage" is explicit — do not
  # scan with the wrong language model; false-negatives explode.
  supported_languages: [en]
  nlp_engine:
    engine_name: spacy
    models:
      - lang_code: en
        model_name: en_core_web_lg          # pinned major.minor via pip lock
  # Confidence threshold at which analyzer findings are kept. 0.4 is
  # Presidio's documented default for recall-leaning workloads; the
  # micro-benchmark in §2.7 is what justifies deviating.
  default_score_threshold: 0.4

recognizers:
  # Entities we rely on from the default registry. Each one must be
  # exercised by at least one row in the micro-benchmark or we drop
  # the registration.
  registry:
    - name: EmailRecognizer
      entity: EMAIL_ADDRESS
    - name: PhoneRecognizer
      entity: PHONE_NUMBER
      context: [phone, mobile, cell, tel]
    - name: UsSsnRecognizer
      entity: US_SSN
    - name: CreditCardRecognizer
      entity: CREDIT_CARD
    - name: IpRecognizer
      entity: IP_ADDRESS
    - name: UrlRecognizer
      entity: URL
    - name: PersonRecognizer                # spaCy NER-backed
      entity: PERSON
      score_threshold: 0.6                  # higher than default
    - name: LocationRecognizer
      entity: LOCATION
      score_threshold: 0.6
  # Custom recognizers declared in code/recognizers.py. Each must be
  # unit-tested with ≥ 2 positive and ≥ 2 negative strings.
  custom:
    - name: InternalUserIdRecognizer
      entity: INTERNAL_USER_ID
      pattern: "u-[0-9a-f]{12}"
      score: 0.9
    - name: UkNhsNumberRecognizer           # 10-digit, Mod-11 checksum
      entity: UK_NHS_NUMBER

anonymizer:
  # Operator table. Each entity maps to a documented primitive. The
  # choice of primitive is the subject of Exercise 02; this run uses
  # the audit-safe default.
  operators:
    DEFAULT:
      type: replace
      new_value: "[REDACTED]"
    EMAIL_ADDRESS:
      type: replace
      new_value: "[EMAIL]"
    PHONE_NUMBER:
      type: replace
      new_value: "[PHONE]"
    US_SSN:
      type: redact                          # full removal, no placeholder
    CREDIT_CARD:
      type: mask
      masking_char: "*"
      chars_to_mask: 12
      from_end: false
    IP_ADDRESS:
      type: replace
      new_value: "[IP]"
    URL:
      type: hash                            # SHA-256, 16-char prefix
      hash_type: sha256
    PERSON:
      type: replace
      new_value: "[PERSON]"
    LOCATION:
      type: replace
      new_value: "[LOCATION]"
    INTERNAL_USER_ID:
      type: hash
      hash_type: sha256
    UK_NHS_NUMBER:
      type: redact

routing:
  # Where to send documents the analyzer cannot score confidently.
  unsupported_language_dir: outputs/unsupported_language/
  analyzer_error_dir:       outputs/analyzer_errors/

performance:
  workers: 8                                # per-host; tune to CPU count
  batch_size_docs: 200
  max_text_bytes_per_doc: 524288            # 512 KB; larger docs chunked

normalisation:
  # Canonical form applied before analyzing. Match Exercise 05's
  # corpus-slice canonical form. Pinned.
  nfc: true
  collapse_whitespace: true
  preserve_case: true
  version: nfc-ws-collapse-v1
```

Why each knob matters at review:

- **`supported_languages` is a hard gate.** A non-`en` document scanned
  with an `en` model produces close-to-zero recall and misleadingly low
  finding counts. Route unsupported documents, do not "try anyway".
- **`default_score_threshold` must be pinned here, not inferred.** The
  threshold is the single knob reviewers attack first — "why 0.4 and
  not 0.5?" is answered by the micro-benchmark results in §2.7.
- **PERSON / LOCATION run at a higher threshold.** The NER backbone is
  the noisiest recognizer; keeping it at 0.6 reduces the precision
  cliff at the cost of recall that the micro-benchmark will quantify.

### 2.3 `code/recognizers.py`

```python
# code/recognizers.py
# Custom recognizers and registry overrides.
from __future__ import annotations

from presidio_analyzer import (
    AnalyzerEngine,
    Pattern,
    PatternRecognizer,
    RecognizerRegistry,
)
from presidio_analyzer.nlp_engine import NlpEngineProvider


INTERNAL_USER_ID_PATTERN = Pattern(
    name="internal-user-id",
    regex=r"\bu-[0-9a-f]{12}\b",
    score=0.9,
)


def build_internal_user_id_recognizer() -> PatternRecognizer:
    return PatternRecognizer(
        supported_entity="INTERNAL_USER_ID",
        patterns=[INTERNAL_USER_ID_PATTERN],
        context=["user", "account", "member"],
        name="InternalUserIdRecognizer",
    )


# UK NHS number: 10 digits, Mod-11 checksum.
NHS_PATTERN = Pattern(
    name="uk-nhs-number",
    regex=r"\b\d{3}\s?\d{3}\s?\d{4}\b",
    score=0.5,     # low until checksum validates
)


class UkNhsNumberRecognizer(PatternRecognizer):
    def __init__(self) -> None:
        super().__init__(
            supported_entity="UK_NHS_NUMBER",
            patterns=[NHS_PATTERN],
            context=["nhs", "patient", "clinical"],
            name="UkNhsNumberRecognizer",
        )

    def validate_result(self, pattern_text: str) -> bool:
        digits = [int(c) for c in pattern_text if c.isdigit()]
        if len(digits) != 10:
            return False
        total = sum(d * (10 - i) for i, d in enumerate(digits[:9]))
        check = (11 - (total % 11)) % 11
        return check != 10 and check == digits[9]


def build_analyzer(cfg: dict) -> AnalyzerEngine:
    provider = NlpEngineProvider(nlp_configuration={
        "nlp_engine_name": cfg["analyzer"]["nlp_engine"]["engine_name"],
        "models": cfg["analyzer"]["nlp_engine"]["models"],
    })
    nlp_engine = provider.create_engine()

    registry = RecognizerRegistry()
    registry.load_predefined_recognizers(
        languages=cfg["analyzer"]["supported_languages"],
        nlp_engine=nlp_engine,
    )
    registry.add_recognizer(build_internal_user_id_recognizer())
    registry.add_recognizer(UkNhsNumberRecognizer())

    return AnalyzerEngine(
        registry=registry,
        nlp_engine=nlp_engine,
        supported_languages=cfg["analyzer"]["supported_languages"],
        default_score_threshold=cfg["analyzer"]["default_score_threshold"],
    )
```

Design notes defensible at review:

- **Checksum validation gates the NHS pattern.** A raw 10-digit regex
  has a false-positive rate on receipts, order numbers, and timestamps
  that makes the recognizer useless. The Mod-11 check is the published
  validator; keep the base regex `score=0.5` and promote on checksum.
- **Context terms are not optional.** `PhoneRecognizer` with no
  `context=["phone", ...]` makes every 10-digit string a finding; the
  context nudge is what separates a phone number from a product SKU.
- **Pinned spaCy model.** `en_core_web_lg` is pinned in
  `requirements.lock`. A silent upgrade of the backbone changes PERSON
  recall and invalidates the calibration.

### 2.4 `code/pipeline.py` (loop skeleton)

```python
# code/pipeline.py
# Resumable, chunked Presidio pipeline.
# Reads a jsonl slice, writes outputs/findings.parquet and anonymized
# documents under outputs/redacted/, plus a per-run manifest.
from __future__ import annotations

import argparse
import hashlib
import json
import logging
import multiprocessing as mp
import unicodedata
from dataclasses import asdict, dataclass
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
import yaml
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

from recognizers import build_analyzer

LOG = logging.getLogger("pipeline")

FINDINGS_SCHEMA = pa.schema([
    ("doc_id",            pa.string()),
    ("span_index",        pa.int32()),
    ("entity_type",       pa.string()),
    ("start",             pa.int32()),
    ("end",               pa.int32()),
    ("score",             pa.float32()),
    ("recognizer_name",   pa.string()),
    ("text_hash",         pa.string()),       # sha256 of the matched text
    ("source_language",   pa.string()),
    ("pipeline_run_id",   pa.string()),
])


@dataclass(slots=True)
class DocResult:
    doc_id: str
    language: str
    findings: list[dict]
    redacted_text: str


def canonicalise(text: str, cfg: dict) -> str:
    if cfg["normalisation"]["nfc"]:
        text = unicodedata.normalize("NFC", text)
    if cfg["normalisation"]["collapse_whitespace"]:
        text = " ".join(text.split())
    return text


def build_operators(cfg: dict) -> dict[str, OperatorConfig]:
    out: dict[str, OperatorConfig] = {}
    for entity, op in cfg["anonymizer"]["operators"].items():
        params = {k: v for k, v in op.items() if k != "type"}
        out[entity] = OperatorConfig(operator_name=op["type"], params=params)
    return out


def process_doc(doc: dict, analyzer, anonymizer, operators,
                run_id: str, cfg: dict) -> DocResult:
    text = canonicalise(doc["text"], cfg)
    results = analyzer.analyze(
        text=text,
        language=cfg["analyzer"]["supported_languages"][0],
    )
    anonymized = anonymizer.anonymize(
        text=text, analyzer_results=results, operators=operators,
    )
    findings = []
    for i, r in enumerate(results):
        matched = text[r.start:r.end]
        findings.append({
            "doc_id":           doc["doc_id"],
            "span_index":       i,
            "entity_type":      r.entity_type,
            "start":            r.start,
            "end":              r.end,
            "score":            float(r.score),
            "recognizer_name":  r.recognition_metadata.get("recognizer_name", ""),
            "text_hash":        hashlib.sha256(matched.encode("utf-8")).hexdigest(),
            "source_language":  doc.get("language", "en"),
            "pipeline_run_id":  run_id,
        })
    return DocResult(
        doc_id=doc["doc_id"],
        language=doc.get("language", "en"),
        findings=findings,
        redacted_text=anonymized.text,
    )


def iter_slice(slice_path: Path):
    with slice_path.open() as f:
        for line in f:
            if line.strip():
                yield json.loads(line)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config",   required=True, type=Path)
    ap.add_argument("--slice",    required=True, type=Path)
    ap.add_argument("--run-id",   required=True)
    ap.add_argument("--resume",   action="store_true")
    args = ap.parse_args()

    cfg = yaml.safe_load(args.config.read_text())
    slice_sha = hashlib.sha256(args.slice.read_bytes()).hexdigest()
    if slice_sha != cfg["slice"]["sha256"]:
        raise SystemExit(
            f"slice hash drift: config says {cfg['slice']['sha256']}, "
            f"on disk is {slice_sha}"
        )

    out_dir = Path("outputs")
    out_dir.mkdir(exist_ok=True)
    (out_dir / "redacted").mkdir(exist_ok=True)

    analyzer   = build_analyzer(cfg)
    anonymizer = AnonymizerEngine()
    operators  = build_operators(cfg)

    processed_ids: set[str] = set()
    findings_path = out_dir / "findings.parquet"
    if args.resume and findings_path.exists():
        existing = pq.read_table(findings_path)
        processed_ids.update(existing.column("doc_id").to_pylist())
        LOG.info("resuming after %d documents", len(processed_ids))

    buffer: list[dict] = []
    BUFFER_LIMIT = 10_000

    writer = None
    if findings_path.exists():
        writer = pq.ParquetWriter(findings_path, FINDINGS_SCHEMA,
                                   compression="zstd")
    else:
        writer = pq.ParquetWriter(findings_path, FINDINGS_SCHEMA,
                                   compression="zstd")

    try:
        for doc in iter_slice(args.slice):
            if doc["doc_id"] in processed_ids:
                continue
            res = process_doc(doc, analyzer, anonymizer, operators,
                              args.run_id, cfg)
            buffer.extend(res.findings)
            red_path = out_dir / "redacted" / f"{doc['doc_id']}.txt"
            red_path.write_text(res.redacted_text)
            if len(buffer) >= BUFFER_LIMIT:
                writer.write_table(pa.Table.from_pylist(buffer,
                                                         schema=FINDINGS_SCHEMA))
                buffer.clear()
        if buffer:
            writer.write_table(pa.Table.from_pylist(buffer,
                                                     schema=FINDINGS_SCHEMA))
    finally:
        writer.close()

    _write_sha256(findings_path)
    _emit_manifest(args, cfg, slice_sha)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
```

Design notes defensible at review:

- **Slice hash drift aborts the run.** A `slice_sha` mismatch means the
  input diverged from the pin; continuing would silently invalidate the
  manifest. Fail loud, fail fast.
- **Deduped on `doc_id` during `--resume`.** Re-reading the parquet is
  authoritative. Any other state (progress file, SQLite) is a second
  source of truth that will drift.
- **Schema pinned via `FINDINGS_SCHEMA`.** `pq.ParquetWriter` enforces
  the schema row-by-row; a drift on `score` dtype or a new column fails
  the write, which is the behaviour we want.
- **Matched text is not persisted — only its SHA-256.** The findings
  table lives alongside less-trusted systems; storing the raw PII would
  recreate the risk the pipeline was designed to remove.

### 2.5 `code/manifest.py`

```python
# code/manifest.py
from __future__ import annotations

import hashlib
import platform
import subprocess
import sys
from datetime import datetime, timezone
from pathlib import Path

import yaml


def _sha256(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def emit_manifest(run_id: str, config_path: Path, slice_path: Path,
                  findings_path: Path, out_path: Path) -> None:
    manifest = {
        "run_id":                run_id,
        "started_at_utc":        datetime.now(timezone.utc)
                                   .isoformat(timespec="seconds"),
        "python":                sys.version.split()[0],
        "platform":              platform.platform(),
        "git_commit":            _git_commit(),
        "config_path":           str(config_path),
        "config_sha256":         _sha256(config_path),
        "slice_path":            str(slice_path),
        "slice_sha256":          _sha256(slice_path),
        "findings_path":         str(findings_path),
        "findings_sha256":       _sha256(findings_path),
        "pip_lock_sha256":       _sha256(Path("requirements.lock")),
    }
    out_path.write_text(yaml.safe_dump(manifest, sort_keys=True))


def _git_commit() -> str:
    try:
        return subprocess.check_output(
            ["git", "rev-parse", "HEAD"], text=True).strip()
    except Exception:
        return "unknown"
```

### 2.6 `calibration/micro_benchmark.jsonl`

A hand-labelled benchmark of ≥ 200 spans spanning every entity declared
in the recipe. Each line is one document with a gold-standard span
list:

```json
{
  "doc_id": "bench-00001",
  "text": "Call Dr Alice Mendez at alice.mendez@clinic.example or +44 20 7946 0958.",
  "spans": [
    {"entity_type": "PERSON",        "start": 8,  "end": 20},
    {"entity_type": "EMAIL_ADDRESS", "start": 24, "end": 51},
    {"entity_type": "PHONE_NUMBER",  "start": 55, "end": 71}
  ]
}
```

Rules enforced by `code/calibrate.py`:

- ≥ 10 positive and ≥ 10 negative examples per entity type. "Negative"
  means a near-miss (e.g., a phone-shaped product SKU) that the
  recognizer should not flag.
- Span boundaries use offsets after the pipeline's canonical form
  (`nfc + collapse-whitespace + preserve-case`). Raw-text offsets
  drift silently when the normaliser runs.
- Gold labels are reviewed by a second author; the reviewer's email is
  pinned in the calibration note.

### 2.7 `calibration/calibration.md`

One page. Required sections:

1. **Summary.** Slice id, scanner config digest, micro-benchmark size,
   wall-clock, and cost.
2. **Per-entity precision / recall / F1** at the pinned score
   threshold, computed by `calibrate.py` against the micro-benchmark.
3. **Threshold-sweep plot** (or table) for the two entities with the
   highest false-positive rate — typically `PERSON` and `LOCATION`.
4. **Known failure classes.** 3–6 concrete regression samples the
   pipeline gets wrong, categorised (e.g., "non-Latin names below the
   NER confidence cliff").
5. **Coverage gap.** Entities declared in the recipe but not exercised
   by the benchmark fail the pre-publish check; list them if any.
6. **Operational posture.** The downstream consumer of
   `redacted/`. The consumer must be told what entities the pipeline
   does not scan for.

Any claim that cannot be verified at authoring time (e.g., an upstream
throughput number from a vendor doc) is marked
`<!-- pending-verify: ... -->` and blocks auto-merge.

## 3. Validation steps

1. **Slice hash matches the recipe.** `sha256sum <slice>.jsonl` equals
   the `slice.sha256` value in `recipes/presidio_pipeline.yaml`. A
   drift fails the run at startup (`code/pipeline.py`).
2. **Pipeline runs clean on the micro-benchmark.** `python
   code/pipeline.py --config … --slice calibration/micro_benchmark.jsonl
   --run-id bench` completes without an analyzer error.
3. **Findings schema is byte-stable.** `python -c "import pyarrow.parquet as pq;
   print(pq.read_schema('outputs/findings.parquet'))"` prints the
   schema declared in `pipeline.py` with no additions or deletions.
4. **Confidence threshold is honoured.** `findings.parquet` has no row
   with `score < default_score_threshold` for any default-recognizer
   row and no row below the per-entity override for `PERSON` /
   `LOCATION`.
5. **Resumability.** `rm outputs/findings.parquet.part && make run &
   kill mid-run && make resume` — the pipeline continues from the
   highest unprocessed `doc_id`; the final table row count matches a
   fresh full run.
6. **Redacted documents round-trip.** For every `doc_id` in the
   findings table, `outputs/redacted/<doc_id>.txt` exists; its byte
   length is consistent with the sum of operator offsets (byte
   length may differ from source length because of replace /
   mask operators).
7. **No raw PII in `findings.parquet`.** A regex scan of the parquet
   column `text_hash` matches the SHA-256 shape; a scan of every
   string column asserts none contains the literal emails, phones, or
   SSNs in the micro-benchmark. This is the PII-leakage test.
8. **Calibration thresholds.** The pre-publish check asserts
   per-entity recall ≥ 0.85 on the micro-benchmark for every entity
   present in `recipes/…yaml`. If not met, the recognizer is either
   removed or its confidence threshold is lowered with a documented
   re-review.
9. **Manifest signs the run.** `outputs/pipeline_manifest.yaml`
   records `config_sha256`, `slice_sha256`, `findings_sha256`,
   `pip_lock_sha256`, and the git commit. All four hashes match the
   on-disk artifacts.
10. **Unit tests pass.** `pytest tests/` passes, including the
    plural-leakage, confidence-threshold, and resume-schema tests.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros
on critical rows.

| #  | Dimension                                                                                                      | Critical? |
|----|----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `recipes/presidio_pipeline.yaml` pins slice hash, language, recognizer registry, thresholds, operator table    | Yes       |
| 2  | spaCy backbone version is pinned in `requirements.lock`                                                        | Yes       |
| 3  | At least one custom recognizer with a validator (checksum or context gating) is registered                     | Yes       |
| 4  | `code/pipeline.py` fails the run on slice-hash drift                                                           | Yes       |
| 5  | `outputs/findings.parquet` schema matches `FINDINGS_SCHEMA` exactly (no added or dropped columns)               | Yes       |
| 6  | Raw matched text is NOT written to `findings.parquet`; only `text_hash` is stored                              | Yes       |
| 7  | `--resume` continues a killed run and preserves schema                                                         | Yes       |
| 8  | `calibration/micro_benchmark.jsonl` has ≥ 10 positive + ≥ 10 negative spans per registered entity              | Yes       |
| 9  | `calibration/calibration.md` reports per-entity precision / recall / F1 at the pinned threshold                | Yes       |
| 10 | Unsupported-language documents are routed, not scanned with the wrong model                                    | Yes       |
| 11 | `outputs/pipeline_manifest.yaml` records config, slice, findings, pip-lock hashes and git commit               | Yes       |
| 12 | Governance anchors (OWASP ML Top 10 ML02 / ML04, MITRE ATLAS AML.T0024, NIST AI RMF MEASURE 2.10 / MANAGE 2.2) cited | No        |
| 13 | Unverifiable claims are marked `<!-- pending-verify: ... -->`                                                  | Yes       |
| 14 | `pytest tests/` passes (plural leakage, threshold, resume-schema tests included)                               | Yes       |
| 15 | PII-leakage test confirms `findings.parquet` contains no raw benchmark emails / phones / SSNs                  | Yes       |

## 5. Common mistakes

- **Scanning non-English documents with the English model.** Presidio
  silently returns almost zero findings; the operator mistakes the
  low count for a clean slice. Route unsupported languages into a
  separate channel.
- **Shipping a bare spaCy model name.** `en_core_web_lg` without a pin
  in `requirements.lock` drifts on the next `pip install`. Pin both
  the Presidio and spaCy-model versions.
- **Persisting raw PII into `findings.parquet`.** The findings table
  often travels further than the redacted corpus; storing the matched
  text undoes the point. Only the SHA-256 goes in.
- **Flat threshold across all entities.** `PERSON` and `LOCATION` live
  on a precision cliff at the default threshold. Override per entity
  and justify the override in `calibration.md`.
- **No negative examples in the micro-benchmark.** Precision becomes
  unmeasurable; the operator ships a scanner tuned purely for recall.
- **Canonical form drifts between recipe and pipeline.** Document
  offsets in the micro-benchmark correspond to one normalisation
  version. If the pipeline silently changes the normaliser, every
  span offset shifts. Pin `normalisation.version`.
- **Turning on all default recognizers "just in case".** Each extra
  recognizer with no benchmark coverage is a silent noise source. If
  the micro-benchmark does not exercise the entity, drop the
  recognizer.
- **Running the anonymizer without declaring the `DEFAULT` operator.**
  Any entity the operator table forgot (a future custom recognizer,
  for example) falls through to a soft-default that may be `replace("")`
  — i.e., unannounced text loss. The explicit `DEFAULT` closes this.
- **Using `mask` for SSN or NHS numbers.** Partial masking of
  short-fixed-length identifiers leaves enough bits to be recovered by
  context. Use `redact` (full removal); Exercise 02 justifies this.
- **Treating resume state as "whatever is on disk".** A failed write
  mid-batch leaves the parquet in a broken state. Write a batch at
  `BUFFER_LIMIT` boundaries and resume on the parquet's `doc_id`
  column; do not maintain a sidecar progress file.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack / ML04:2023 Membership Inference Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 anchors unredacted personal data as a self-poisoning vector;
  ML04 anchors the downstream membership-inference surface that
  redaction closes.
- **MITRE ATLAS — AML.T0024 Exfiltration via ML Inference API:**
  <https://atlas.mitre.org/>. The attacker-facing technique the
  pipeline's redaction controls are designed to blunt.
- **NIST AI Risk Management Framework — MEASURE 2.10, MANAGE 2.2:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.10 anchors the privacy-risk characterisation; MANAGE 2.2 anchors
  the deactivation posture when scanner precision / recall degrades.
- **Microsoft Presidio documentation:**
  <https://microsoft.github.io/presidio/>. The analyzer / anonymizer
  reference, recognizer registry mechanics, and operator primitives
  the recipe pins.
- **Presidio recognizer registry reference:**
  <https://microsoft.github.io/presidio/analyzer/adding_recognizers/>.
  The source for custom-recognizer design — pattern, score, context
  terms, and `validate_result`.
- **spaCy NER backbone (`en_core_web_lg`):**
  <https://spacy.io/models/en#en_core_web_lg>. The model card for the
  NER backbone; pin the exact version in `requirements.lock`.
- **GDPR Article 32 — Security of processing:**
  <https://gdpr-info.eu/art-32-gdpr/>. The statutory anchor for
  "pseudonymisation and encryption of personal data" as a technical
  measure; the pipeline is the operational implementation of this
  obligation for the training corpus.
- **Local anchors:** Exercise 02 (redaction primitive choice — the
  operator table in §2.2 is the concrete output of that exercise);
  Exercise 03 (lawful basis — the memo that justifies the pipeline's
  existence); Exercise 05 (privacy risk report — the aggregate
  evidence package this run feeds); `mod-104` exercise-01 (Great
  Expectations suite on the slice — the pipeline's redacted outputs
  are an input to the quality suite); `mod-109` (manifest signing of
  `outputs/pipeline_manifest.yaml` for lineage).
