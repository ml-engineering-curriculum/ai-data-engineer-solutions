# Exercise 05 — Pipeline-Side Detection of Nightshade and Adversarial Perturbations (Solution)

## 1. Solution overview

This is a **design + implementation** exercise. The learner adds the
four pipeline-side detection heuristics from Chapter 07 on top of the
Exercise 04 dedup output, routes flagged content to a signed
quarantine bucket, and populates the `§5` adversarial-perturbation
review block of the Chapter 08 dedup report — including the
`ai_risk_engineer_review` sign-off slot that stays **`pending_review`**
until the peer actually signs it.

The *division of labour* is load-bearing and must be stated up front:

- **`ai-data-engineer` (you)** owns pipeline-side quarantine. You
  emit signals, route flagged content, produce audit evidence, and
  hand the briefing to the peer.
- **`ai-risk-engineer` (peer, level 25)** owns detection methodology.
  They choose detectors, run them, tune thresholds, and sign the §5
  verdict.

A `pending_review` verdict is a *complete* report at this layer; the
report becomes "complete enough to ship the slice" only when the peer
signs. Fabricating a sign-off is an acceptance-criterion failure and a
governance failure.

The four detection heuristics (Chapter 07 §"Pipeline-side detection
heuristics"):

1. **Cluster-size anomaly** per `(source, crawl_week)` at ≥ 3σ above
   per-source baseline.
2. **Source-level removal-rate delta** vs prior slice at ≥ 30
   percentage points.
3. **Zero-day clusters** whose every source is outside the historical
   source set.
4. **(Multimodal only) perturbation-signature column** per image: pHash,
   DCT high-band ratio, high-frequency noise energy summary. Explicitly
   a *feature set*, not a detection verdict.

Governance framing: this exercise is where the dedup report meets
**OWASP LLM Top 10 LLM04:2025 Data and Model Poisoning** (LLM03 in the
2023 edition), **OWASP ML Top 10 ML02:2023 Data Poisoning Attack**, and
**MITRE ATLAS AML.T0019 Publish Poisoned Datasets**, **AML.T0020 Poison
Training Data**, and **AML.T0051.001 LLM Prompt Injection via Training
Data**. NIST AI RMF **MEASURE 2.7** (security and resilience) and
**MANAGE 1.3** (risks tracked and documented) are the control anchors.
The quarantine manifest is in-scope for `mod-109` Sigstore signing.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── handoff_to_ai_risk_engineer.md
├── code/
│   ├── perturbation_signals.py         # the four heuristics
│   ├── multimodal_signatures.py        # optional, image-only
│   ├── quarantine.py                   # route + manifest
│   ├── emit_report.py                  # fills in §5
│   └── verify_signature.sh             # cosign verify command
├── historical/
│   ├── prior_slice_source_removal_rates.yaml
│   └── historical_sources.txt
└── results/
    ├── cluster_size_anomalies.parquet
    ├── source_removal_rate_deltas.parquet
    ├── zero_day_clusters.parquet
    ├── multimodal_signatures.parquet    # optional
    ├── quarantine/
    │   ├── cluster_size_anomaly/
    │   ├── source_rate_spike/
    │   └── zero_day_cluster/
    ├── quarantine_manifest.json
    ├── quarantine_manifest.json.sig
    └── dedup_report.yaml                # §5 populated, pending_review
```

### 2.2 `code/perturbation_signals.py`

```python
# code/perturbation_signals.py
# The four Chapter 07 pipeline-side heuristics.
# NOTE: these are SIGNALS, not detectors. Do not quarantine on your own
# authority beyond this pipeline-side role; ai-risk-engineer owns the
# detection methodology.
from __future__ import annotations

import argparse
import json
import math
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
import yaml

CLUSTER_SIGMA_THRESHOLD = 3.0
REMOVAL_RATE_DELTA_THRESHOLD = 0.30


# --- 1. Cluster-size anomaly ------------------------------------------

def cluster_size_anomalies(clusters_parquet: Path, output: Path) -> None:
    """Flag (source, crawl_week) cells whose max cluster size is >= 3σ
    above the per-source baseline mean."""
    df = pq.read_table(clusters_parquet).to_pandas()
    # Expand cluster → member rows with per-member source + crawl_week
    rows = []
    for _, row in df.iterrows():
        for m in row["members"]:
            rows.append(
                {
                    "cluster_id": row["cluster_id"],
                    "cluster_size": row["cluster_size"],
                    "source": m["source"],
                    "crawl_week": m["crawl_week"],
                }
            )
    import pandas as pd
    md = pd.DataFrame(rows)

    # Per-source baseline (mean, std) over all crawl weeks.
    baseline = md.groupby("source")["cluster_size"].agg(["mean", "std"]).reset_index()
    baseline["std"] = baseline["std"].fillna(0.0)

    # Per-cell max cluster size.
    cell_max = (
        md.groupby(["source", "crawl_week"])["cluster_size"].max().reset_index()
    )
    joined = cell_max.merge(baseline, on="source")
    joined["threshold"] = joined["mean"] + CLUSTER_SIGMA_THRESHOLD * joined["std"]
    flagged = joined[joined["cluster_size"] > joined["threshold"]].copy()
    flagged["heuristic"] = "cluster_size_anomaly"
    flagged["sigma_ratio"] = (flagged["cluster_size"] - flagged["mean"]) / \
        flagged["std"].replace(0, math.nan)

    pq.write_table(pa.Table.from_pandas(flagged), output)


# --- 2. Source-level removal-rate delta -------------------------------

def source_removal_rate_deltas(
    current_report: Path, prior_rates: Path, output: Path
) -> None:
    current = yaml.safe_load(current_report.read_text())
    prior = yaml.safe_load(prior_rates.read_text())

    rows = []
    cur = (current.get("near_dedup_document") or {}).get(
        "per_source_removal_rate", {}
    )
    for source, cur_rate in cur.items():
        prior_rate = prior.get(source)
        if prior_rate is None:
            continue
        delta = cur_rate - prior_rate
        if abs(delta) >= REMOVAL_RATE_DELTA_THRESHOLD:
            rows.append(
                {
                    "source": source,
                    "prior_rate": prior_rate,
                    "current_rate": cur_rate,
                    "delta": delta,
                    "heuristic": "source_rate_spike",
                }
            )
    pq.write_table(pa.Table.from_pylist(rows), output)


# --- 3. Zero-day clusters ---------------------------------------------

def zero_day_clusters(
    clusters_parquet: Path,
    historical_sources_path: Path,
    output: Path,
) -> None:
    hist = {
        s.strip()
        for s in historical_sources_path.read_text().splitlines()
        if s.strip() and not s.startswith("#")
    }
    clusters = pq.read_table(clusters_parquet).to_pandas()

    rows = []
    for _, row in clusters.iterrows():
        sources = {m["source"] for m in row["members"]}
        if sources and sources.isdisjoint(hist):
            rows.append(
                {
                    "cluster_id": row["cluster_id"],
                    "cluster_size": row["cluster_size"],
                    "sources": sorted(sources),
                    "heuristic": "zero_day_cluster",
                }
            )
    pq.write_table(pa.Table.from_pylist(rows), output)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--clusters", required=True, type=Path)
    ap.add_argument("--current-report", required=True, type=Path)
    ap.add_argument("--prior-rates", required=True, type=Path)
    ap.add_argument("--historical-sources", required=True, type=Path)
    ap.add_argument("--output-dir", required=True, type=Path)
    args = ap.parse_args()
    args.output_dir.mkdir(parents=True, exist_ok=True)

    cluster_size_anomalies(
        args.clusters, args.output_dir / "cluster_size_anomalies.parquet"
    )
    source_removal_rate_deltas(
        args.current_report,
        args.prior_rates,
        args.output_dir / "source_removal_rate_deltas.parquet",
    )
    zero_day_clusters(
        args.clusters,
        args.historical_sources,
        args.output_dir / "zero_day_clusters.parquet",
    )


if __name__ == "__main__":
    main()
```

Three design calls worth defending at review:

- **The 3σ and 30pp thresholds are starting points.** Chapter 07 is
  explicit that a heuristic that quarantines 10% of a clean slice is
  unusable; the exercise's stretch-goal measurement is that
  false-positive cost. Record the thresholds in `§5` of the report.
- **Zero-day uses set-disjointness.** A cluster where *every* member
  source is new to the pipeline's history. One known source in the
  cluster disqualifies it — that is the operational definition (strict,
  precision-biased).
- **Source-rate deltas in both directions.** A source that *drops*
  from 60% removal to 2% is also suspicious — it may have shifted its
  content strategy in response to your filters. Flag `abs(delta) ≥ 30pp`.

### 2.3 `code/multimodal_signatures.py` (optional, image slices)

```python
# code/multimodal_signatures.py
# Perturbation-signature FEATURE SET per image. NOT a Nightshade detector.
from __future__ import annotations

import io
from pathlib import Path

import imagehash
import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
from PIL import Image
from scipy.fftpack import dct


def _dct_high_band_ratio(gray: np.ndarray) -> float:
    """Fraction of DCT energy in the upper third of the frequency band.
    Known perturbation tools tend to concentrate energy there."""
    coef = dct(dct(gray, axis=0, norm="ortho"), axis=1, norm="ortho")
    power = coef ** 2
    h, w = power.shape
    hi = power[h // 3 :, w // 3 :].sum()
    total = power.sum()
    return float(hi / total) if total > 0 else 0.0


def _noise_energy(gray: np.ndarray) -> float:
    """High-frequency Laplacian energy; a crude signal for pixel-level
    perturbation. Compare against a natural-image baseline before
    interpreting."""
    from scipy.ndimage import laplace
    return float(np.mean(laplace(gray) ** 2))


def compute_signatures(image_bytes: bytes) -> dict:
    img = Image.open(io.BytesIO(image_bytes)).convert("L").resize((256, 256))
    gray = np.asarray(img, dtype=np.float32)
    return {
        "phash": str(imagehash.phash(Image.fromarray(gray.astype(np.uint8)))),
        "dct_high_band_ratio": _dct_high_band_ratio(gray),
        "noise_energy_high_freq": _noise_energy(gray),
    }
```

The parquet column header **must** state that this is a feature set,
not a verdict. The exercise's acceptance criterion is "explicitly state
in the dedup report that this column is a feature set, not a detection
verdict". Chapter 07: "You are not writing a Nightshade detector."

### 2.4 `code/quarantine.py`

```python
# code/quarantine.py
# Route flagged artifacts to quarantine/<reason>/ with a signed manifest.
# Does NOT silently drop.
from __future__ import annotations

import argparse
import hashlib
import json
import shutil
import subprocess
from datetime import datetime, timezone
from pathlib import Path

import pyarrow.parquet as pq


REASON_CODES = {
    "cluster_size_anomaly",
    "source_rate_spike",
    "zero_day_cluster",
    "multimodal_signature_outlier",
}


def sha256_file(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def route(
    source_root: Path,
    quarantine_root: Path,
    flagged: list[dict],
) -> list[dict]:
    """Copy (do not move) flagged artifacts into quarantine/<reason>/
    subprefixes. The source slice is preserved so dedup rows remain
    traceable; the quarantine subset is the ai-risk-engineer's review
    surface."""
    manifest_rows = []
    for item in flagged:
        reason = item["reason_code"]
        assert reason in REASON_CODES, f"unknown reason code: {reason}"
        src = source_root / item["artifact_path"]
        dst_dir = quarantine_root / reason
        dst_dir.mkdir(parents=True, exist_ok=True)
        dst = dst_dir / src.name
        shutil.copy2(src, dst)
        manifest_rows.append(
            {
                "artifact_id": item["artifact_id"],
                "original_path": str(src),
                "quarantine_path": str(dst),
                "reason_code": reason,
                "detector_version": item.get("detector_version", "heuristic-v1"),
                "evidence_sha256": sha256_file(src),
            }
        )
    return manifest_rows


def emit_manifest(
    slice_id: str, slice_version: str,
    manifest_rows: list[dict], output: Path,
) -> None:
    manifest = {
        "slice_id": slice_id,
        "slice_version": slice_version,
        "created_utc": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "quarantine_items": manifest_rows,
    }
    output.write_text(json.dumps(manifest, indent=2, sort_keys=True))


def sign_manifest(manifest: Path, key: Path | None = None) -> None:
    """cosign sign-blob the manifest. If no key is provided, emit a
    documented mock signature for development."""
    if key is None:
        # Dev shim: a mock signature is acceptable per the exercise
        # prerequisites; document the choice in the report.
        sig = hashlib.sha256(manifest.read_bytes()).hexdigest()
        (manifest.with_suffix(manifest.suffix + ".sig")).write_text(sig + "\n")
        return
    subprocess.run(
        [
            "cosign", "sign-blob",
            "--key", str(key),
            "--output-signature", str(manifest) + ".sig",
            str(manifest),
        ],
        check=True,
    )
```

Two things worth defending:

- **Copy, not move.** The quarantine bucket is a review surface; the
  original slice is preserved so cluster-row traceability survives.
  Moving the artifact breaks referential integrity into
  `removed_docs.parquet` / `clusters.parquet`.
- **Mock signature is explicit when used.** A development run without
  a cosign key is acceptable per Chapter 07, but the dedup report must
  record that the signature is a development mock. Fabricating a
  signature is a governance failure.

### 2.5 `code/emit_report.py` — populate §5

```python
# code/emit_report.py
# Fill in dedup_report.yaml §5 per Chapter 08 schema. Explicit
# pending_review verdict until the peer signs.
from __future__ import annotations

import argparse
import hashlib
from pathlib import Path

import pyarrow.parquet as pq
import yaml


def sha256_file(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def rowcount(p: Path) -> int:
    return pq.read_table(p).num_rows


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--baseline-report", required=True, type=Path)
    ap.add_argument("--cluster-anomalies", required=True, type=Path)
    ap.add_argument("--source-rate-deltas", required=True, type=Path)
    ap.add_argument("--zero-day-clusters", required=True, type=Path)
    ap.add_argument("--quarantine-manifest", required=True, type=Path)
    ap.add_argument("--multimodal-signatures", type=Path, default=None)
    ap.add_argument("--reviewer-email", required=True,
                    help="target ai-risk-engineer; verdict stays pending")
    ap.add_argument("--signature-is-mock", action="store_true")
    ap.add_argument("--output", required=True, type=Path)
    args = ap.parse_args()

    report = yaml.safe_load(args.baseline_report.read_text())
    sig_sidecar = args.quarantine_manifest.with_suffix(
        args.quarantine_manifest.suffix + ".sig"
    )

    report["adversarial_review"] = {
        "cluster_size_histogram_sha256": sha256_file(args.cluster_anomalies),
        "source_removal_rate_delta_sha256": sha256_file(args.source_rate_deltas),
        "zero_day_cluster_count": rowcount(args.zero_day_clusters),
        "perturbation_signature_stats_present": args.multimodal_signatures is not None,
        "perturbation_signature_feature_set_only": True,  # Chapter 07
        "quarantined_doc_count": rowcount(args.quarantine_manifest),
        "quarantine_manifest_sha256": sha256_file(args.quarantine_manifest),
        "quarantine_manifest_signature_path": str(sig_sidecar),
        "quarantine_manifest_signature_is_mock": args.signature_is_mock,
        "atlas_techniques_referenced": [
            "AML.T0019",     # Publish Poisoned Datasets
            "AML.T0020",     # Poison Training Data
            "AML.T0051.001", # LLM Prompt Injection via Training Data
        ],
        "owasp_items_referenced": [
            "ML02:2023",     # Data Poisoning Attack
            # Cite the edition you're auditing against; 2023 or 2025:
            "LLM03:2023",    # Training Data Poisoning (2023 edition)
            # "LLM04:2025",  # Data and Model Poisoning (2025 edition)
        ],
        "ai_risk_engineer_review": {
            "reviewer": args.reviewer_email,
            "sign_off_utc": None,
            "verdict": "pending_review",
        },
    }

    args.output.write_text(yaml.safe_dump(report, sort_keys=False))


if __name__ == "__main__":
    main()
```

Note: the cluster-anomalies / zero-day-clusters files are the *evidence*
the `cluster_size_histogram_sha256` references. If you emit a separate
pre-computed histogram artifact (e.g. a PNG + JSON summary), SHA that
file instead and record the path.

### 2.6 `handoff_to_ai_risk_engineer.md`

```markdown
# Handoff — adversarial-perturbation review for slice <slice-id>

**From:** <operator email>
**To:** <ai-risk-engineer email>
**Slice:** <slice-id> @ version <v>
**Base dedup report SHA-256:** <sha256>
**Quarantine manifest SHA-256:** <sha256>
**Quarantine manifest signature:** <path>; mock=<bool>.

## What is in quarantine

| reason code                 | count | artifact path prefix                                      |
|-----------------------------|-------|-----------------------------------------------------------|
| cluster_size_anomaly        | <n>   | s3://.../quarantine/cluster_size_anomaly/                 |
| source_rate_spike           | <n>   | s3://.../quarantine/source_rate_spike/                    |
| zero_day_cluster            | <n>   | s3://.../quarantine/zero_day_cluster/                     |
| multimodal_signature_outlier| <n>   | s3://.../quarantine/multimodal_signature_outlier/         |

## What I need from you

1. **Review each flagged cell** from `results/cluster_size_anomalies.parquet`.
   Confirm or reject; record the verdict against the cell.
2. **Run detectors you own** (Nightshade, Glaze, general poisoning
   detectors) on the quarantine content. The quarantine bucket is your
   review surface.
3. **Sign the §5 `ai_risk_engineer_review` block** of
   `results/dedup_report.yaml`:
     - `reviewer`: your email.
     - `sign_off_utc`: UTC timestamp at sign-off.
     - `verdict`: `clear_to_release` | `quarantine_confirmed` |
        `partial_release` | `pause_slice`.

## MITRE ATLAS and OWASP context

The §5 report block cites:

- **ATLAS:** `AML.T0019`, `AML.T0020`, `AML.T0051.001`.
- **OWASP ML:** `ML02:2023`.
- **OWASP LLM:** `LLM03:2023` (or `LLM04:2025` if auditing against the
  current edition).

## SLA

Slice release gates on your sign-off. Target review window: <SLA>.
```

## 3. Validation steps

1. **All four heuristics run.** `ls results/*.parquet` contains
   `cluster_size_anomalies.parquet`, `source_removal_rate_deltas.parquet`,
   `zero_day_clusters.parquet`. The multimodal file is present iff the
   slice is multimodal.
2. **Quarantine is not a silent drop.** For each flagged artifact,
   `results/quarantine/<reason>/<name>` exists AND the artifact is also
   present at its original path in the slice.
3. **Manifest is signed.** `results/quarantine_manifest.json.sig`
   exists. `bash code/verify_signature.sh` exits 0. If the signature
   is a development mock, `signature_is_mock: true` is in the report.
4. **§5 fields populated.** `yq '.adversarial_review' results/dedup_report.yaml`
   contains every Chapter 08 schema field.
5. **Verdict is `pending_review`.** `yq '.adversarial_review.ai_risk_engineer_review.verdict'
   results/dedup_report.yaml` equals `pending_review`. `sign_off_utc` is
   `null`. The reviewer email is populated.
6. **ATLAS and OWASP IDs present.** `yq
   '.adversarial_review.atlas_techniques_referenced | length'
   results/dedup_report.yaml` ≥ 3; `yq
   '.adversarial_review.owasp_items_referenced | length'` ≥ 2.
7. **Reason codes conserved.** `quarantined_doc_count` in the report
   matches `len(quarantine_manifest.json["quarantine_items"])` and
   equals the sum of row counts across the quarantine subdirectories.
8. **Feature-set caveat present.** If `multimodal_signatures.parquet`
   exists, `perturbation_signature_feature_set_only: true` is in the
   report.
9. **Handoff is one page and actionable.** `wc -l
   handoff_to_ai_risk_engineer.md` fits on a printed page, and the
   peer can execute the review checklist without a follow-up briefing.
10. **No fabricated sign-off.** `verdict == pending_review` and
    `sign_off_utc` null means the slice is not yet releasable. The
    README should state this explicitly.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible
with evidence). Pass threshold: ≥ 22/32 with zero zeros on critical rows.

| #  | Dimension                                                                                           | Critical? |
|----|------------------------------------------------------------------------------------------------------|-----------|
| 1  | All four heuristics implemented and run; outputs are in parquet                                      | Yes       |
| 2  | Cluster-size anomaly uses per-`(source, crawl_week)` grouping and a ≥ 3σ threshold                   | Yes       |
| 3  | Source rate delta threshold at ≥ 30 pp; includes drops (two-sided)                                   | Yes       |
| 4  | Zero-day clusters defined by set-disjointness with a *historical* source set                         | Yes       |
| 5  | Multimodal column is explicitly a feature set, not a verdict (report flag set)                       | Yes       |
| 6  | Quarantine is a routing / copy operation; original slice intact                                      | Yes       |
| 7  | `quarantine_manifest.json` is signed; mock signatures declared as mock                               | Yes       |
| 8  | `§5.ai_risk_engineer_review.verdict == pending_review` and `sign_off_utc` null                       | Yes       |
| 9  | ATLAS techniques (≥ 3 IDs) and OWASP items (ML02 + an LLM-Top-10 item) referenced                    | Yes       |
| 10 | `quarantined_doc_count` matches manifest row count                                                   | Yes       |
| 11 | `cluster_size_histogram_sha256` and `source_removal_rate_delta_sha256` present                       | Yes       |
| 12 | `handoff_to_ai_risk_engineer.md` is one page, actionable, with SLA                                   | Yes       |
| 13 | README states division of labour (operator vs ai-risk-engineer)                                      | Yes       |
| 14 | Governance anchors (NIST AI RMF, OWASP, MITRE ATLAS) cited in README                                 | No        |
| 15 | Threshold choices (3σ, 30pp) are documented and challengeable                                        | No        |
| 16 | Any unverifiable claim flagged with `<!-- followup-research: ... -->`                                | Yes       |

## 5. Common mistakes

- **Silent-drop quarantine.** Removing flagged content from the slice
  without routing it to the quarantine bucket breaks the audit trail.
  Copy to quarantine *and* leave the original in the slice (dedup
  removal happens elsewhere); the quarantine is the review surface.
- **Fabricating the `ai_risk_engineer_review` sign-off.** You are not
  the reviewer. `verdict: pending_review` and `sign_off_utc: null` is
  the correct state until the peer signs. A fabricated sign-off is a
  governance failure, not a merge annoyance.
- **Claiming the multimodal signature column is a detector.** It is a
  feature set. "DCT high-band ratio > X implies Nightshade" is
  **wrong**: known perturbation tools cluster in these statistics, but
  natural images with high-frequency content do too. `ai-risk-engineer`
  builds the detector; you produce the inputs.
- **Thresholds that catch 10% of a clean slice.** A pipeline-side
  heuristic that quarantines 10% of clean data is unusable. The
  stretch-goal measurement is to run the heuristics on a synthetic
  clean slice and record the false-positive rate; if it is high,
  revisit the thresholds *before* sending the handoff.
- **Zero-day defined as "contains any new source".** Too loose; every
  cluster that touches a newly-discovered source would be flagged.
  The operational definition is *every* member source is new.
- **Running `cosign` with a key you committed to the repo.** Private
  keys do not belong in the repo. A development mock signature is
  acceptable per Chapter 07 as long as it is *declared* as a mock in
  the report. For production, use Sigstore keyless or a managed KMS
  key.
- **Aggregate cluster-size histograms.** The signal lives in the
  `(source, crawl_week)` disaggregation. A whole-slice histogram will
  miss the one source publishing 50k near-dups in a week.
- **Failing to cite the OWASP edition.** LLM03:2023 is
  "Training Data Poisoning"; LLM04:2025 is "Data and Model Poisoning".
  The item moved. Cite the edition.
- **Omitting the `signature_is_mock` flag.** A development run can
  produce a mock signature; the report must say so. `cosign verify`
  on a mock sidecar will succeed trivially — that is misleading
  without the flag.
- **Treating `pending_review` as "incomplete and must be fixed".** It
  is the correct state at your handoff. The report is complete *at
  this layer*; the gate is `ai-risk-engineer`'s sign-off.

## 6. References

- **Shan, S., Ding, W., Passananti, J., Wu, H., Zheng, H., Zhao, B.,
  *"Nightshade: Prompt-Specific Poisoning Attacks on Text-to-Image
  Generative Models"*, IEEE S&P 2024** —
  <https://ieeexplore.ieee.org/document/10646862> /
  arXiv 2310.13828 <https://arxiv.org/abs/2310.13828>. Project page:
  <https://nightshade.cs.uchicago.edu/>.
- **Shan, S., Cryan, J., Wenger, E., Zheng, H., Hanocka, R., Zhao, B.,
  *"Glaze: Protecting Artists from Style Mimicry by Text-to-Image
  Models"*, USENIX Security 2023** —
  <https://www.usenix.org/conference/usenixsecurity23/presentation/shan>
  / arXiv 2302.04222. Project page: <https://glaze.cs.uchicago.edu/>.
- **Carlini, N., Jagielski, M., Choquette-Choo, C.A. et al.,
  *"Poisoning Web-Scale Training Datasets is Practical"*, 2023** —
  <https://arxiv.org/abs/2302.10149>. Proof that web-scale pretraining
  corpora can be poisoned for a few hundred dollars; the Chapter 07
  anchor for why pipeline-side signals are mandatory.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. AML.T0019, AML.T0020,
  AML.T0051.001, AML.T0018 are the techniques §5 cites. Re-check
  technique titles at read time; ATLAS continues to evolve (the
  2026.07 release renamed AML.T0020 from "Poison Training Data" to
  "Training Data Poisoning" without changing the identifier).
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02:2023 Data Poisoning Attack is the canonical governance anchor.
- **OWASP Large Language Model Top 10** —
  <https://genai.owasp.org/llm-top-10/>. LLM03:2023 (Training Data
  Poisoning) in the 2023 edition; renumbered and broadened as
  LLM04:2025 Data and Model Poisoning in the 2025 edition. Cite the
  edition you audit against.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE 2.7
  (security and resilience are measured) and MANAGE 1.3 (risks tracked
  and documented) are the control anchors.
- **Sigstore** — <https://www.sigstore.dev/>. Canonical signing path
  for the quarantine manifest. `cosign sign-blob` is the CLI entry
  point.
- **Local anchors:** Chapter 07 (this exercise's primary content);
  Chapter 08 §5 (report schema); Exercise 04 (the pipeline output this
  exercise consumes); `mod-101` (division-of-labour source for the
  `ai-data-engineer` vs `ai-risk-engineer` split); `mod-109` (the
  Sigstore signing workflow); `mod-110` (consumes
  `adversarial_review.quarantined_doc_count` for the privacy-risk
  report); `mod-112` (consumes the quarantine manifest for the dataset
  card's "known filtered-out content" section).
