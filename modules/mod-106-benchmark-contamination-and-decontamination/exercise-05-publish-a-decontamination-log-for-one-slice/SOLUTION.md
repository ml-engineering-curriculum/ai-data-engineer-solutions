# Exercise 05 — Publish a Decontamination Log for One Slice (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner publishes the
public-facing decontamination log for one slice — the aggregate of
Exercises 01–04 structured as a reproducible, signed, auditor-ready
artifact that ships with the model card. The log is the outward
face of `mod-106`: everything inside the slice's firewall can be
inspected by someone who was not in the room when the slice was
built.

A decontamination log is **not** a marketing document. It is a
reproducibility packet — the external auditor's instructions for
reproducing every number the slice claims. If the auditor runs the
packet and gets different numbers, the log has failed its purpose
regardless of how well-written it is.

Deliverables:

- `DECONTAMINATION_LOG.md` — the top-level public document. Four
  sections: identity, firewall configuration, results per eval
  suite, and a reproduction block.
- `log/manifest.json` — the machine-readable index of every artifact
  referenced in the log, each with a SHA-256 and a URL.
- `log/manifest.json.sig` — cosign signature of the manifest.
- `log/decontamination_report.yaml` — Exercises 01–04's combined
  report; the `§1` through `§9` blocks all populated.
- `log/eval_index.parquet` + `log/eval_index.parquet.sha256` —
  the frozen eval index the firewall scanned against.
- `log/quarantine/` — the per-verdict directories from Exercise 01
  (`contaminated`, `quarantine_review`) plus the signed manifest.
- `log/rebuttal/rebuttal.md` + `log/rebuttal/test_plan.md` —
  Exercise 03's rebuttal brief and the pre-registered test plan.
- `log/synthetic/CONTRACT.md` +
  `log/synthetic/firewall_manifest.schema.json` + at least one
  per-batch manifest — the public face of Exercise 04.
- `log/reproduce.sh` — the auditor's one-shot reproduction script.
- `log/VERIFY.md` — the auditor-facing verification checklist.

Governance framing. The decontamination log is a transparency
artifact — publishing it is what makes an evaluation claim
**auditable**. The NIST AI RMF anchor is **GOVERN 1.4** (processes
and personnel are in place to provide transparency and explain
system outcomes) and **MEASURE 2.8** (risks and potential impacts
are informed by input from domain experts and other relevant AI
actors to capture risks that are not identified by developers).
The OWASP ML Top 10 anchor remains **ML02:2023 Data Poisoning
Attack** — the published log is the compensating control that
establishes the slice's contamination posture. MITRE ATLAS does not
specify a defender technique; the log is the artifact that makes
defender techniques reviewable.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── DECONTAMINATION_LOG.md
├── Makefile
├── code/
│   ├── collect_artifacts.py       # copy Ex 01–04 outputs under log/
│   ├── build_manifest.py          # emit log/manifest.json
│   ├── render_log_md.py           # render DECONTAMINATION_LOG.md from report + manifest
│   └── reproduce_sanity.py        # local pre-publish verifier
├── log/
│   ├── manifest.json
│   ├── manifest.json.sig
│   ├── decontamination_report.yaml
│   ├── eval_index.parquet
│   ├── eval_index.parquet.sha256
│   ├── quarantine/
│   │   ├── contaminated/
│   │   ├── quarantine_review/
│   │   ├── quarantine_manifest.json
│   │   └── quarantine_manifest.json.sig
│   ├── rebuttal/
│   │   ├── test_plan.md
│   │   ├── rebuttal.md
│   │   └── min_k_prob_scores.parquet
│   ├── synthetic/
│   │   ├── CONTRACT.md
│   │   ├── firewall_manifest.schema.json
│   │   └── batches/
│   │       └── synth-example-2026W46-01.json
│   ├── reproduce.sh
│   └── VERIFY.md
└── tests/
    ├── test_manifest_shas_match.py
    ├── test_signatures_verify.py
    └── test_reproduce_sh_exits_zero.py
```

### 2.2 `DECONTAMINATION_LOG.md`

```markdown
# Decontamination Log — slice `<slice-id>` v<slice-version>

| Field                     | Value                                                |
|---------------------------|------------------------------------------------------|
| Slice id                  | <slice-id>                                           |
| Slice version             | <slice-version>                                      |
| Published                 | 2026-11-17                                           |
| Operator                  | <operator email>                                     |
| Peer reviewer             | <risk-engineer email>                                |
| Log version               | 1.0.0                                                |
| Signing key fingerprint   | <cosign fingerprint>                                 |

## 1. What this log covers

This log publishes the evidence that eval suites `suite_a`,
`suite_b`, and `suite_c` were held outside the training mixture
of slice `<slice-id>`. It covers:

1. The firewall configuration (Exercise 01).
2. The lm-evaluation-harness / dolma recipe runs (Exercise 02).
3. The Min-K% Prob and exchangeability-test rebuttal (Exercise 03).
4. The synthetic-data firewall contract and per-batch manifests
   (Exercise 04).

It does not cover eval suites that were deliberately included in
training (`suite_d` in this slice's firewall configuration; see
Exercise 01 `policy_note.md` for the exclusion rationale).

## 2. Firewall configuration

| Parameter               | Value                       |
|-------------------------|-----------------------------|
| `N`                     | 13                          |
| Canonical form          | NFC + collapse-whitespace + preserve-case |
| Canonical form version  | nfc-ws-collapse-v1          |
| Fingerprint algorithm   | xxh3-128                    |
| Firewall verdict rule   | `clean` / `quarantine_review` / `contaminated` |
| Quarantine threshold T  | 1 matching eval n-gram      |

Eval index content hash:
`<sha256 of log/eval_index.parquet>`.
Policy note content hash:
`<sha256 of log/policy_note.md>`.

## 3. Results per eval suite

Each row is produced from `log/decontamination_report.yaml`'s
`§3 per_eval_suite_match_counts` block. Columns are
machine-generated; no row is hand-edited.

| Suite    | Firewall matched docs | Firewall match rate | Harness recipe matched docs | Dolma recipe matched docs | Rebuttal verdict |
|----------|-----------------------|---------------------|-----------------------------|---------------------------|------------------|
| suite_a  | <n>                   | <rate>              | <n>                         | <n>                       | `cleared`        |
| suite_b  | <n>                   | <rate>              | <n>                         | <n>                       | `cleared`        |
| suite_c  | <n>                   | <rate>              | <n>                         | <n>                       | `cleared`        |

A row with `rebuttal verdict: pending_review` is a row the auditor
should treat as unresolved — `cleared` requires both the n-gram
firewall and the detector rebuttal to be signed by the peer.

## 4. Synthetic channel

Contract version `1.0.0` governs the synthetic-data channel for
this slice. `N` batches were ingested under the contract; every
batch's manifest hash and the consumer-side firewall verdict are in
`log/synthetic/batches/`.

Batch index:

| Batch id                           | Producer-dropped docs | Consumer-matched docs | Verdict |
|------------------------------------|------------------------|-----------------------|---------|
| synth-example-2026W46-01           | 42                     | 0                     | clean   |
| …                                  | …                      | …                     | …       |

## 5. How to reproduce every number in this log

```bash
cd log/
./reproduce.sh
```

`reproduce.sh` verifies every signature, hashes every artifact, and
re-runs the firewall scan over a public canary document to confirm
the eval index behaves as published. See `log/VERIFY.md` for the
auditor-facing step-by-step.

## 6. Scope and expiry

- This log is valid until a change to any item in the "firewall
  configuration" table above forces a slice-version bump, at which
  point a new log is published alongside the new slice.
- The log covers the slice's training mixture. It does **not** make
  claims about downstream fine-tunes or RLHF data.
- Detector rebuttals are scoped to the target model id recorded in
  Exercise 03's `test_plan.md`. A new fine-tune requires a new
  rebuttal.
```

### 2.3 `log/manifest.json`

```json
{
  "log_version": "1.0.0",
  "slice_id": "<slice-id>",
  "slice_version": "<slice-version>",
  "published_at_utc": "2026-11-17T18:00:00Z",
  "artifacts": [
    {
      "path": "decontamination_report.yaml",
      "sha256": "<hash>",
      "type": "report"
    },
    {
      "path": "eval_index.parquet",
      "sha256": "<hash>",
      "type": "eval_index"
    },
    {
      "path": "quarantine/quarantine_manifest.json",
      "sha256": "<hash>",
      "signature_path": "quarantine/quarantine_manifest.json.sig",
      "type": "quarantine_manifest"
    },
    {
      "path": "rebuttal/test_plan.md",
      "sha256": "<hash>",
      "type": "test_plan"
    },
    {
      "path": "rebuttal/rebuttal.md",
      "sha256": "<hash>",
      "type": "rebuttal"
    },
    {
      "path": "synthetic/CONTRACT.md",
      "sha256": "<hash>",
      "type": "contract"
    },
    {
      "path": "synthetic/firewall_manifest.schema.json",
      "sha256": "<hash>",
      "type": "contract_schema"
    }
  ]
}
```

`manifest.json` is **the index of record** for the log. Every file
referenced in `DECONTAMINATION_LOG.md` must be in the manifest and
every file in the manifest must exist at the recorded path with the
recorded SHA-256.

### 2.4 `log/reproduce.sh`

```bash
#!/usr/bin/env bash
# log/reproduce.sh
# Auditor-facing reproduction script. Idempotent: running it twice
# produces identical output. Exits non-zero on any check failure.
set -euo pipefail
cd "$(dirname "$0")"

echo "[1/5] verifying manifest signature …"
cosign verify-blob \
  --key cosign.pub \
  --signature manifest.json.sig \
  manifest.json

echo "[2/5] verifying quarantine signature …"
cosign verify-blob \
  --key cosign.pub \
  --signature quarantine/quarantine_manifest.json.sig \
  quarantine/quarantine_manifest.json

echo "[3/5] verifying every artifact's SHA-256 matches the manifest …"
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

echo "[4/5] rerunning firewall over the canary corpus …"
python code/reproduce_canary.py \
  --eval-index eval_index.parquet \
  --canary data/canaries.jsonl \
  --expect-contaminated data/canaries_expected.json

echo "[5/5] rerunning exchangeability test with the recorded seed …"
python code/exchangeability_test.py \
  --records rebuttal/exchangeability_input.jsonl \
  --seed "$(jq -r '.exchangeability_test.seed' rebuttal/rebuttal.json)" \
  --output /tmp/exch_out.json
python code/diff_exchangeability.py \
  --expected rebuttal/exchangeability_expected.json \
  --actual /tmp/exch_out.json

echo "OK"
```

Two design notes defensible at review:

- **The canary corpus.** A tiny public file of known-contaminated
  and known-clean documents ships with the log. The reproduction
  script reruns the firewall against this corpus; if the firewall
  no longer flags the known-contaminated canary, the published
  eval index is corrupted or the firewall implementation has
  drifted. This is a smoke test, not a full re-scan.
- **Idempotence.** `reproduce.sh` must not mutate anything under
  `log/`. The script writes to `/tmp` only.

### 2.5 `log/VERIFY.md` — the auditor checklist

```markdown
# Verify this decontamination log

## 5-minute check

1. `bash reproduce.sh` exits 0.
2. `grep "Log version" DECONTAMINATION_LOG.md` matches
   `jq -r '.log_version' manifest.json`.
3. `jq -r '.artifacts[].path' manifest.json | sort | uniq -d` is empty.

## Deeper checks (30 minutes)

4. Inspect `quarantine/contaminated/` — spot-check that each
   document matches an eval record in `eval_index.parquet`.
5. Inspect `quarantine/quarantine_review/` — spot-check the peer
   review verdict in `decontamination_report.yaml`'s
   `§6 ai_risk_engineer_review` block.
6. Inspect `rebuttal/test_plan.md` — confirm its commit timestamp
   pre-dates every timestamp in `rebuttal/rebuttal.md`.
7. Inspect `synthetic/batches/*.json` — pick one, independently
   validate against `synthetic/firewall_manifest.schema.json`.

## What this log does NOT claim

- Freedom from contamination against detectors not listed in
  Exercise 03's test plan.
- Decontamination of eval suites not listed in Exercise 01's
  firewall configuration.
- Any claim about downstream fine-tunes.
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


def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


ARTIFACT_TYPES = {
    "decontamination_report.yaml":                     "report",
    "eval_index.parquet":                              "eval_index",
    "quarantine/quarantine_manifest.json":             "quarantine_manifest",
    "rebuttal/test_plan.md":                           "test_plan",
    "rebuttal/rebuttal.md":                            "rebuttal",
    "synthetic/CONTRACT.md":                           "contract",
    "synthetic/firewall_manifest.schema.json":         "contract_schema",
}


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--log-root", required=True, type=Path)
    ap.add_argument("--slice-id", required=True)
    ap.add_argument("--slice-version", required=True)
    ap.add_argument("--log-version", required=True)
    args = ap.parse_args()

    artifacts = []
    for rel_path, type_ in sorted(ARTIFACT_TYPES.items()):
        p = args.log_root / rel_path
        entry = {
            "path": rel_path,
            "sha256": sha256_file(p),
            "type": type_,
        }
        sig = p.with_suffix(p.suffix + ".sig")
        if sig.exists():
            entry["signature_path"] = str(sig.relative_to(args.log_root))
        artifacts.append(entry)

    # Synthetic per-batch manifests (variable count).
    batches_dir = args.log_root / "synthetic" / "batches"
    for batch_json in sorted(batches_dir.glob("*.json")):
        artifacts.append({
            "path": str(batch_json.relative_to(args.log_root)),
            "sha256": sha256_file(batch_json),
            "type": "synthetic_batch_manifest",
        })

    manifest = {
        "log_version": args.log_version,
        "slice_id": args.slice_id,
        "slice_version": args.slice_version,
        "published_at_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "artifacts": artifacts,
    }
    (args.log_root / "manifest.json").write_text(
        json.dumps(manifest, indent=2, sort_keys=True) + "\n"
    )


if __name__ == "__main__":
    main()
```

Determinism safeguards:

- **`sort_keys=True` + trailing newline**: byte-identical output on
  re-runs for byte-identical inputs. The `published_at_utc` field is
  the single non-deterministic entry; §3 of validation covers how
  to handle it.
- **Sorted iteration everywhere** (`sorted(ARTIFACT_TYPES.items())`,
  `sorted(batches_dir.glob(...))`) — Python dict iteration is
  insertion-ordered but glob is filesystem-ordered, so sort
  explicitly.

### 2.7 `Makefile`

```makefile
.PHONY: all collect manifest render sign verify clean

all: collect manifest render sign verify

collect:
	python code/collect_artifacts.py --log-root log/

manifest:
	python code/build_manifest.py --log-root log/ \
		--slice-id slice-example-v1 --slice-version 1.0.0 \
		--log-version 1.0.0

render:
	python code/render_log_md.py \
		--log-root log/ --output DECONTAMINATION_LOG.md

sign:
	cosign sign-blob --yes --key "$$COSIGN_KEY" \
		--output-signature log/manifest.json.sig \
		log/manifest.json

verify:
	bash log/reproduce.sh

clean:
	rm -rf log/manifest.json log/manifest.json.sig DECONTAMINATION_LOG.md
```

## 3. Validation steps

1. **`reproduce.sh` exits 0 from a clean clone.** Fresh clone, run
   `cd log && bash reproduce.sh`; the script must succeed without
   network access except for the cosign public-key lookup.
2. **Every artifact in `manifest.json` exists and hashes match.**
   `code/reproduce_sanity.py` verifies this locally before publish.
3. **No unreferenced files under `log/`.** `find log/ -type f` is a
   subset of the files listed in `manifest.json` plus signatures
   plus `reproduce.sh` plus `VERIFY.md` plus `cosign.pub`. Extra
   files are unsigned artifacts and fail review.
4. **Signature chain verifies.** `cosign verify-blob` succeeds for
   `manifest.json.sig` and for `quarantine_manifest.json.sig`.
5. **Determinism (modulo `published_at_utc`).** Running `make
   manifest render` twice with the same artifact set produces
   byte-identical `manifest.json` and `DECONTAMINATION_LOG.md`
   except for the `published_at_utc` field.
6. **Canary round-trip.** The reproduction canary corpus contains
   one known-contaminated and one known-clean document; the firewall
   must flag the first and not flag the second when run against the
   published eval index.
7. **Rebuttal determinism.** Re-running the exchangeability test
   with the published seed reproduces the p-value in
   `rebuttal/rebuttal.md`.
8. **No claims outside the log's scope.** The §6 scope block in
   `DECONTAMINATION_LOG.md` matches the suites listed in Exercise
   01's `policy_note.md`; a new suite in the firewall but missing
   from the log fails this check.
9. **Linter runs clean.** Markdown lint on `DECONTAMINATION_LOG.md`
   and `VERIFY.md` has zero errors; broken relative links fail.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                                 | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `DECONTAMINATION_LOG.md` covers identity, firewall config, per-suite results, synthetic channel, reproduce, and scope      | Yes       |
| 2  | `manifest.json` lists every referenced artifact with path, sha256, and signature path where applicable                     | Yes       |
| 3  | `manifest.json.sig` verifies against the published cosign public key                                                      | Yes       |
| 4  | `reproduce.sh` is idempotent, exits 0 from a clean clone, and writes only to `/tmp`                                        | Yes       |
| 5  | Canary round-trip (known-contaminated flagged, known-clean passed) in the reproduction script                             | Yes       |
| 6  | Rebuttal determinism: fixed seed re-runs the exchangeability test and reproduces the published p-value                    | Yes       |
| 7  | No unreferenced files under `log/`                                                                                         | Yes       |
| 8  | Per-suite table is machine-generated from `decontamination_report.yaml`, not hand-edited                                  | Yes       |
| 9  | Scope block explicitly names what the log does NOT claim                                                                  | Yes       |
| 10 | Log version, slice version, and firewall configuration bump rules are consistent across `DECONTAMINATION_LOG.md` and `manifest.json` | Yes       |
| 11 | Governance anchors (NIST AI RMF GOVERN 1.4 / MEASURE 2.8, OWASP ML Top 10 ML02) cited                                      | No        |

## 5. Common mistakes

- **Hand-editing the per-suite table.** If one number drifts from
  `decontamination_report.yaml`, the log claims what the report
  does not. Generate the table; do not type it.
- **Publishing the log without the eval index.** The eval index is
  the hash the reproduction script re-runs against; without it,
  "reproduce every number" is a hollow promise.
- **Signing `manifest.json` before the quarantine manifest.** A
  signature over a manifest whose entries point at unsigned
  quarantine data creates a chain with a broken link.
- **Omitting the scope block.** A log without an explicit "this
  log does NOT claim …" section invites the auditor to read every
  claim as universal.
- **Burying the reproduction script in a long README.** The
  auditor runs `reproduce.sh`; a buried step-by-step inside prose
  will be skipped and the auditor will conclude the log is
  unreproducible.
- **Treating the canary corpus as optional.** Without a canary
  round-trip, the reproduction only verifies hashes — it does not
  verify that the firewall still functions.
- **Publishing the log without a scope-limit on the rebuttal.** A
  rebuttal is valid for one target-model id. A log that forgets
  this invites readers to extrapolate to other models.
- **Shipping `published_at_utc` as a semver component.** The
  publication timestamp is metadata; the `log_version` is the
  SemVer identifier.
- **Omitting the peer reviewer's email from the identity block.**
  The log is a joint sign-off; the peer must be named.
- **Allowing network access in `reproduce.sh`.** Any step that
  reaches the internet (other than key discovery) is a hidden
  dependency that can expire.

## 6. References

- **NIST AI Risk Management Framework — GOVERN 1.4, MEASURE 2.8:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN
  1.4 ("processes and personnel are in place to provide
  transparency and explain system outcomes") is the primary anchor
  for publishing a decontamination log. MEASURE 2.8 anchors the
  external-input part of the review cadence.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Publishing the log is the compensating control that makes the
  slice's contamination posture reviewable.
- **MITRE ATLAS:** <https://atlas.mitre.org/>. ATLAS lists
  attacker-facing techniques; the decontamination log is the
  defender's artifact.
- **Sigstore / cosign:** <https://docs.sigstore.dev/>.
  `cosign sign-blob` and `cosign verify-blob` are the
  signature-chain primitives the log uses.
- **JSON Schema Draft 2020-12:** <https://json-schema.org/draft/2020-12>.
  The dialect Exercise 04's manifest schema targets.
- **Local anchors:** Exercise 01 (the firewall whose outputs the
  log republishes); Exercise 02 (the recipe-run outputs the log
  links to); Exercise 03 (the detector rebuttal the log embeds);
  Exercise 04 (the synthetic-channel contract the log cites);
  `mod-109` (the module that signs the log's root manifest and the
  module that teaches signing root-of-trust rotations).
