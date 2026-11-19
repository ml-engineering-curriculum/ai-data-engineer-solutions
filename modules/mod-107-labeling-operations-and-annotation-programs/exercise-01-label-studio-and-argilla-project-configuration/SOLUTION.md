# Exercise 01 — Label Studio and Argilla Project Configuration (Solution)

## 1. Solution overview

This is a **mixed implementation + design** exercise. The learner stands
up two annotation projects for the same labeling program — one in
**Label Studio** (classical supervised labeling, e.g., span-level NER
or multi-class classification) and one in **Argilla** (LLM-centric
feedback: rating, ranking, response editing). Both projects must be
driven from version-controlled configuration so that the labeling
*contract* (schema, guidelines, consensus rule, quality bar) is the
artifact the module signs — not the clicks an admin made in a UI.

Deliverables:

- A **labeling policy note** committed before any project is created
  — scope of the program, the taxonomy, consensus rule, and the bar at
  which a label is accepted into the slice (`doc_matches` of mod-106
  is not sufficient; labels must be *explained*).
- A **Label Studio** `XMLLabelConfig` + `project.json` + `tasks.jsonl`
  seed, importable with `label-studio init` / `label-studio-converter`.
- An **Argilla** `FeedbackDataset` created with the Python SDK;
  `settings.yaml` declares the fields, questions, and annotation
  guidelines so `argilla.FeedbackDataset.from_yaml(...)` reproduces
  the project from source.
- A **gold set** of ≥25 pre-labeled items that both projects import
  as a calibration slice; annotator accuracy on gold feeds the
  quality gate in mod-107 ex-03.
- A one-page **operator runbook** describing how a labeler gets
  seated, how a reviewer adjudicates, and how the program closes a
  batch (freeze → export → sign manifest).

Governance framing. Labeling is where class and task-definition
drift enter the training set, so the control anchor is **NIST AI RMF
MEASURE 2.3** (performance measured *as intended* — a class the
annotators disagree on cannot be measured as intended) combined with
**MAP 2.3** (scientific integrity of training data). From an
adversary-facing view, **OWASP ML Top 10 ML02:2023 Data Poisoning
Attack** covers the "insider annotator poisons labels" and the
"vendor poisons a batch" scenarios, and **MITRE ATLAS AML.T0020
Poison Training Data** / **AML.T0018 Backdoor ML Model** are the
techniques the project configuration must deny. Cite the framework
URLs under §6; do not paraphrase the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── policy_note.md                  # committed before any config
├── taxonomy.yaml                   # single source of truth for classes
├── label_studio/
│   ├── label_config.xml            # XMLLabelConfig
│   ├── project.json                # title, description, color, mapping
│   ├── tasks_seed.jsonl            # 25 gold items + 25 production items
│   ├── gold_items.jsonl            # pre-adjudicated answers (held out of UI)
│   └── make_project.sh             # idempotent CLI bootstrap
├── argilla/
│   ├── settings.yaml               # FeedbackDataset schema in YAML form
│   ├── make_dataset.py             # creates dataset, pushes records
│   ├── records.jsonl               # production records
│   └── gold_records.jsonl          # calibration records for `suggestions`
├── runbook.md                      # operator-facing one-pager
└── exports/
    ├── labelstudio_export.json     # batch export at freeze
    ├── argilla_export.jsonl
    └── batch_manifest.json         # signed by cosign on freeze
```

### 2.2 `policy_note.md` — commit this before any project is created

```markdown
# Labeling policy — program <program-id>

**Decision date:** 2026-11-18
**Author:** <operator email>
**Scope:** applies to batches 001–NNN of program <program-id>. Any
change to `taxonomy.yaml`, the consensus rule, or the review bar
invalidates batches opened after the change and requires a new
`policy_note.md` version.

## Task and taxonomy

- **Task:** span-level NER + multi-class classification over the
  `claims` corpus (Label Studio); single-turn **response rating +
  response ranking** over the `chat_reviews` corpus (Argilla).
- **Taxonomy file:** `taxonomy.yaml` is the single source of truth.
  Both projects import from it. No label exists only in one UI.
- **Open-world rule:** `none_of_the_above` is a required class;
  annotators must not force a prediction when the taxonomy does not
  fit.

## Consensus rule

- Each production item receives **3 independent annotations** (no
  annotator sees the others' work).
- A label is **adopted** iff ≥ 2 of 3 annotations agree on both the
  class and (for spans) the character offsets within ±3 characters.
- An item lacking adoption is routed to **adjudication** by a
  reviewer; the reviewer's label is final and is logged with the
  disagreement trace.

## Quality bar

- **Gold accuracy ≥ 0.90** per annotator on the calibration slice is
  the seating gate. Below that, the annotator stays in training.
- **Cohen's κ ≥ 0.70** pairwise, **Krippendorff's α ≥ 0.67** across
  all annotators (mod-107 ex-03 computes these) at batch close. A
  batch failing either number does not ship; the batch reopens with
  updated guidelines.
- Labels adopted purely by consensus but carrying any single
  annotator `gold_accuracy < 0.80` are flagged for reviewer
  spot-check at 10% of batch volume.

## Adversarial controls (ML02 / AML.T0020)

- Annotator pool is partitioned; no single annotator sees more than
  15% of a batch, bounding insider-poisoning blast radius.
- A **canary set** of known-answer items is injected at ~3% of
  volume; a drop in canary accuracy triggers vendor audit under
  mod-107 ex-02.
```

Commit `policy_note.md` and `taxonomy.yaml` in a dedicated commit
before any `label_studio/` or `argilla/` file. The `git log --follow`
trail is what mod-109 lineage signs against.

### 2.3 `taxonomy.yaml` — the shared source of truth

```yaml
# taxonomy.yaml
version: 1
updated_utc: 2026-11-18T00:00:00Z

ner_classes:
  - id: PERSON
    description: Named natural person; excludes job titles and roles.
    examples_positive: ["Dr. Ada Lovelace"]
    examples_negative: ["the surgeon", "a software engineer"]
  - id: ORG
    description: Named organisation; excludes generic groups.
    examples_positive: ["Anthropic", "the Red Cross"]
    examples_negative: ["the company", "a team"]
  - id: CLAIM_SUBJECT
    description: The entity a factual claim is *about*.
  - id: NONE_OF_THE_ABOVE
    description: Required open-world escape hatch.

classification_classes:
  - id: factual
  - id: opinion
  - id: question
  - id: none_of_the_above

rating_scale:
  type: likert_5
  anchors:
    1: "Unhelpful or wrong"
    2: "Partial"
    3: "Helpful with caveats"
    4: "Helpful"
    5: "Exemplary"

ranking:
  mode: pairwise
  ties_allowed: true

annotator_instructions_ref: policy_note.md#task-and-taxonomy
```

### 2.4 Label Studio — `label_studio/label_config.xml`

```xml
<!-- label_config.xml — Label Studio XMLLabelConfig -->
<View>
  <Header value="Task — span NER + classification"/>
  <Text name="document" value="$text"/>

  <Labels name="ner" toName="document" allowEmpty="true">
    <Label value="PERSON" background="#1f77b4"/>
    <Label value="ORG" background="#ff7f0e"/>
    <Label value="CLAIM_SUBJECT" background="#2ca02c"/>
    <Label value="NONE_OF_THE_ABOVE" background="#7f7f7f"/>
  </Labels>

  <Choices name="classification" toName="document" required="true">
    <Choice value="factual"/>
    <Choice value="opinion"/>
    <Choice value="question"/>
    <Choice value="none_of_the_above"/>
  </Choices>

  <TextArea name="rationale" toName="document"
            editable="true" required="true"
            placeholder="Brief rationale — one sentence is enough."/>
</View>
```

Design points defensible at review:

- **`allowEmpty="true"` on the `Labels` region.** Forcing at least
  one span against `none_of_the_above` taxonomy would push
  annotators to invent entities. The classification `Choices`
  region carries `required="true"` because *every* document has a
  class (even if that class is `none_of_the_above`).
- **Mandatory `rationale` TextArea.** Free-text rationale is the
  disagreement-trace signal mod-107 ex-03 uses to triage κ dips
  per class rather than per annotator.

### 2.5 Label Studio — `label_studio/project.json`

```json
{
  "title": "program-xyz-batch-001",
  "description": "NER + classification. See policy_note.md v1.",
  "color": "#2ca02c",
  "label_config_path": "label_config.xml",
  "sampling": "Uniform sampling",
  "show_annotation_history": false,
  "show_collab_predictions": false,
  "maximum_annotations": 3,
  "enable_empty_annotation": false,
  "show_ground_truth_first": false,
  "reveal_preannotations_interactively": false,
  "overlap_cohort_percentage": 100
}
```

Key choices:

- `maximum_annotations: 3` operationalises the *three independent
  annotators per item* rule from the policy note.
- `show_ground_truth_first: false` and `reveal_preannotations_interactively: false`
  stop the UI from priming annotators with gold labels during a
  production task — gold items are a separate project view.
- `overlap_cohort_percentage: 100` forces every item to be
  annotated by the full `maximum_annotations` quota.

### 2.6 Label Studio — `label_studio/make_project.sh`

```bash
#!/usr/bin/env bash
# Idempotent bootstrap. Running this twice is safe; the project is
# created if it doesn't exist and the tasks are re-imported idempotently
# via the task `data.external_id` field.
set -euo pipefail

: "${LABEL_STUDIO_URL:?LABEL_STUDIO_URL is required}"
: "${LABEL_STUDIO_TOKEN:?LABEL_STUDIO_TOKEN is required}"

PROJECT_TITLE="program-xyz-batch-001"

project_id="$(
  curl -fsSL -H "Authorization: Token ${LABEL_STUDIO_TOKEN}" \
    "${LABEL_STUDIO_URL}/api/projects?title=${PROJECT_TITLE}" \
    | jq -r '.results[0].id // empty'
)"

if [ -z "${project_id}" ]; then
  project_id="$(
    curl -fsSL -X POST \
      -H "Authorization: Token ${LABEL_STUDIO_TOKEN}" \
      -H "Content-Type: application/json" \
      --data @project.json \
      "${LABEL_STUDIO_URL}/api/projects" \
      | jq -r '.id'
  )"
fi

curl -fsSL -X POST \
  -H "Authorization: Token ${LABEL_STUDIO_TOKEN}" \
  -H "Content-Type: application/json" \
  --data-binary @tasks_seed.jsonl \
  "${LABEL_STUDIO_URL}/api/projects/${project_id}/import"

echo "project_id=${project_id}"
```

### 2.7 Argilla — `argilla/settings.yaml`

```yaml
# settings.yaml — reproducible FeedbackDataset schema
name: program-xyz-feedback-batch-001
workspace: program-xyz

fields:
  - name: prompt
    required: true
  - name: response_a
    required: true
  - name: response_b
    required: true

questions:
  - name: rating_a
    type: rating
    values: [1, 2, 3, 4, 5]
    required: true
    description: Rate response A using the Likert anchors in taxonomy.yaml.
  - name: rating_b
    type: rating
    values: [1, 2, 3, 4, 5]
    required: true
  - name: preference
    type: label_selection
    labels: ["A is better", "B is better", "Tie", "Both bad"]
    required: true
  - name: rationale
    type: text
    required: true
    description: One sentence — what tipped the preference.

guidelines: |
  See policy_note.md. Use the Likert anchors from taxonomy.yaml§rating_scale.
  "Tie" is only valid when both responses would score the same rating; it
  is not a shortcut for "I don't want to pick".

metadata_properties:
  - name: source
    type: terms
    values: ["chat_reviews_prod", "chat_reviews_gold", "chat_reviews_canary"]

allow_extra_metadata: false
```

Design points:

- **Fields vs. metadata.** `prompt` / `response_a` / `response_b` are
  **fields** (displayed to annotators). `source` is **metadata**
  (used for filtering, invisible on the task page) so that annotators
  cannot game canary accuracy by recognising canary metadata.
- **`allow_extra_metadata: false`.** Locking metadata keys forces
  additions to flow through a schema change rather than silently
  creeping in at ingest time.

### 2.8 Argilla — `argilla/make_dataset.py`

```python
# make_dataset.py — idempotent Argilla dataset bootstrap.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import argilla as rg
import yaml


def build_settings(settings_path: Path) -> rg.Settings:
    cfg = yaml.safe_load(settings_path.read_text())
    questions: list[rg.QuestionBase] = []
    for q in cfg["questions"]:
        if q["type"] == "rating":
            questions.append(rg.RatingQuestion(
                name=q["name"], values=q["values"], required=q["required"],
                description=q.get("description", ""),
            ))
        elif q["type"] == "label_selection":
            questions.append(rg.LabelQuestion(
                name=q["name"], labels=q["labels"], required=q["required"],
            ))
        elif q["type"] == "text":
            questions.append(rg.TextQuestion(
                name=q["name"], required=q["required"],
                description=q.get("description", ""),
            ))
        else:
            raise ValueError(f"unsupported question type: {q['type']}")

    fields = [rg.TextField(name=f["name"], required=f["required"])
              for f in cfg["fields"]]

    metadata = [rg.TermsMetadataProperty(name=m["name"], options=m["values"])
                for m in cfg.get("metadata_properties", [])]

    return rg.Settings(
        fields=fields, questions=questions, metadata=metadata,
        guidelines=cfg["guidelines"],
        allow_extra_metadata=cfg.get("allow_extra_metadata", False),
    )


def push(records_path: Path, dataset: rg.Dataset) -> int:
    count = 0
    records: list[rg.Record] = []
    for line in records_path.read_text().splitlines():
        row = json.loads(line)
        records.append(rg.Record(
            fields={
                "prompt": row["prompt"],
                "response_a": row["response_a"],
                "response_b": row["response_b"],
            },
            metadata={"source": row["source"]},
            external_id=row["external_id"],  # dedup key — idempotent reruns
        ))
        count += 1
    dataset.records.log(records)
    return count


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--settings", required=True, type=Path)
    ap.add_argument("--records", required=True, type=Path)
    ap.add_argument("--gold", required=True, type=Path)
    ap.add_argument("--api-url", required=True)
    ap.add_argument("--api-key", required=True)
    args = ap.parse_args()

    rg.init(api_url=args.api_url, api_key=args.api_key)
    cfg = yaml.safe_load(args.settings.read_text())

    dataset = rg.Dataset(
        name=cfg["name"], workspace=cfg["workspace"],
        settings=build_settings(args.settings),
    )
    dataset.create()  # safe to call repeatedly; raises only on schema drift

    n_prod = push(args.records, dataset)
    n_gold = push(args.gold, dataset)
    print(json.dumps(
        {"production_records": n_prod, "gold_records": n_gold}, indent=2))


if __name__ == "__main__":
    main()
```

Argilla's SDK API shape varies across minor releases; pin the SDK in
`requirements.lock` and treat the import paths above as the contract
you'll upgrade against. `external_id` is the idempotency key — the
SDK deduplicates on `(workspace, dataset, external_id)` on subsequent
pushes, so reruns are safe.

### 2.9 Gold set construction

- **Size:** ≥ 25 items for calibration seating, ≥ 3% of each
  production batch as inline canaries.
- **Sourcing:** gold items are **adjudicated by the program lead**
  (not by any of the three production annotators) in a prior
  session, with the final answer stored in `gold_items.jsonl` /
  `gold_records.jsonl` outside the annotation UI.
- **Rotation:** the canary subset rotates every batch so annotators
  cannot memorise answers.
- **Separation of concerns:** gold answers are **never** loaded as
  pre-annotations into the production project; they are compared
  server-side after a batch closes.

### 2.10 `runbook.md`

```markdown
# Operator runbook — program <program-id>

## Seating a new annotator
1. Invite in Label Studio AND Argilla (separate seats; same email).
2. Assign the **calibration project** only (gold items, reviewer-mode OFF).
3. Annotator must reach **≥ 0.90 gold accuracy** before seating in prod.
4. Send accuracy report + a ten-item worked walkthrough back to the vendor.

## Daily operation
- Reviewer adjudicates the disagreement queue first thing.
- Volume meter posts hourly; a > 2x spike vs. the vendor's committed rate
  triggers a canary-accuracy spot-check.

## Closing a batch
1. Freeze intake (`PATCH /api/projects/<id>` with `enable_task_sampling=false`).
2. Export via `label-studio-converter` to COCO/JSONL; export Argilla via
   `dataset.records(with_responses=True).to_hf_dataset()` → JSONL.
3. Compute IAA (mod-107 ex-03). A batch failing policy thresholds *does
   not ship*; it reopens with updated guidelines.
4. Generate `batch_manifest.json` (counts, hashes, operator, SLA metrics)
   and sign with cosign; store under `exports/`.
5. Notify mod-109 (lineage) of the new batch artifact SHA.
```

## 3. Validation steps

1. **Policy predates config.** `git log --follow policy_note.md | tail -1`
   returns a commit older than any file under `label_studio/` or
   `argilla/`. Retro-fitted policies fail review.
2. **Taxonomy is the single source.** `taxonomy.yaml:ner_classes[].id`
   matches the `Label[@value]` set in `label_studio/label_config.xml`
   exactly (no label exists only in the XML).
3. **Three-annotator overlap holds.** In Label Studio API,
   `GET /api/projects/<id>` returns `maximum_annotations == 3` and
   `overlap_cohort_percentage == 100`.
4. **No accidental ground-truth leakage.** `reveal_preannotations_interactively`
   and `show_ground_truth_first` are both `false`; the gold project
   is a *separate* project id with its own UI route.
5. **Argilla schema reproduces from YAML.** On a fresh workspace,
   `make_dataset.py --settings settings.yaml` creates a dataset whose
   `settings.to_dict()` is byte-equal (sorted keys) to the YAML.
6. **Gold seating gate wired.** A deliberately-wrong annotator (seeded
   with reversed labels) fails calibration and is *not* seated;
   reporting shows `gold_accuracy < 0.90` for them.
7. **Canary rate at ~3%.** `jq -r '.[].data.source' tasks_seed.jsonl |
   sort | uniq -c` shows canary share within 2.5–3.5%.
8. **Batch manifest signs.** `cosign sign-blob --key …
   exports/batch_manifest.json` succeeds; `cosign verify-blob` exits 0.
9. **Idempotent bootstrap.** Running `make_project.sh` and
   `make_dataset.py` twice in a row produces no duplicate tasks or
   records (verified by task count equality).
10. **Rationale is mandatory.** Submitting a Label Studio annotation
    with an empty `rationale` returns `400 Bad Request` from the API.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `policy_note.md` pre-dates all UI config; versioned in git                                                      | Yes       |
| 2  | `taxonomy.yaml` is the single source; both tools import the same class IDs                                       | Yes       |
| 3  | Label Studio `XMLLabelConfig` + `project.json` are source-controlled and reproduce the project via API           | Yes       |
| 4  | `maximum_annotations: 3`, `overlap_cohort_percentage: 100`, pre-annotation hiding enabled                         | Yes       |
| 5  | Argilla `FeedbackDataset` builds from `settings.yaml`; `allow_extra_metadata: false`                             | Yes       |
| 6  | Gold set ≥ 25 items; held out of the production UI; reviewer-adjudicated                                         | Yes       |
| 7  | Canary injection at ~3% of production volume; rotates batch-over-batch                                           | Yes       |
| 8  | Consensus rule and κ/α thresholds documented *before* any annotation starts                                      | Yes       |
| 9  | Operator runbook covers seating → daily → close with explicit freeze/sign steps                                   | Yes       |
| 10 | Idempotent bootstrap (both `make_project.sh` and `make_dataset.py`) demonstrated via double-run                   | Yes       |
| 11 | Governance anchors (OWASP ML02, MITRE ATLAS AML.T0020/T0018, NIST AI RMF MEASURE 2.3 / MAP 2.3) cited              | No        |
| 12 | Vendor partition cap (≤ 15% of batch per annotator) enforced in task assignment                                  | Yes       |
| 13 | Batch manifest signed with cosign and handed to mod-109 lineage                                                   | Yes       |
| 14 | Rationale TextArea is required; validated at API level                                                           | No        |

## 5. Common mistakes

- **Clicking the project together in the UI and declaring it done.**
  The *clicks* are not the artifact; the versioned config is. A
  reviewer who cannot reproduce the project from git cannot sign
  it.
- **Letting Label Studio and Argilla carry different class IDs.**
  Any divergence between the two UIs makes `mod-109` lineage
  impossible — the same label means two different things depending
  on which tool exported it. Import both from `taxonomy.yaml`.
- **Pre-annotating with model output and calling it "weak
  supervision".** Pre-annotation primes annotators toward the model
  and inflates κ without improving the dataset. Weak supervision
  belongs in `mod-107` ex-05 (Snorkel), not in Label Studio's
  `predictions` field.
- **Mixing gold items into the production project.** Gold answers
  must live in a separate project / workspace so annotators cannot
  spot the canary pattern. A single project with mixed items leaks
  the test.
- **Setting `maximum_annotations: 1` "for speed".** A single
  annotation per item collapses the consensus rule and makes IAA
  (ex-03) impossible.
- **Omitting the rationale field.** Without a per-item rationale,
  κ regressions at batch close are opaque — you see the drop but
  cannot attribute it to a class, a guideline sentence, or an
  annotator cohort.
- **Letting the vendor exceed the per-annotator share cap.** A
  single annotator covering 50% of a batch is a blast-radius
  problem for insider poisoning (ML02), independent of their IAA.
- **Hard-coding Argilla / Label Studio URLs or API tokens.**
  Anything secret belongs in CI secrets and is read at runtime;
  committing a token in `project.json` makes rotation expensive.
- **Treating `allow_extra_metadata: true` as "useful flexibility".**
  Open metadata means filters quietly start depending on keys
  nobody declared, and batch exports fail to round-trip.
- **Skipping the signed batch manifest.** The manifest is the
  lineage input mod-109 verifies; an unsigned export is an
  anonymous dump and cannot enter the training mixture.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Insider annotators and compromised vendors are first-class
  training-data-poisoning actors.
- **MITRE ATLAS — AML.T0020 Poison Training Data; AML.T0018 Backdoor
  ML Model:** <https://atlas.mitre.org/>. Adversary techniques that
  the three-annotator overlap and canary accuracy are meant to deny.
- **NIST AI Risk Management Framework — MEASURE 2.3, MAP 2.3:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 anchors the "measured as intended" bar that an ambiguous
  taxonomy violates; MAP 2.3 anchors the scientific-integrity
  requirement behind reproducible project configs.
- **Label Studio XMLLabelConfig reference:**
  <https://labelstud.io/tags/>. The authoritative tag list, including
  `Labels`, `Choices`, `TextArea`, and the `required`/`allowEmpty`
  attributes used above.
- **Label Studio API — Projects / Import:**
  <https://api.labelstud.io/api-reference/introduction/getting-started>.
  Documents the `maximum_annotations`, `overlap_cohort_percentage`,
  and `enable_empty_annotation` fields used in `project.json`.
- **Argilla documentation — FeedbackDataset, settings, Python SDK:**
  <https://docs.argilla.io/>. The reference for `rg.Settings`,
  `TextField`, `RatingQuestion`, `LabelQuestion`, `TextQuestion`,
  `TermsMetadataProperty`, and `dataset.records.log`.
- **Cohen's κ and Krippendorff's α** (practitioner references in
  `mod-107` ex-03). The threshold choices (κ ≥ 0.70, α ≥ 0.67) in
  this policy note are illustrative starting points; ex-03 computes
  them and justifies their calibration.
- **Local anchors:** `mod-105` (canonical-form conventions reused
  when comparing spans at character offsets); `mod-107` ex-02
  (vendor audit that enforces the canary-drop escalation); `mod-107`
  ex-03 (κ / α computation that gates batch release); `mod-109`
  (Sigstore signing of the batch manifest).
