# Exercise 06 — SDV or Gretel Tabular Synthetic Data with DP (Solution)

> **Exercise-prompt note.** The learning file for this exercise
> (`exercise-06-sdv-or-gretel-tabular-synthetic-data-with-dp.md`) is
> a scaffolded stub. This solution is grounded in Chapter 09
> (`09-privacy-preserving-tabular-synthesis-sdv-gretel-mostly.md`),
> which is fully written, and in the exercise contract from
> `README.md`. Any future divergence from the exercise prompt should
> re-ground against that chapter.
> <!-- pending-verify: confirm the exercise prompt once it lands; the
> scaffolded stub does not yet pin dataset, tool choice, or (ε, δ)
> target. -->

## 1. Solution overview

This is a **mixed design + implementation** exercise. The learner
picks one of the three Chapter 09 tools (SDV, Gretel AI, or
MOSTLY AI), trains a differentially-private generator on a sensitive
tabular source, samples a synthetic table, measures utility against
the real table across four levels (marginal, pairwise, TSTR,
privacy audit), and ships a counsel-signoff ledger ready for
legal review before the synthetic table is released to any
downstream recipient.

Deliverables:

- `generator/dp_training_log.json` — the training-time privacy
  accountant output: realised `(ε, δ)`, composition analysis,
  training iterations, noise scale, clipping norm.
- `tabular/counsel_signoff_ref.md` — the Chapter 09 counsel ledger
  capturing regulatory class, generator identity, DP parameters,
  utility report, and the counsel sign-off slot.
- `tabular/utility_report.md` — the four-level utility report:
  marginal distribution fidelity, pairwise correlation fidelity,
  TSTR downstream-model accuracy, empirical privacy-attack audit.
- `outputs/synthetic_table.parquet` — the final DP-synthesised
  table, with provenance back to the real table and the generator.
- `recipes/synthesis_config.yaml` — tool, model class, (ε, δ)
  target, utility targets, downstream intent.

Governance framing. Tabular DP synthesis is where the module's
licence / privacy exposure is sharpest: the inputs are regulated
data (customer records, clinical records), the output is shared
with partners or vendors, and the formal guarantee `(ε, δ)` is the
control counsel reads.

- OWASP Machine Learning Security Top 10 anchors: **ML03:2023
  Model Inversion Attack** (membership inference and attribute
  inference against the DP generator); **ML06:2023 AI Supply Chain
  Attacks** (the hosted DP-synthesis vendor is an upstream
  dependency).
- MITRE ATLAS techniques: **AML.T0024.001 Membership Inference**
  (the Level-4 privacy audit); **AML.T0010 ML Supply Chain
  Compromise** (vendor integrity for the hosted DP tools).
- NIST AI Risk Management Framework controls: **GOVERN 6.1** (third-
  party risk management — SDV / Gretel / MOSTLY AI vendor posture);
  **MAP 4.1** (legal and regulatory requirements mapping — GDPR /
  HIPAA / CCPA / GLBA posture on `(ε, δ)`); **MEASURE 2.6** (safety
  — empirical privacy-attack audit).

Cite the framework URLs rather than paraphrasing. The ledger
captures facts; counsel renders the legal opinion.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-06/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── synthesis_config.yaml
├── generator/
│   ├── dp_training_log.json          # realised (ε, δ) + composition
│   └── snapshots/
│       ├── tool_release_notes.md     # which tool + version used
│       └── license_terms.md          # SDV BSL-1.1 or Apache 2.0
├── tabular/
│   ├── utility_report.md
│   └── counsel_signoff_ref.md
├── code/
│   ├── train_dp_generator.py         # SDV / Gretel / MOSTLY wrapper
│   ├── sample_synthetic.py
│   ├── marginal_fidelity.py          # Level 1
│   ├── pairwise_fidelity.py          # Level 2
│   ├── tstr.py                        # Level 3
│   └── privacy_audit.py               # Level 4 (membership inference)
├── inputs/
│   ├── real_table.parquet            # sensitive source
│   ├── real_table.parquet.sha256
│   ├── real_split_train.parquet
│   ├── real_split_test.parquet       # held-out for TSTR
│   └── schema.yaml                    # column types + constraints
├── outputs/
│   ├── synthetic_table.parquet
│   └── synthetic_table.parquet.sha256
└── Makefile
```

### 2.2 Tool-choice decision record

Chapter 09 §"Picking between the three" lays out the decision. Pin
the choice in a short `generator/snapshots/tool_release_notes.md`:

```markdown
# Tool choice: Gretel AI (`gretel-synthetics` + hosted platform)

**Decision date:** 2026-01-15
**Decision maker:** <operator email>

## Decision criteria (Chapter 09 §"Picking between the three")

| Criterion                      | Gretel (chosen)                             |
|--------------------------------|---------------------------------------------|
| Open-source baseline           | yes (`gretel-synthetics`, Apache 2.0)       |
| DP first-class                 | yes (training-time knob, accountant output) |
| Relational / multi-table       | yes                                         |
| Time-series                    | yes (DGAN)                                  |
| Regulatory use cases           | enterprise                                  |
| Jurisdiction                   | EU + US                                     |
| Vendor procurement posture     | approved (existing MSA)                     |

## Why not SDV

SDV's community synthesizers (`CTGANSynthesizer`, `TVAESynthesizer`)
are not DP by default. Chapter 09 §"SDV" is explicit: DP variants
such as `DPGCSynthesizer` are documented as SDV Enterprise / bundle
features. For this run the open-source community edition does not
clear the DP-first-class criterion.

## Why not MOSTLY AI

Vendor procurement path is longer. Comparable DP posture and utility
profile; defer to Gretel for this exercise.

## Licence posture

- `gretel-synthetics` is Apache 2.0.
- The Gretel hosted platform is commercial; MSA #12345 governs use.
- The real table remains in-jurisdiction (EU); the Gretel
  self-hosted deployment is used to keep processing in-region.
```

### 2.3 `code/train_dp_generator.py` — the DP-training wrapper

```python
# code/train_dp_generator.py
# Gretel-synthetics DP training. The specific class names and knobs
# below are indicative of the Gretel open-source shape; verify
# against the installed library version. The releasable shape of
# this wrapper is: train, extract (ε, δ) from the accountant, emit
# dp_training_log.json, sample, verify sample provenance.
from __future__ import annotations

import argparse
import json
from pathlib import Path

# <!-- pending-verify: pin exact Gretel-synthetics class path and
# version string. The DP tabular path has moved across releases
# (ACTGAN vs DGAN vs tabular-DP variants); verify against the
# installed library. -->
from gretel_synthetics.tabular.dp_trainer import DpTabularTrainer


def train_and_log(real_table: Path, output_dir: Path,
                   epsilon: float, delta: float,
                   epochs: int, batch_size: int, seed: int) -> dict:
    output_dir.mkdir(parents=True, exist_ok=True)
    trainer = DpTabularTrainer(
        epsilon=epsilon, delta=delta,
        epochs=epochs, batch_size=batch_size, seed=seed,
    )
    trainer.fit(real_table)
    trainer.save(output_dir)

    accountant = trainer.privacy_accountant()
    log = {
        "mechanism": "DP-SGD (Gretel tabular)",
        "library": "gretel-synthetics",
        "library_version": trainer.version,
        "epsilon_requested": epsilon,
        "delta_requested": delta,
        "epsilon_realised": accountant.epsilon(),
        "delta_realised": accountant.delta(),
        "composition": "Renyi-DP (RDP)",
        "noise_multiplier": trainer.noise_multiplier,
        "l2_clip_norm": trainer.l2_clip_norm,
        "microbatches": trainer.microbatches,
        "steps": trainer.steps,
        "batch_size": batch_size,
        "epochs": epochs,
        "seed": seed,
        "generator_dir": str(output_dir),
    }
    (output_dir / "dp_training_log.json").write_text(
        json.dumps(log, indent=2, sort_keys=True))
    return log
```

Design notes defensible at review:

- **Realised `(ε, δ)` is the audit anchor, not the requested
  value.** Chapter 09 §"Where tabular synthesis breaks" §1 (implicit
  non-DP baseline) is explicit — the accountant output is what
  passes to counsel. Pin both the requested and the realised value;
  if they diverge by more than a small tolerance, the training step
  is misconfigured.
- **Composition is reported.** Renyi-DP (RDP) accounting is the
  standard for DP-SGD (Mironov 2017 / Abadi et al. 2016). Pin the
  accounting method; pure-DP, zCDP, or Gaussian-DP produce
  different `(ε, δ)` numbers for the same noise scale.
- **Seed is pinned in the training log.** Reproducibility across
  runs requires the seed; without it, the accountant output cannot
  be rechecked against a replay.

### 2.4 `code/marginal_fidelity.py` — Level 1

```python
# code/marginal_fidelity.py
# Per-column marginal-distribution comparison between real and
# synthetic. Chapter 09 §"Level 1 — marginal distribution fidelity".
from __future__ import annotations

from dataclasses import dataclass
from typing import Any

import numpy as np
import pandas as pd
from scipy.stats import ks_2samp


@dataclass(slots=True)
class MarginalResult:
    column: str
    kind: str              # "categorical" | "numeric"
    metric: str            # "total_variation" | "ks_statistic"
    value: float


def tv_distance(real: pd.Series, syn: pd.Series) -> float:
    r = real.value_counts(normalize=True)
    s = syn.value_counts(normalize=True)
    cats = set(r.index) | set(s.index)
    return 0.5 * sum(abs(r.get(c, 0.0) - s.get(c, 0.0)) for c in cats)


def ks(real: pd.Series, syn: pd.Series) -> float:
    return ks_2samp(real.dropna(), syn.dropna()).statistic


def evaluate(real: pd.DataFrame, syn: pd.DataFrame,
              schema: dict[str, Any]) -> list[MarginalResult]:
    rows: list[MarginalResult] = []
    for col, meta in schema.items():
        if meta["kind"] == "categorical":
            rows.append(MarginalResult(col, "categorical", "total_variation",
                                         tv_distance(real[col], syn[col])))
        else:
            rows.append(MarginalResult(col, "numeric", "ks_statistic",
                                         ks(real[col], syn[col])))
    return rows
```

### 2.5 `code/tstr.py` — Level 3, Train-on-Synthetic Test-on-Real

```python
# code/tstr.py
# Level 3: train a reference downstream model on the synthetic
# table, test on a held-out real split, and compare against the
# same model trained on the real training split. The ratio is the
# TSTR utility score.
from __future__ import annotations

from dataclasses import dataclass

import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import accuracy_score, roc_auc_score


@dataclass(slots=True)
class TSTRResult:
    task: str
    accuracy_real: float
    accuracy_syn: float
    accuracy_ratio: float
    auc_real: float
    auc_syn: float


def tstr(task: str, features: list[str], target: str,
          real_train: pd.DataFrame, real_test: pd.DataFrame,
          syn: pd.DataFrame, seed: int = 42) -> TSTRResult:
    def fit_eval(train: pd.DataFrame) -> tuple[float, float]:
        X_tr, y_tr = train[features], train[target]
        X_te, y_te = real_test[features], real_test[target]
        clf = GradientBoostingClassifier(random_state=seed).fit(X_tr, y_tr)
        preds = clf.predict(X_te)
        probs = clf.predict_proba(X_te)[:, 1]
        return accuracy_score(y_te, preds), roc_auc_score(y_te, probs)

    a_real, auc_real = fit_eval(real_train)
    a_syn, auc_syn = fit_eval(syn)
    return TSTRResult(task, a_real, a_syn,
                        a_syn / a_real if a_real > 0 else 0.0,
                        auc_real, auc_syn)
```

### 2.6 `code/privacy_audit.py` — Level 4, membership inference

```python
# code/privacy_audit.py
# Level 4: empirical membership-inference audit against the DP
# generator. Even with DP, run the attack — Chapter 09 §"Level 4"
# calls it "a sanity check that the DP implementation actually
# behaved as advertised". Shokri et al. 2017 (arXiv 1610.05820) is
# the canonical reference.
from __future__ import annotations

from dataclasses import dataclass

import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score


@dataclass(slots=True)
class MembershipInferenceResult:
    auc: float                  # near 0.5 is good (no advantage)
    n_shadow_models: int
    attack_type: str


def membership_inference_audit(generator, real_members: list[dict],
                                 real_non_members: list[dict],
                                 n_shadow: int = 10,
                                 seed: int = 42) -> MembershipInferenceResult:
    """Shadow-model membership inference.

    Train shadow models with and without each row; build a loss-
    distribution attack model; evaluate AUC on held-out
    members-vs-non-members. A detailed sandbox implementation is out
    of scope here; the shape of the result is what's pinned.
    """
    rng = np.random.default_rng(seed)
    scores: list[tuple[int, float]] = []    # (label, attack_score)

    # Pseudocode: in-shadow-model loss for each row vs in-no-shadow loss.
    for row in real_members + real_non_members:
        label = 1 if row in real_members else 0
        score = _attack_score(generator, row, n_shadow=n_shadow, rng=rng)
        scores.append((label, score))

    labels = [s[0] for s in scores]
    vals = [s[1] for s in scores]
    auc = roc_auc_score(labels, vals)
    return MembershipInferenceResult(auc=auc, n_shadow_models=n_shadow,
                                       attack_type="shadow_model_loss")


def _attack_score(generator, row, *, n_shadow: int, rng) -> float:
    ...   # out of scope — depends on the generator's log-likelihood API
```

Two design choices worth defending at review:

- **The audit uses a shadow-model attack.** Shokri et al. 2017 is
  the published anchor; it gives an AUC-grounded number counsel
  can read. A simpler loss-based attack is cheaper but less
  sensitive; call out the choice in the utility report.
- **AUC near 0.5 is the pass signal.** AUC meaningfully above 0.5
  (0.55 or higher at `N ≥ a few thousand`) is a fail — the DP
  implementation did not deliver what the accountant reported. The
  threshold is report-level, not a code-level gate.

### 2.7 `tabular/utility_report.md`

```markdown
# Utility report — run `distillation-run-7` / `alpha=2.0`

## Privacy
- Mechanism: DP-SGD (Gretel tabular)
- Library: gretel-synthetics `0.21.0` <!-- pending-verify: pin to the version actually installed -->
- Epsilon (requested): 2.0
- Epsilon (realised): 2.08
- Delta: 1.0e-6
- Composition: Renyi-DP
- Training iterations: 10,000
- Noise multiplier: 1.1
- L2 clipping norm: 1.0

## Utility — Level 1 (marginal)
| Column            | Kind        | Metric            | Value |
|-------------------|-------------|-------------------|-------|
| age               | numeric     | KS statistic      | 0.03  |
| income            | numeric     | KS statistic      | 0.09  |
| region            | categorical | Total variation   | 0.04  |
| product_category  | categorical | Total variation   | 0.11  | <!-- worst column -->

## Utility — Level 2 (pairwise)
- Mean correlation error (Pearson, numeric pairs): 0.07
- Worst-pair error: (income, credit_balance) at 0.22

## Utility — Level 3 (TSTR, downstream classifier)
- Task: default_risk_classifier
- Features: [age, income, region, ...]
- Target: defaulted_12m
- Accuracy on real test, trained on real: 0.843
- Accuracy on real test, trained on synthetic: 0.802
- Accuracy ratio: 0.951
- AUC (real-trained): 0.892
- AUC (synthetic-trained): 0.851

## Utility — Level 4 (privacy audit)
- Attack: shadow-model membership inference (Shokri et al. 2017)
- Shadow models: 10
- AUC: 0.52      # near 0.5 → no meaningful advantage
- Attribute-inference accuracy delta vs baseline: +0.01

## Interpretation
The realised ε (2.08) is within 5% of the request; the DP
accountant output is consistent. Marginal and pairwise fidelity are
acceptable with the known worst column (`product_category`) and
worst pair (`income, credit_balance`) acknowledged. TSTR ratio at
0.951 is above the 0.90 threshold the downstream user specified.
Membership-inference AUC at 0.52 is within tolerance of the
no-advantage baseline.

Known caveats:
- The TSTR task is a binary classifier; a regression downstream
  task has not been measured and would be a separate ledger entry.
- The audit used a shadow-model attack; a stronger attack (e.g.,
  LiRA, Carlini et al. 2022) is deferred to a future run.
```

### 2.8 `tabular/counsel_signoff_ref.md` — the counsel ledger

```yaml
# tabular/counsel_signoff_ref.md (front-matter + prose)
real_table:
  identity: finance_customer_records_2025_q3
  sha256: <64-hex-digest>
  regulatory_class: [GDPR, GLBA]
  source_dataset_licence_ref: ...
  data_subject_count: 420318

generator:
  tool: gretel_platform
  tool_version: 2026.01    # <!-- pending-verify: pin actual release -->
  model_class: DP-tabular   # see tool_release_notes.md
  training_epochs: 100
  epsilon_requested: 2.0
  epsilon_realised: 2.08
  delta: 1.0e-6
  training_log_sha256: <SHA of generator/dp_training_log.json>

intended_use:
  recipient: partner_bank_quant_team
  jurisdiction: EU
  purpose: downstream_modelling_of_default_risk
  retention_period: 180_days
  distribution_terms: restricted-use-and-redaction-contract

utility_evidence:
  utility_report_path: tabular/utility_report.md
  utility_report_sha256: <64-hex-digest>

questions_for_counsel:
  - Does the real-table processing authorisation permit training a DP
    generator on this data for the purpose stated above?
  - Is DP training at realised ε = 2.08, δ = 1e-6 an acceptable
    risk profile for this use case under GDPR Art. 25 and the
    supervisory authority's guidance on synthetic data?
  - Can the synthetic output be shared with the named recipient,
    for the named purpose, under the terms specified?
  - Does the derived synthetic table need additional controls
    (watermarking, row-level provenance, re-identification
    reporting obligations) beyond what we've captured?

counsel_signoff:
  reviewer: TBD
  reviewer_email: TBD
  reviewed_at: TBD
  opinion_summary: TBD
  opinion_sha256: TBD
  gate_passed: TBD
```

Pre-populating `counsel_signoff` is a rubric failure. The author
captures facts; counsel signs. The ledger is a release gate: a
`gate_passed` other than `true` or `conditional` (with the
conditions captured) blocks hand-off to the recipient.

### 2.9 `recipes/synthesis_config.yaml`

```yaml
# recipes/synthesis_config.yaml
real_table:
  path: inputs/real_table.parquet
  sha256: <64-hex-digest>
  schema_path: inputs/schema.yaml
  regulatory_class: [GDPR, GLBA]

generator:
  tool: gretel_synthetics
  tool_version: 0.21.0     # <!-- pending-verify: pin actual release -->
  model_class: DP-tabular
  epsilon: 2.0
  delta: 1.0e-6
  composition: RDP
  training:
    epochs: 100
    batch_size: 512
    noise_multiplier: 1.1
    l2_clip_norm: 1.0
    microbatches: 1
    seed: 42

utility_targets:
  marginal:
    mean_tv_distance_max: 0.10
    worst_column_tv_max: 0.15
  pairwise:
    mean_correlation_error_max: 0.10
  tstr:
    task: default_risk_classifier
    accuracy_ratio_min: 0.90
  privacy_audit:
    membership_inference_auc_max: 0.55

hand_off:
  recipient: partner_bank_quant_team
  jurisdiction: EU
  purpose: downstream_modelling_of_default_risk
  counsel_signoff_ref: tabular/counsel_signoff_ref.md
```

## 3. Validation steps

1. **Real-table hash matches.** Sidecar and manifest agree.
2. **DP training log is present and consistent.**
   `generator/dp_training_log.json` reports realised
   `(ε, δ)` within tolerance of the requested values and the
   composition method is pinned.
3. **Non-DP baseline is not shipped.** The training log explicitly
   names a DP mechanism; a run where the training log would be
   produced by a non-DP model class (`CTGANSynthesizer`,
   `TVAESynthesizer`, `GaussianCopulaSynthesizer`) fails. Chapter
   09 §"Where tabular synthesis breaks" §1 is the anchor.
4. **Utility report covers all four levels.** Marginal, pairwise,
   TSTR, and privacy audit sections all exist and all pass their
   manifest thresholds (or document the override).
5. **Membership-inference AUC is near 0.5.** A value meaningfully
   above the manifest threshold blocks the hand-off.
6. **Counsel ledger is pre-populated with `TBD` on the sign-off
   block.** Pre-signing fails the rubric.
7. **Opinion attached when `gate_passed == true`.**
   `opinion_sha256` resolves to a file; the SHA matches.
8. **Schema constraints respected in the synthetic table.** Join
   keys, foreign keys, column types, and declared ranges from
   `inputs/schema.yaml` are enforced on the output.
9. **Jurisdiction captured.** The ledger's `intended_use.jurisdiction`
   and the training environment's jurisdiction match (self-hosted
   or in-region hosted). A cross-jurisdiction transfer is a flag.
10. **Row-level provenance.** Every row in
    `outputs/synthetic_table.parquet` carries a `generator_run_id`
    and `training_log_sha256` so a later audit can trace the row
    back to the DP training run.
11. **Deterministic reproducibility at fixed seed.** Training a
    second time with the same seed produces the same accountant
    output (modulo library-level non-determinism — pin and
    document).

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Tool choice is documented with the Chapter 09 decision criteria matrix                                              | No        |
| 2  | DP training log reports realised `(ε, δ)` + composition + noise + clipping + seed                                    | Yes       |
| 3  | `δ ≤ 1 / data_subject_count` — standard practice                                                                     | Yes       |
| 4  | Non-DP model class is not used (community CTGAN / TVAE / GaussianCopula fails)                                       | Yes       |
| 5  | Marginal (Level 1) fidelity covers every column in the schema                                                        | Yes       |
| 6  | Pairwise (Level 2) correlation error reported with worst-pair identified                                              | Yes       |
| 7  | TSTR (Level 3) accuracy ratio measured against a held-out real split                                                  | Yes       |
| 8  | Privacy audit (Level 4) membership-inference AUC reported with shadow count                                           | Yes       |
| 9  | Counsel ledger (`tabular/counsel_signoff_ref.md`) is pre-populated with `TBD` on the sign-off block                   | Yes       |
| 10 | Questions for counsel cover at least: permission-to-train, permission-to-share, derived-table licence                 | Yes       |
| 11 | Synthetic rows carry `generator_run_id` + `training_log_sha256` for lineage                                           | Yes       |
| 12 | Jurisdiction and recipient terms are pinned in the ledger                                                            | Yes       |
| 13 | Governance anchors (OWASP ML Top 10 ML03 / ML06, MITRE ATLAS AML.T0024.001 / AML.T0010, NIST AI RMF GOVERN 6.1 / MAP 4.1 / MEASURE 2.6) cited | No |
| 14 | Unverifiable claims marked `<!-- pending-verify: ... -->`                                                           | Yes       |
| 15 | Utility report's "known caveats" section names the attacks and tasks that were *not* measured                        | No        |

## 5. Common mistakes

- **Implicit non-DP baseline.** Shipping `CTGANSynthesizer` or
  `TVAESynthesizer` output and labelling it "synthetic data"
  without a DP accountant output fails Chapter 09 §"Where tabular
  synthesis breaks" §1. The training log must name a DP mechanism.
- **Realised `(ε, δ)` ignored.** The requested value is a
  configuration knob; the realised value is what passes to
  counsel. A mismatch between the two is a training-step
  misconfiguration.
- **`δ` too large.** Chapter 09 pins `δ ≤ 1 / |T|` as standard;
  with 420k subjects, `δ = 1e-6` is appropriate. A `δ` of 1e-3
  permits leaking many records with reasonable probability — the
  "formal" guarantee becomes meaningless.
- **`ε` loosened without counsel.** Chapter 09 §"The utility-vs-
  privacy curve" §4 is explicit: loosening `ε` is a risk-
  acceptance change requiring counsel, not a configuration tweak.
- **Pre-signing counsel.** The author captures facts; counsel
  signs. A ledger shipped with `gate_passed: true` and no
  `opinion_sha256` is a rubric failure.
- **Skipping the Level-4 privacy audit.** Even with DP, empirical
  membership inference is the sanity check. A DP implementation
  that advertises `ε = 2` but fails the shadow-model attack is a
  flag worth routing back to counsel.
- **Mixing jurisdictions silently.** Real table in EU, training on
  a US-hosted platform without the self-hosted deployment, no
  Data Processing Agreement: this is a GDPR exposure. The ledger
  must capture jurisdiction explicitly.
- **Missing row-level provenance on the synthetic table.** A row
  that cannot be traced back to its DP training run is unauditable.
- **Relational / multi-table breakage.** Per-table independent
  generators produce orphaned foreign keys. Use the hierarchical
  model variants (SDV's HMA, Gretel's / MOSTLY AI's native
  relational mode) where multi-table integrity matters.
- **Shipping a membership-inference audit AUC meaningfully above
  0.5 without investigation.** Whether this is a DP bug, a
  memorisation-of-rare-rows symptom, or a measurement artifact
  needs to be identified. Chapter 09 §"Where tabular synthesis
  breaks" §3 is the anchor.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML03:2023 Model
  Inversion Attack; ML06:2023 AI Supply Chain Attacks:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML03 is the governance anchor for membership-inference against
  the DP generator; ML06 is the vendor-supply-chain anchor.
- **MITRE ATLAS — AML.T0024.001 Membership Inference;
  AML.T0010 ML Supply Chain Compromise:**
  <https://atlas.mitre.org/>. Adversary-facing anchors.
- **NIST AI Risk Management Framework — GOVERN 6.1 (third-party
  risk management), MAP 4.1 (legal and regulatory requirements
  mapping), MEASURE 2.6 (safety):**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- **Dwork, Roth. *The Algorithmic Foundations of Differential
  Privacy.* Foundations and Trends in TCS 2014:**
  <https://www.cis.upenn.edu/~aaroth/privacybook.html>. The
  canonical textbook reference for `(ε, δ)`-DP and composition.
- **Abadi et al. 2016, *Deep Learning with Differential Privacy*,
  arXiv 1607.00133:** <https://arxiv.org/abs/1607.00133>. DP-SGD,
  the training-time mechanism all three tools use.
- **Shokri et al. 2017, *Membership Inference Attacks Against
  Machine Learning Models*, arXiv 1610.05820:**
  <https://arxiv.org/abs/1610.05820>. The canonical membership-
  inference attack reference used in the Level-4 audit.
- **SDV — Synthetic Data Vault:** <https://github.com/sdv-dev/SDV>,
  docs <https://docs.sdv.dev/>. Chapter 09 §"SDV" details the
  Business Source Licence posture — verify the specific release's
  LICENSE file. Original paper: Patki, Wedge, Veeramachaneni
  2016: <https://dai.lids.mit.edu/>.
- **Gretel AI:** <https://gretel.ai/>, docs
  <https://docs.gretel.ai/>, OSS
  <https://github.com/gretelai/gretel-synthetics>.
- **MOSTLY AI:** <https://mostly.ai/>, docs
  <https://mostly.ai/docs/>, OSS
  <https://github.com/mostly-ai/mostlyai>.
- **Local anchors:** `mod-108` Chapter 09 (the full tool survey
  and the counsel ledger shape); `mod-108` Chapter 05 (the
  licence-ledger pattern the counsel ledger specialises); `mod-110`
  (PII and sensitive-content scanning complementary to the DP
  guarantee); `mod-109` (lineage from training log through
  synthetic rows).
