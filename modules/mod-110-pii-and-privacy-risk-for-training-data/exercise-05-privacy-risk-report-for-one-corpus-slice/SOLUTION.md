# Exercise 05 — Privacy Risk Report for One Corpus Slice (Solution)

## 1. Solution overview

This is a **documentation / integration** exercise. The learner
publishes the privacy risk report for one slice — the aggregate of
Exercises 01–04 structured as a reproducible, signed, auditor-ready
artifact that ships alongside the model card. The report is the
outward face of `mod-110`: everything inside the slice's privacy
envelope can be inspected by someone who was not in the room when
the slice was built.

A privacy risk report is **not** a marketing document. It is a
reproducibility packet — the external auditor's instructions for
reproducing every number the slice claims. If the auditor runs the
packet and gets different numbers, the report has failed its purpose
regardless of how well-written it is.

Deliverables (taken verbatim from the exercise contract):

- `PRIVACY_RISK_REPORT.md` — the top-level public document. Six
  sections: identity, lawful basis, scanner coverage, redaction
  posture, extraction posture, and a reproduction block.
- `report/manifest.json` — the machine-readable index of every
  artifact referenced in the report, each with a SHA-256 and a path.
- `report/manifest.json.sig` — cosign signature of the manifest.
- `report/lawful_basis/` — Exercise 03's `GDPR_LAWFUL_BASIS.md`,
  `HIPAA_DEIDENTIFICATION.md`, `admit_policy.yaml`, and
  `sign_off.yaml`.
- `report/scanner/` — Exercise 01's `recipes/presidio_pipeline.yaml`,
  `outputs/pipeline_manifest.yaml`, `outputs/findings.parquet.sha256`,
  `calibration/calibration.md`.
- `report/redaction/` — Exercise 02's `decision_table.md`,
  `rationale.md`, and one operator table per deployed use case.
- `report/extraction/` — Exercise 04's `extraction_report.yaml`,
  `canaries/injection_manifest.yaml`, and the top-hit exhibit.
- `report/reproduce.sh` — the auditor's one-shot reproduction script.
- `report/VERIFY.md` — the auditor-facing verification checklist.

Governance framing. The privacy risk report is a transparency
artifact — publishing it is what makes a slice's privacy claim
**auditable**. The NIST AI Risk Management Framework anchors are
**GOVERN 1.4** (processes and personnel are in place to provide
transparency and explain system outcomes) and **MEASURE 2.10**
(privacy risk is characterised across the lifecycle). The OWASP ML
Top 10 anchors remain **ML02:2023 Data Poisoning Attack** and
**ML04:2023 Membership Inference Attack** — the published report is
the compensating control that establishes the slice's privacy
posture. MITRE ATLAS does not specify a defender technique; the
report is the artifact that makes defender techniques reviewable.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── PRIVACY_RISK_REPORT.md
├── Makefile
├── code/
│   ├── collect_artifacts.py       # copies Ex01–04 outputs under report/
│   ├── build_manifest.py          # emits report/manifest.json
│   ├── render_report_md.py        # renders PRIVACY_RISK_REPORT.md
│   └── reproduce_sanity.py        # local pre-publish verifier
├── report/
│   ├── manifest.json
│   ├── manifest.json.sig
│   ├── cosign.pub
│   ├── lawful_basis/
│   │   ├── GDPR_LAWFUL_BASIS.md
│   │   ├── HIPAA_DEIDENTIFICATION.md
│   │   ├── admit_policy.yaml
│   │   └── sign_off.yaml
│   ├── scanner/
│   │   ├── presidio_pipeline.yaml
│   │   ├── pipeline_manifest.yaml
│   │   ├── findings.parquet.sha256
│   │   ├── calibration.md
│   │   └── micro_benchmark.jsonl.sha256
│   ├── redaction/
│   │   ├── decision_table.md
│   │   ├── rationale.md
│   │   └── operator_tables/
│   │       ├── UC-A-pretrain-sft.yaml
│   │       ├── UC-B-eval-telemetry.yaml
│   │       ├── UC-C-eng-debug.yaml
│   │       └── UC-D-released.yaml
│   ├── extraction/
│   │   ├── extraction_report.yaml
│   │   ├── injection_manifest.yaml
│   │   ├── canary_set.jsonl.sha256
│   │   └── incidents_summary.yaml
│   ├── reproduce.sh
│   └── VERIFY.md
└── tests/
    ├── test_manifest_shas_match.py
    ├── test_signatures_verify.py
    └── test_reproduce_sh_exits_zero.py
```

### 2.2 `PRIVACY_RISK_REPORT.md`

```markdown
# Privacy Risk Report — slice `<slice-id>` v<slice-version>

| Field                     | Value                                                |
|---------------------------|------------------------------------------------------|
| Slice id                  | <slice-id>                                           |
| Slice version             | <slice-version>                                      |
| Published                 | 2026-11-25                                           |
| Controller                | <legal name, address, DPO contact>                   |
| Operator                  | <operator email>                                     |
| Pipeline reviewer         | <ai-risk engineer email>                             |
| Counsel of record         | <counsel email>                                      |
| Report version            | 1.0.0                                                |
| Signing key fingerprint   | <cosign fingerprint>                                 |

## 1. What this report covers

This report publishes the evidence that slice `<slice-id>` has an
auditable privacy posture. It covers:

1. The lawful-basis analysis per data source (Exercise 03).
2. The PII scanner configuration, coverage, and calibration
   (Exercise 01).
3. The per-use-case redaction primitive choice (Exercise 02).
4. The Carlini-style extraction probe results (Exercise 04).

It does **not** cover:

- Downstream fine-tunes or RLHF data derived from this slice.
- Privacy properties of the trained model's operational posture
  (rate-limiting, output filters, logging retention).
- Privacy properties of data sources acquired after the report's
  publication date.

## 2. Lawful basis posture

See `lawful_basis/GDPR_LAWFUL_BASIS.md` and
`lawful_basis/HIPAA_DEIDENTIFICATION.md`. Summary table:

| Source                      | GDPR 6(1)      | GDPR 9(2)  | HIPAA pathway  | Opt-out honoured | Erasure honoured |
|-----------------------------|----------------|------------|----------------|------------------|------------------|
| A-support-tickets           | 6(1)(b)+6(1)(f) | n/a       | No PHI         | yes              | yes              |
| B-public-web-crawl          | 6(1)(f)        | n/a        | No PHI         | yes (per domain) | yes              |
| C-vendor-conversations      | vendor-attested | n/a       | Safe Harbor    | yes              | yes              |
| D-synthetic                 | inherits B     | n/a        | No PHI         | inherits B       | inherits B       |

Sign-off: see `lawful_basis/sign_off.yaml`. Controller, counsel, and
pipeline-reviewer counter-signatures are required; covered-entity
contact counter-signs when any source's HIPAA pathway is not
`No PHI`.

## 3. Scanner coverage and calibration

Scanner configuration: `scanner/presidio_pipeline.yaml` (SHA-256:
`<hash>`). Micro-benchmark: `scanner/micro_benchmark.jsonl.sha256`.

Per-entity precision / recall on the micro-benchmark at the pinned
threshold (full table in `scanner/calibration.md`):

| Entity              | Precision | Recall | F1    | Micro-bench spans |
|---------------------|-----------|--------|-------|-------------------|
| EMAIL_ADDRESS       | 0.98      | 0.97   | 0.97  | 40                |
| PHONE_NUMBER        | 0.93      | 0.91   | 0.92  | 36                |
| US_SSN              | 0.99      | 0.95   | 0.97  | 22                |
| CREDIT_CARD         | 0.96      | 0.94   | 0.95  | 28                |
| IP_ADDRESS          | 0.95      | 0.98   | 0.96  | 30                |
| URL                 | 0.97      | 0.99   | 0.98  | 44                |
| PERSON              | 0.86      | 0.84   | 0.85  | 70                |
| LOCATION            | 0.83      | 0.78   | 0.80  | 60                |
| INTERNAL_USER_ID    | 1.00      | 1.00   | 1.00  | 14                |
| UK_NHS_NUMBER       | 1.00      | 0.95   | 0.97  | 20                |

> Illustrative numbers. Replace with the run's measured rates
> before publish. <!-- pending-verify -->

Residual-rate attestation: the pipeline's precision / recall on the
micro-benchmark is a bound, not a guarantee, on the full-corpus
behaviour. The slice's finding-rate summary is in
`scanner/pipeline_manifest.yaml`; a shift beyond the
micro-benchmark's 95% confidence interval triggers a re-calibration
per `admit_policy.yaml`'s `recalibration_trigger` block.

Coverage gap: entities registered in the scanner but not exercised
by the micro-benchmark are listed in `scanner/calibration.md` §5.
The pre-publish check requires that list to be empty.

## 4. Redaction posture

The slice's deployed use case is **UC-A (pretrain-SFT)**. The
operator table in effect is `redaction/operator_tables/UC-A-
pretrain-sft.yaml` (SHA-256: `<hash>`). The full per-entity
decision and rationale is in `redaction/decision_table.md` and
`redaction/rationale.md`.

Scope-limits (verbatim from Exercise 02):

- `US_SSN`, `CREDIT_CARD`, `IBAN_CODE`, `UK_NHS_NUMBER` are
  **redacted** (full removal).
- `URL` is **hashed** (SHA-256, 16-char prefix).
- `PERSON`, `LOCATION`, `EMAIL_ADDRESS`, `PHONE_NUMBER`,
  `IP_ADDRESS` are **replaced** with a stable placeholder.
- `US_MEDICAL_RECORD`, biometric identifiers, full-face photographs
  are **suppress-document** (any document containing one is
  dropped).

The report does **not** claim the redacted corpus is anonymous
within the meaning of GDPR Recital 26. Redaction is one layer of
the defence-in-depth posture; the lawful-basis section (§2) is
load-bearing.

## 5. Extraction posture

Probe results summary (full data in
`extraction/extraction_report.yaml`):

| Family                           | Canaries | Exact matches | Rate  |
|----------------------------------|----------|---------------|-------|
| F1 — unique PII-shaped            | 1 000    | 4             | 0.004 |
| F2 — duplicated PII-shaped (d=128)| 125      | 38            | 0.304 |
| F3 — high-entropy non-PII         | 1 000    | 0             | 0.000 |

> Illustrative numbers. Replace with the run's measured rates
> before publish. <!-- pending-verify -->

The F2 curve is the empirical evidence that deduplication (`mod-
105`) functions as a privacy control: the memorisation rate rises
sharply with duplication. The current slice's post-dedup target is
documented in `mod-105` exercise-01's acceptance criteria.

Unexpected real-PII hits (non-canary extractions that match PII
shape) are summarised in `extraction/incidents_summary.yaml`. The
report publishes the count and the ticket resolution; it does not
publish the matched strings.

## 6. How to reproduce every number in this report

```bash
cd report/
./reproduce.sh
```

`reproduce.sh` verifies every signature, hashes every artifact, and
re-runs the micro-benchmark through the scanner. See `VERIFY.md`
for the auditor-facing step-by-step.

## 7. Scope and expiry

- This report is valid until a change to the slice's lawful-basis
  memo, scanner config, operator table, or model checkpoint forces
  a slice-version bump, at which point a new report is published
  alongside the new slice.
- The report covers the slice's training mixture. It does **not**
  make claims about downstream fine-tunes or RLHF data derived
  from this slice.
- Extraction-probe results are scoped to the target model id
  recorded in Exercise 04's `extraction_report.yaml`. A new fine-
  tune requires a new probe run.
```

### 2.3 `report/manifest.json`

```json
{
  "report_version": "1.0.0",
  "slice_id": "<slice-id>",
  "slice_version": "<slice-version>",
  "published_at_utc": "2026-11-25T18:00:00Z",
  "artifacts": [
    {"path": "lawful_basis/GDPR_LAWFUL_BASIS.md",        "sha256": "<hash>", "type": "lawful_basis_memo"},
    {"path": "lawful_basis/HIPAA_DEIDENTIFICATION.md",   "sha256": "<hash>", "type": "hipaa_memo"},
    {"path": "lawful_basis/admit_policy.yaml",           "sha256": "<hash>", "type": "admit_policy"},
    {"path": "lawful_basis/sign_off.yaml",               "sha256": "<hash>", "type": "sign_off"},
    {"path": "scanner/presidio_pipeline.yaml",           "sha256": "<hash>", "type": "scanner_config"},
    {"path": "scanner/pipeline_manifest.yaml",           "sha256": "<hash>", "type": "scanner_run_manifest"},
    {"path": "scanner/findings.parquet.sha256",          "sha256": "<hash>", "type": "findings_pointer"},
    {"path": "scanner/calibration.md",                   "sha256": "<hash>", "type": "calibration"},
    {"path": "scanner/micro_benchmark.jsonl.sha256",     "sha256": "<hash>", "type": "micro_benchmark_pointer"},
    {"path": "redaction/decision_table.md",              "sha256": "<hash>", "type": "decision_table"},
    {"path": "redaction/rationale.md",                   "sha256": "<hash>", "type": "rationale"},
    {"path": "redaction/operator_tables/UC-A-pretrain-sft.yaml", "sha256": "<hash>", "type": "operator_table"},
    {"path": "extraction/extraction_report.yaml",        "sha256": "<hash>", "type": "extraction_report"},
    {"path": "extraction/injection_manifest.yaml",       "sha256": "<hash>", "type": "injection_manifest"},
    {"path": "extraction/canary_set.jsonl.sha256",       "sha256": "<hash>", "type": "canary_set_pointer"},
    {"path": "extraction/incidents_summary.yaml",        "sha256": "<hash>", "type": "incidents_summary"}
  ]
}
```

### 2.4 `report/reproduce.sh`

```bash
#!/usr/bin/env bash
# report/reproduce.sh
# Auditor-facing reproduction script. Idempotent: running it twice
# produces identical output except for runtime logs written to /tmp.
# Exits non-zero on any check failure.
set -euo pipefail
cd "$(dirname "$0")"

echo "[1/6] verifying manifest signature …"
cosign verify-blob \
  --key cosign.pub \
  --signature manifest.json.sig \
  manifest.json

echo "[2/6] verifying every artifact's SHA-256 matches the manifest …"
python - <<'PY'
import hashlib, json, pathlib, sys
m = json.loads(pathlib.Path("manifest.json").read_text())
failed = []
for a in m["artifacts"]:
    p = pathlib.Path(a["path"])
    got = hashlib.sha256(p.read_bytes()).hexdigest()
    if got != a["sha256"]:
        failed.append((str(p), a["sha256"], got))
if failed:
    for p, want, got in failed:
        print(f"MISMATCH {p}: want {want}, got {got}")
    sys.exit(1)
print("all artifact hashes match manifest")
PY

echo "[3/6] replaying scanner over the micro-benchmark canary …"
python code/replay_scanner.py \
  --config scanner/presidio_pipeline.yaml \
  --slice data/micro_benchmark_canary.jsonl \
  --expected data/micro_benchmark_canary_expected.json

echo "[4/6] verifying operator table loads in Presidio anonymizer …"
python code/validate_operator_table.py \
  --table redaction/operator_tables/UC-A-pretrain-sft.yaml

echo "[5/6] verifying sign_off.yaml document hashes match the on-disk files …"
python code/verify_sign_off.py \
  --sign-off lawful_basis/sign_off.yaml

echo "[6/6] summarising extraction report consistency …"
python code/verify_extraction_report.py \
  --report extraction/extraction_report.yaml \
  --injection-manifest extraction/injection_manifest.yaml

echo "OK"
```

Two design notes defensible at review:

- **The micro-benchmark canary file.** A tiny public jsonl file of
  known-positive and known-negative spans ships with the report. The
  reproduction script reruns the scanner against this file; if the
  scanner no longer flags the known-positive spans or flags the
  known-negatives, the published scanner config has drifted or the
  Presidio version in the reader's environment differs from the
  pipeline's pinned version. This is a smoke test, not a full
  re-scan.
- **Idempotence.** `reproduce.sh` must not mutate anything under
  `report/`. The script writes to `/tmp` only.

### 2.5 `report/VERIFY.md`

```markdown
# Verify this privacy risk report

## 5-minute check

1. `bash reproduce.sh` exits 0.
2. `grep "Report version" PRIVACY_RISK_REPORT.md` matches
   `jq -r '.report_version' manifest.json`.
3. `jq -r '.artifacts[].path' manifest.json | sort | uniq -d` is
   empty.

## Deeper checks (30 minutes)

4. Inspect `lawful_basis/sign_off.yaml` — confirm every required
   role is signed and every document SHA-256 matches the file on
   disk.
5. Inspect `scanner/calibration.md` — confirm the per-entity
   precision / recall numbers in `PRIVACY_RISK_REPORT.md` §3 match
   the calibration file.
6. Inspect `redaction/decision_table.md` — confirm the operator
   table used in §4 matches the decision table's UC-A row.
7. Inspect `extraction/extraction_report.yaml` — confirm the
   per-family rates in `PRIVACY_RISK_REPORT.md` §5 match the
   extraction report.
8. Inspect `extraction/incidents_summary.yaml` — confirm
   `unexpected_pii_hit_count` is zero or that every open ticket is
   named with a resolution.

## What this report does NOT claim

- That the redacted corpus is anonymous within the meaning of GDPR
  Recital 26.
- Freedom from extraction against attacker configurations not
  listed in Exercise 04's probe design.
- Freedom from membership inference against models that are not
  the one named in `extraction/extraction_report.yaml`.
- Any claim about downstream fine-tunes or RLHF data.
```

### 2.6 `code/build_manifest.py`

```python
# code/build_manifest.py
from __future__ import annotations

import argparse
import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path


ARTIFACT_TYPES = {
    "lawful_basis/GDPR_LAWFUL_BASIS.md":                      "lawful_basis_memo",
    "lawful_basis/HIPAA_DEIDENTIFICATION.md":                 "hipaa_memo",
    "lawful_basis/admit_policy.yaml":                         "admit_policy",
    "lawful_basis/sign_off.yaml":                             "sign_off",
    "scanner/presidio_pipeline.yaml":                         "scanner_config",
    "scanner/pipeline_manifest.yaml":                         "scanner_run_manifest",
    "scanner/findings.parquet.sha256":                        "findings_pointer",
    "scanner/calibration.md":                                 "calibration",
    "scanner/micro_benchmark.jsonl.sha256":                   "micro_benchmark_pointer",
    "redaction/decision_table.md":                            "decision_table",
    "redaction/rationale.md":                                 "rationale",
    "redaction/operator_tables/UC-A-pretrain-sft.yaml":       "operator_table",
    "extraction/extraction_report.yaml":                      "extraction_report",
    "extraction/injection_manifest.yaml":                     "injection_manifest",
    "extraction/canary_set.jsonl.sha256":                     "canary_set_pointer",
    "extraction/incidents_summary.yaml":                      "incidents_summary",
}


def _sha256(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--report-root", required=True, type=Path)
    ap.add_argument("--slice-id", required=True)
    ap.add_argument("--slice-version", required=True)
    ap.add_argument("--report-version", required=True)
    args = ap.parse_args()

    artifacts = []
    for rel_path, type_ in sorted(ARTIFACT_TYPES.items()):
        p = args.report_root / rel_path
        artifacts.append({
            "path":   rel_path,
            "sha256": _sha256(p),
            "type":   type_,
        })

    manifest = {
        "report_version":   args.report_version,
        "slice_id":         args.slice_id,
        "slice_version":    args.slice_version,
        "published_at_utc": datetime.now(timezone.utc)
                              .isoformat(timespec="seconds"),
        "artifacts":        artifacts,
    }
    (args.report_root / "manifest.json").write_text(
        json.dumps(manifest, indent=2, sort_keys=True) + "\n"
    )


if __name__ == "__main__":
    main()
```

### 2.7 `Makefile`

```makefile
.PHONY: all collect manifest render sign verify clean

all: collect manifest render sign verify

collect:
	python code/collect_artifacts.py --report-root report/

manifest:
	python code/build_manifest.py --report-root report/ \
		--slice-id slice-example-v1 --slice-version 1.0.0 \
		--report-version 1.0.0

render:
	python code/render_report_md.py \
		--report-root report/ --output PRIVACY_RISK_REPORT.md

sign:
	cosign sign-blob --yes --key "$$COSIGN_KEY" \
		--output-signature report/manifest.json.sig \
		report/manifest.json

verify:
	bash report/reproduce.sh

clean:
	rm -rf report/manifest.json report/manifest.json.sig PRIVACY_RISK_REPORT.md
```

### 2.8 `extraction/incidents_summary.yaml`

```yaml
# extraction/incidents_summary.yaml
# Public-safe summary of unexpected real-PII hits from the
# extraction probe. Counts only — matched strings are internal.

run_id: probe-slice-example-v1-20261125
total_unexpected_hits: 0
open_tickets: []
closed_tickets: []

# Example of a resolved ticket row (shape, not contents):
# closed_tickets:
#   - ticket_id: PRIVACY-1234
#     opened_at_utc: 2026-11-20T09:00:00Z
#     closed_at_utc: 2026-11-22T17:00:00Z
#     resolution: "scanner-recall-regression; patched EmailRecognizer
#                   context terms; re-scan passed."
#     resolution_pointer: scanner/calibration.md#2026-11-22-patch
```

## 3. Validation steps

1. **`reproduce.sh` exits 0 from a clean clone.** Fresh clone, run
   `cd report && bash reproduce.sh`; the script must succeed without
   network access except for the cosign public-key lookup.
2. **Every artifact in `manifest.json` exists and hashes match.**
   `code/reproduce_sanity.py` verifies this locally before publish.
3. **No unreferenced files under `report/`.** `find report/ -type
   f` is a subset of the files listed in `manifest.json` plus
   `reproduce.sh`, `VERIFY.md`, `manifest.json.sig`, and
   `cosign.pub`. Extra files are unsigned artifacts and fail
   review.
4. **Signature chain verifies.** `cosign verify-blob` succeeds for
   `manifest.json.sig`.
5. **Determinism (modulo `published_at_utc`).** Running `make
   manifest render` twice with the same artifact set produces
   byte-identical `manifest.json` and `PRIVACY_RISK_REPORT.md`
   except for the `published_at_utc` field.
6. **Scanner replay round-trip.** The reproduction micro-benchmark
   canary file contains labelled positive and negative spans; the
   scanner must flag every known-positive and no known-negative
   when run against the pinned config.
7. **Sign-off document hashes match.** `code/verify_sign_off.py`
   recomputes the SHA-256 of every file named in `sign_off.yaml`;
   a drift fails the pre-publish check.
8. **Numbers in the markdown match the YAML sources.** Per-entity
   precision / recall numbers in `PRIVACY_RISK_REPORT.md` §3 are
   generated from `scanner/calibration.md`; extraction rates in §5
   are generated from `extraction/extraction_report.yaml`. The
   render script does not hand-type numbers.
9. **Scope block consistency.** The §1 and §7 scope blocks in
   `PRIVACY_RISK_REPORT.md` match `VERIFY.md`'s "does NOT claim"
   section.
10. **Linter runs clean.** Markdown lint on
    `PRIVACY_RISK_REPORT.md` and `VERIFY.md` has zero errors;
    broken relative links fail the lint.
11. **Incidents resolution gate.** `extraction/incidents_summary.
    yaml` has either `total_unexpected_hits: 0` or an `open_tickets`
    list that is empty with every ticket moved to `closed_tickets`
    with a resolution.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros
on critical rows.

| #  | Dimension                                                                                                             | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `PRIVACY_RISK_REPORT.md` covers identity, lawful basis, scanner, redaction, extraction, reproduce, and scope          | Yes       |
| 2  | `manifest.json` lists every referenced artifact with path, sha256, and type                                           | Yes       |
| 3  | `manifest.json.sig` verifies against the published cosign public key                                                 | Yes       |
| 4  | `reproduce.sh` is idempotent, exits 0 from a clean clone, and writes only to `/tmp`                                   | Yes       |
| 5  | Scanner replay round-trip in the reproduction script (known-positive spans flagged, known-negative passed)            | Yes       |
| 6  | `lawful_basis/sign_off.yaml` document hashes match on-disk files                                                      | Yes       |
| 7  | Operator table in `redaction/operator_tables/` loads in Presidio anonymizer without error                             | Yes       |
| 8  | `extraction/incidents_summary.yaml` publishes counts but no matched strings                                          | Yes       |
| 9  | No unreferenced files under `report/`                                                                                 | Yes       |
| 10 | Numbers in §3 and §5 are generated from the YAML sources, not hand-edited                                             | Yes       |
| 11 | Scope block explicitly names what the report does NOT claim                                                           | Yes       |
| 12 | Report version, slice version, and bump rules are consistent across `PRIVACY_RISK_REPORT.md` and `manifest.json`      | Yes       |
| 13 | Governance anchors (NIST AI RMF GOVERN 1.4 / MEASURE 2.10, OWASP ML Top 10 ML02 / ML04) cited                        | No        |
| 14 | Cross-exercise provenance: Exercises 01–04 outputs are the artifacts, not re-authored content                        | Yes       |
| 15 | Unverifiable claims are marked `<!-- pending-verify: ... -->`                                                        | Yes       |

## 5. Common mistakes

- **Hand-editing numbers in the markdown.** If one precision or one
  extraction rate drifts from its YAML source, the report claims
  what the sources do not. Generate the tables; do not type them.
- **Publishing the report without the scanner micro-benchmark
  pointer.** Without the pinned micro-benchmark hash, the
  reproduction script cannot replay the scanner; "reproduce every
  number" becomes a hollow promise.
- **Signing `manifest.json` before the sign-off has been
  recomputed.** A `sign_off.yaml` whose document hashes drifted
  between manifest-build and sign-off breaks the chain of custody.
  Rebuild the manifest after any sign-off change.
- **Omitting the scope block.** A report without an explicit
  "does NOT claim …" section invites the auditor to read every
  claim as universal — including claims about downstream
  fine-tunes the report does not cover.
- **Publishing unexpected-PII strings.** `incidents_summary.yaml`
  must carry counts and ticket pointers only. The matched strings
  are the privacy liability the report is designed to remediate.
- **Treating the extraction probe as optional.** Without §5, the
  report is a promise, not evidence. Even a report whose
  extraction rate is zero must publish the number.
- **Bundling the model card and the risk report.** The model card
  references the risk report; the risk report does not duplicate
  the model card. Separating the two keeps the risk report's
  version-bump surface clean.
- **Allowing network access in `reproduce.sh`.** Any step that
  reaches the internet (other than key discovery) is a hidden
  dependency that can expire.
- **Shipping `published_at_utc` as a semver component.** The
  publication timestamp is metadata; the `report_version` is the
  SemVer identifier.
- **Omitting the pipeline reviewer's email from the identity
  block.** The report is a joint sign-off; the reviewer must be
  named.
- **Claiming anonymity.** The report makes no claim that the
  redacted corpus is anonymous within the meaning of GDPR
  Recital 26. Redaction is defence-in-depth; lawful basis is
  load-bearing.
- **Reusing an older risk report after a scanner-config bump.** A
  change to `recipes/presidio_pipeline.yaml` forces a slice-version
  bump and a fresh report. A stale report's claims are no longer
  supported.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack / ML04:2023 Membership Inference Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchors for the slice's privacy posture — ML02 for
  the memorisation surface the probe measures; ML04 for the
  membership-inference surface redaction and lawful basis close.
- **MITRE ATLAS:** <https://atlas.mitre.org/>. The attacker
  technique index; the report's defender-side evidence is the
  compensating control that makes those techniques reviewable.
- **NIST AI Risk Management Framework — GOVERN 1.4, MEASURE 2.10:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN
  1.4 ("processes and personnel are in place to provide
  transparency and explain system outcomes") is the primary anchor
  for publishing a privacy risk report. MEASURE 2.10 anchors the
  privacy-risk characterisation across the slice's lifecycle.
- **GDPR Article 35 — Data Protection Impact Assessment (DPIA):**
  <https://gdpr-info.eu/art-35-gdpr/>. The DPIA excerpt the
  lawful-basis section references; the privacy risk report is the
  downstream artifact the DPIA points at.
- **GDPR Recital 26 — Anonymous information:**
  <https://gdpr-info.eu/recitals/no-26/>. The recital that
  distinguishes anonymous data from pseudonymous / redacted data;
  the report's scope block is explicit about this distinction.
- **HIPAA Privacy Rule — 45 CFR 164.514(b):**
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>.
  The regulatory anchor for the Safe Harbor / Expert Determination
  pathway the HIPAA section references.
- **Sigstore / cosign:** <https://docs.sigstore.dev/>. The
  signature primitives `reproduce.sh` uses.
- **Carlini et al., Extracting Training Data from Large Language
  Models, USENIX Security 2021, arXiv:2012.07805:**
  <https://arxiv.org/abs/2012.07805>. The empirical basis for the
  extraction-rate reporting shape in §5.
- **Local anchors:** Exercise 01 (the scanner whose outputs the
  report republishes); Exercise 02 (the operator tables the report
  embeds); Exercise 03 (the lawful-basis memos the report
  embeds); Exercise 04 (the extraction probe the report
  summarises); `mod-105` exercise-02 (the dedup posture the F2
  curve empirically measures); `mod-109` (the module that signs
  the report's root manifest and teaches signing root-of-trust
  rotations); `mod-104` exercise-01 (the Great Expectations
  suite over the redacted corpus, which is an upstream input to
  the scanner coverage numbers in §3).
