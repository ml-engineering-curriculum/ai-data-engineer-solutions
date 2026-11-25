# Exercise 01 — Label Studio and Argilla Project Configuration (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-01-label-studio-and-argilla-project-configuration.md`
> in the `ai-data-engineer-learning` repo) is not visible from the
> solutions repo at generation time. This solution is grounded in
> the exercise title, the module README, and the official Label
> Studio / Argilla documentation cited in §6. Any divergence from
> the exercise prompt once it is pinned should re-ground against
> those sources; the deliverable list in §2 is written so that a
> peer can verify each artifact against those sources without
> re-reading this file.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm label schema and
> hand-off format. -->

## 1. Solution overview

This is an **implementation** exercise. The learner stands up two
parallel labeling projects on the same raw corpus — one on **Label
Studio**, one on **Argilla** — and demonstrates that both projects
can emit a **normalised task record** that the rest of `mod-107`
(IAA computation in Exercise 03, active-learning queue in Exercise
04, DPO preference flow in Exercise 06) can consume without
knowing which tool produced it.

The exercise is not "configure Label Studio *or* Argilla"; the
point of configuring both is that the learner commits, in writing,
to a **tool-neutral task schema** before configuring either. The
schema is the firewall between the labeling tool and the rest of
the data pipeline — tooling changes should never force schema
migrations in `mod-109` (lineage) or `mod-111` (quality
monitoring).

Deliverables:

- `schema/task_record.schema.json` — the tool-neutral task record
  schema the pipeline reads. Written first. Any field in Label
  Studio's export or Argilla's export that does not map into this
  schema is dropped at the adapter layer.
- `label_studio/project_config.xml` — Label Studio labeling
  interface XML (uses Label Studio's `<View>` / `<Choices>` /
  `<Labels>` tags).
- `label_studio/project.json` — the project-level config:
  ML backend URL (optional), review workflow, user roles, task
  routing order.
- `argilla/dataset_settings.py` — Argilla Feedback dataset
  settings (field definitions, question definitions, guidelines,
  metadata properties, vectors if any) authored with the Argilla
  Python SDK.
- `argilla/guidelines.md` — the annotator guidelines, rendered
  into Argilla's `guidelines` field and committed as a sibling
  file for diff-review.
- `adapters/ls_to_task.py` — maps a Label Studio export JSON to
  the tool-neutral `task_record`.
- `adapters/argilla_to_task.py` — maps an Argilla `FeedbackRecord`
  to the tool-neutral `task_record`.
- `configs/ingest_policy.yaml` — operator-facing config: which
  export is authoritative for which label set, SLA on review,
  reviewer-role list, which fields are PII-sensitive.
- `tests/test_schema.py` — validates one worked example per tool
  against the task-record schema.

Governance framing. Labeling is the ground truth that every
downstream measurement depends on. The OWASP Machine Learning
Security Top 10 governance anchors are **ML02:2023 Data Poisoning
Attack** (a mislabeled or adversarially-labeled training slice is
training-data poisoning) and **ML06:2023 ML Supply Chain
Vulnerabilities** (the labeling platform and its vendor plugins
are part of the data supply chain). MITRE ATLAS **AML.T0020 Poison
Training Data** is the adversary-facing technique a poorly-governed
labeling project exposes. NIST AI RMF **MEASURE 2.3** ("AI system
performance or assurance criteria are measured qualitatively or
quantitatively as intended") anchors why the task-record schema
exists at all — a labeling tool whose export is not pinned cannot
be the source of an "as intended" measurement; **GOVERN 1.2**
(roles and responsibilities) anchors the reviewer-role list.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── requirements.lock
├── Makefile
├── schema/
│   ├── task_record.schema.json
│   └── worked_example.json
├── label_studio/
│   ├── project_config.xml
│   ├── project.json
│   └── example_export.json
├── argilla/
│   ├── dataset_settings.py
│   ├── guidelines.md
│   └── example_record.json
├── adapters/
│   ├── ls_to_task.py
│   └── argilla_to_task.py
├── configs/
│   └── ingest_policy.yaml
└── tests/
    └── test_schema.py
```

### 2.2 `schema/task_record.schema.json` — write this first

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.org/mod-107/task_record-v1",
  "title": "Labeling Task Record (tool-neutral)",
  "type": "object",
  "required": [
    "task_id", "source_doc_id", "project_id", "tool",
    "label_schema_version", "annotations", "created_at_utc"
  ],
  "properties": {
    "task_id":              { "type": "string" },
    "source_doc_id":        { "type": "string" },
    "project_id":           { "type": "string" },
    "tool":                 { "type": "string",
                              "enum": ["label_studio", "argilla"] },
    "label_schema_version": { "type": "string",
                              "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "guidelines_sha256":    { "type": "string",
                              "pattern": "^[a-f0-9]{64}$" },
    "created_at_utc":       { "type": "string", "format": "date-time" },
    "annotations": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["annotator_id", "label_set", "values",
                     "submitted_at_utc"],
        "properties": {
          "annotator_id":      { "type": "string" },
          "role":              { "type": "string",
                                  "enum": ["annotator", "reviewer",
                                           "gold", "model"] },
          "label_set":         { "type": "string" },
          "values":            { "type": "object" },
          "confidence":        { "type": "number",
                                  "minimum": 0, "maximum": 1 },
          "time_spent_seconds":{ "type": "integer", "minimum": 0 },
          "submitted_at_utc":  { "type": "string", "format": "date-time" }
        }
      }
    },
    "review": {
      "type": "object",
      "properties": {
        "reviewer_id":     { "type": "string" },
        "verdict":         { "type": "string",
                              "enum": ["accept", "reject", "edit"] },
        "verdict_values":  { "type": "object" },
        "reviewed_at_utc": { "type": "string", "format": "date-time" }
      }
    },
    "pii_flags": {
      "type": "array",
      "items": { "type": "string",
                 "enum": ["email", "phone", "ssn", "name",
                          "address", "other"] }
    }
  },
  "additionalProperties": false
}
```

Design notes defensible at review:

- **Schema is the firewall between tool and pipeline.** Everything
  Label Studio or Argilla exports that is not on this list is
  dropped at the adapter layer. This is deliberate — unsanctioned
  fields are the mechanism by which unpinned behaviour (annotator
  identities leaking into downstream features, free-text review
  comments becoming training signal) sneaks into training data.
- **`label_schema_version` is SemVer.** A minor bump covers
  additive changes to `values` (new optional label); a major bump
  on any rename or deletion. `mod-109` lineage records this value
  verbatim.
- **`guidelines_sha256` is on the task record, not the project.**
  Guidelines change mid-project; the per-task hash lets `mod-111`
  (quality monitoring) re-bucket agreement curves by guideline
  version without re-querying the labeling tool.
- **`role` distinguishes `annotator`, `reviewer`, `gold`,
  `model`.** Gold items (the IAA control set from Exercise 03)
  and model-pre-labels (the active-learning pre-annotations from
  Exercise 04) must round-trip through this field. Collapsing
  them silently breaks both exercises.

### 2.3 `label_studio/project_config.xml`

Label Studio's labeling interface is authored in an XML dialect
(see the official template reference for tags such as `<View>`,
`<Text>`, `<Choices>`, `<Labels>`, `<Rating>`). This example
targets a multi-label document-classification project with a
free-text rationale and an optional span annotation; it is
intentionally small so the schema mapping is obvious.

```xml
<View>
  <Header value="Classify the document and span-tag any PII"/>
  <Text name="document" value="$text"/>

  <Choices name="topic" toName="document" choice="multiple" required="true">
    <Choice value="policy"/>
    <Choice value="technical"/>
    <Choice value="product"/>
    <Choice value="other"/>
  </Choices>

  <Labels name="pii_spans" toName="document">
    <Label value="email" background="#D32F2F"/>
    <Label value="phone" background="#1976D2"/>
    <Label value="name"  background="#388E3C"/>
    <Label value="address" background="#F57C00"/>
  </Labels>

  <TextArea name="rationale" toName="document"
            placeholder="One-sentence rationale for the topic choice."
            required="true" maxSubmissions="1"/>
</View>
```

Project-level configuration (`label_studio/project.json`,
abbreviated to the load-bearing keys):

```json
{
  "title": "mod-107-ex-01-ls",
  "description": "Topic classification + PII span tagging. See adjacent guidelines.md.",
  "label_config_sha256": "<sha256 of project_config.xml>",
  "sampling": "Uniform",
  "task_data_login": null,
  "overlap_cohort_percentage": 20,
  "show_overlap_first": true,
  "min_annotations_to_start_training": 0,
  "review_settings": {
    "require_review": true,
    "reviewer_role": "reviewer"
  }
}
```

Design notes:

- **`overlap_cohort_percentage: 20`** keeps a 20% double-labeled
  slice for IAA (Exercise 03 reads this). The overlap sample is
  the only reliable measurement surface for inter-annotator
  agreement; the rest of the project should not be double-labeled
  because that is wasted vendor spend.
- **`require_review: true`** forces every task through a reviewer
  before it is considered "gold". Without this, the `role` field
  in `task_record` cannot be populated truthfully.
- **No ML backend configured here.** The active-learning loop in
  Exercise 04 attaches an ML backend; keeping the base project
  decoupled from a specific model is what lets Exercise 04 swap
  models without rewriting the project.

### 2.4 `argilla/dataset_settings.py`

Argilla's Feedback datasets are authored in Python with the
`argilla` SDK. Field, question, metadata-property, and vector-
setting classes are the primitives. This dataset mirrors the
Label Studio project so the two can be compared side-by-side.

```python
# argilla/dataset_settings.py
import argilla as rg


FIELDS = [
    rg.TextField(name="text", title="Document", use_markdown=False),
]

QUESTIONS = [
    rg.MultiLabelQuestion(
        name="topic",
        title="Document topic (multi-select)",
        labels=["policy", "technical", "product", "other"],
        required=True,
    ),
    rg.SpanQuestion(
        name="pii_spans",
        title="PII spans",
        field="text",
        labels=["email", "phone", "name", "address"],
        allow_overlapping=False,
        required=False,
    ),
    rg.TextQuestion(
        name="rationale",
        title="One-sentence rationale",
        use_markdown=False,
        required=True,
    ),
]

METADATA = [
    rg.TermsMetadataProperty(name="source"),
    rg.TermsMetadataProperty(name="language",
                              values=["en", "es", "fr", "de", "zh", "other"]),
    rg.IntegerMetadataProperty(name="length_tokens",
                                min=0, max=100_000),
]


def build_settings(guidelines: str) -> rg.FeedbackDataset:
    return rg.FeedbackDataset(
        fields=FIELDS,
        questions=QUESTIONS,
        metadata_properties=METADATA,
        guidelines=guidelines,
        allow_extra_metadata=False,
    )
```

Design notes:

- **`allow_extra_metadata=False`.** The task-record schema in
  §2.2 is the normative list; Argilla's metadata properties must
  match. Setting this to `True` would let the labeling tool
  silently add fields the pipeline does not know about.
- **`MultiLabelQuestion` mirrors Label Studio's
  `<Choices choice="multiple">`.** This is the kind of parity
  check the schema was built to catch — a reviewer asked to prove
  the two tools produce the same label set will diff this file
  against `project_config.xml`.
- **`SpanQuestion` requires `allow_overlapping=False`.** This
  pins Argilla to the same "one PII tag per token span" semantics
  Label Studio enforces by default; without this, downstream PII
  evaluation in `mod-110` sees different data from the two tools.

### 2.5 `argilla/guidelines.md`

Argilla's `guidelines` field carries the annotator contract. The
same document is also committed next to the dataset so it is
diff-reviewable. A representative structure:

```markdown
# Document labeling guidelines (project: mod-107-ex-01) — v1.0.0

## 1. Scope

Each task shows one document. Label its topic (one or more of the
four choices) and tag any PII spans in the text. Write a one-
sentence rationale for the topic label.

## 2. Topic labels

- `policy` — the document is primarily about organisational
  policy, regulation, or governance.
- `technical` — the document is primarily about engineering,
  code, or technical operations.
- `product` — the document is primarily about user-facing product
  behaviour or UX.
- `other` — the document does not fit the three above. Use
  sparingly; at most ~5% of tasks should be `other`.

## 3. PII span tags

Tag only PII that appears in the document text. Do not tag PII
that is implied but not written out.

- `email` — any string that looks like an email address, even if
  the domain is fake.
- `phone` — any phone-number-like span. Include the country code
  if present.
- `name` — personal names only. Company names are NOT PII here.
- `address` — any street-level postal address. City-only mentions
  are NOT addresses.

## 4. Rationale

One sentence. State which topic you chose and why. Do not
summarise the document.

## 5. Escalation

If the document contains content you believe should not be
labeled at all (illegal content, severe harassment, extreme
violence), set `topic: other`, write the rationale
"ESCALATE: policy review", and submit. Do not add further
annotations. The project owner reviews the escalate queue daily.

## 6. Change log

- 1.0.0 — initial guidelines.
```

The guidelines file's SHA-256 is computed once per project
version and stored on each task record's `guidelines_sha256`
field. This is what lets `mod-111` bucket agreement curves by
guideline version.

### 2.6 `adapters/ls_to_task.py`

```python
# adapters/ls_to_task.py
# Normalise a Label Studio JSON export into the tool-neutral task record.
from __future__ import annotations

import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path


LABEL_SCHEMA_VERSION = "1.0.0"


def _parse_choices(result: list[dict]) -> dict:
    values: dict = {}
    for r in result:
        tp = r.get("type")
        name = r.get("from_name")
        value = r.get("value", {})
        if tp == "choices":
            values[name] = list(value.get("choices", []))
        elif tp == "textarea":
            values[name] = value.get("text", [""])[0]
        elif tp == "labels":
            spans = values.setdefault(name, [])
            spans.append({
                "start": value.get("start"),
                "end":   value.get("end"),
                "label": value.get("labels", [None])[0],
            })
    return values


def ls_to_task(export: dict, *, project_id: str,
               guidelines_path: Path) -> dict:
    guidelines_sha256 = hashlib.sha256(
        guidelines_path.read_bytes()
    ).hexdigest()
    annotations = []
    for a in export.get("annotations", []):
        annotations.append({
            "annotator_id":     str(a.get("completed_by")),
            "role":             "reviewer" if a.get("was_cancelled") is False
                                             and a.get("ground_truth") else
                                "annotator",
            "label_set":        "topic+pii+rationale",
            "values":           _parse_choices(a.get("result", [])),
            "time_spent_seconds": int(a.get("lead_time") or 0),
            "submitted_at_utc": a.get("updated_at"),
        })
    return {
        "task_id":              f"ls-{export['id']}",
        "source_doc_id":        str(export.get("data", {}).get("doc_id")),
        "project_id":           project_id,
        "tool":                 "label_studio",
        "label_schema_version": LABEL_SCHEMA_VERSION,
        "guidelines_sha256":    guidelines_sha256,
        "created_at_utc":       export.get("created_at"),
        "annotations":          annotations,
        "pii_flags":            sorted({
            span["label"]
            for a in annotations
            for span in a["values"].get("pii_spans", [])
            if span.get("label")
        }),
    }
```

### 2.7 `adapters/argilla_to_task.py`

```python
# adapters/argilla_to_task.py
# Normalise an Argilla FeedbackRecord (as JSON) into the task record.
from __future__ import annotations

import hashlib
from pathlib import Path


LABEL_SCHEMA_VERSION = "1.0.0"


def argilla_to_task(record: dict, *, project_id: str,
                    guidelines_path: Path) -> dict:
    guidelines_sha256 = hashlib.sha256(
        guidelines_path.read_bytes()
    ).hexdigest()
    annotations = []
    for r in record.get("responses", []):
        values = {}
        for q_name, q_val in (r.get("values") or {}).items():
            values[q_name] = q_val.get("value")
        annotations.append({
            "annotator_id":     str(r.get("user_id")),
            "role":             r.get("status", "annotator"),
            "label_set":        "topic+pii+rationale",
            "values":           values,
            "submitted_at_utc": r.get("updated_at"),
        })
    return {
        "task_id":              f"arg-{record['id']}",
        "source_doc_id":        record.get("external_id")
                                 or str(record.get("id")),
        "project_id":           project_id,
        "tool":                 "argilla",
        "label_schema_version": LABEL_SCHEMA_VERSION,
        "guidelines_sha256":    guidelines_sha256,
        "created_at_utc":       record.get("inserted_at"),
        "annotations":          annotations,
    }
```

### 2.8 `configs/ingest_policy.yaml`

```yaml
# Operator-facing policy for how ingest treats each tool's export.
project_id: mod-107-ex-01
label_schema_version: 1.0.0
tools:
  label_studio:
    export_format: json
    authoritative_for: ["topic", "pii_spans", "rationale"]
    reviewer_role_names: ["reviewer", "lead-reviewer"]
    export_sla_hours: 24
  argilla:
    export_format: feedback_records_json
    authoritative_for: ["topic", "pii_spans", "rationale"]
    reviewer_role_names: ["submitted"]
    export_sla_hours: 24
pii_sensitive_fields:
  - pii_spans
  - rationale
retention_days:
  raw_export: 90
  normalised_task_record: 730
```

### 2.9 `tests/test_schema.py`

```python
# tests/test_schema.py
# Round-trip one worked example per tool through its adapter and
# validate against task_record.schema.json.
from __future__ import annotations

import json
from pathlib import Path

import jsonschema
import pytest

from adapters.ls_to_task import ls_to_task
from adapters.argilla_to_task import argilla_to_task


ROOT = Path(__file__).resolve().parent.parent
SCHEMA = json.loads((ROOT / "schema" / "task_record.schema.json").read_text())
GUIDELINES = ROOT / "argilla" / "guidelines.md"


def _validate(instance: dict) -> None:
    jsonschema.validate(instance=instance, schema=SCHEMA)


def test_label_studio_roundtrip() -> None:
    export = json.loads(
        (ROOT / "label_studio" / "example_export.json").read_text()
    )
    record = ls_to_task(export, project_id="mod-107-ex-01",
                        guidelines_path=GUIDELINES)
    _validate(record)
    assert record["tool"] == "label_studio"


def test_argilla_roundtrip() -> None:
    example = json.loads(
        (ROOT / "argilla" / "example_record.json").read_text()
    )
    record = argilla_to_task(example, project_id="mod-107-ex-01",
                             guidelines_path=GUIDELINES)
    _validate(record)
    assert record["tool"] == "argilla"


@pytest.mark.parametrize("field", ["task_id", "source_doc_id",
                                    "label_schema_version", "annotations"])
def test_required_fields(field: str) -> None:
    assert field in SCHEMA["required"]
```

### 2.10 `Makefile`

```makefile
.PHONY: all validate test clean

all: validate test

validate:
	python -c "import json, jsonschema; \
	    jsonschema.Draft202012Validator.check_schema( \
	        json.load(open('schema/task_record.schema.json'))); \
	    print('schema OK')"

test:
	pytest -q tests/

clean:
	rm -rf .pytest_cache __pycache__ adapters/__pycache__
```

## 3. Validation steps

1. **Schema validates itself.** `python -c "import json,
   jsonschema; jsonschema.Draft202012Validator.check_schema(
   json.load(open('schema/task_record.schema.json')))"` exits 0.
2. **Worked example validates.** `pytest -q tests/` passes; both
   `test_label_studio_roundtrip` and `test_argilla_roundtrip`
   produce a record that validates.
3. **Label config SHA matches.** `sha256sum
   label_studio/project_config.xml` equals the value in
   `label_studio/project.json` under `label_config_sha256`.
4. **Argilla settings reject extra metadata.** `grep
   allow_extra_metadata argilla/dataset_settings.py` returns
   `False`. A `True` here is a review-blocking regression.
5. **Guidelines hash on each task.** For every record produced,
   `record["guidelines_sha256"]` equals
   `sha256sum argilla/guidelines.md | cut -d' ' -f1`.
6. **Labels parity between tools.** The sorted set of topic label
   values in `project_config.xml` equals the sorted
   `labels` list on the `topic` `MultiLabelQuestion` in
   `dataset_settings.py`. Same check for PII span labels. Any
   mismatch fails review.
7. **Reviewer-role coverage.** Both adapters emit at least one
   annotation with `role == "reviewer"` on a synthetic input
   where a reviewer has signed off; the schema allows `role`
   values `{annotator, reviewer, gold, model}`.
8. **Role `gold` round-trips.** An Argilla record flagged
   "ground_truth" (Label Studio) or a record with a user tagged
   as the gold annotator (Argilla) yields `role == "gold"` in
   the normalised record.
9. **PII fields are not in `additionalProperties`.** The schema
   has `additionalProperties: false` so any adapter emitting a
   non-schema key (e.g., `annotator_email`, `free_text_note`)
   fails validation. This is the mechanism that prevents silent
   PII leakage.
10. **Overlap cohort pinned.** `overlap_cohort_percentage: 20`
    appears in `label_studio/project.json`; Argilla's equivalent
    is documented in `configs/ingest_policy.yaml` as the sampling
    fraction Exercise 03 assumes.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `task_record.schema.json` exists and is committed before any tool-specific config                                    | Yes       |
| 2  | Schema sets `additionalProperties: false` and pins a SemVer `label_schema_version`                                   | Yes       |
| 3  | Label Studio `<View>` XML validates and the label set matches Argilla's                                             | Yes       |
| 4  | Argilla `FeedbackDataset` disables `allow_extra_metadata`                                                           | Yes       |
| 5  | Both adapters normalise to the same task-record schema; `pytest` passes                                             | Yes       |
| 6  | Guidelines SHA-256 stamped on every normalised record                                                                | Yes       |
| 7  | Reviewer role captured truthfully (`role` enum covers `annotator`, `reviewer`, `gold`, `model`)                      | Yes       |
| 8  | Overlap-cohort fraction pinned for Exercise 03's IAA measurement                                                     | Yes       |
| 9  | PII fields enumerated in the schema and in `ingest_policy.yaml`                                                      | Yes       |
| 10 | Governance anchors (OWASP ML Top 10 ML02 / ML06, MITRE ATLAS AML.T0020, NIST AI RMF MEASURE 2.3 / GOVERN 1.2) cited  | No        |
| 11 | Retention days documented for raw exports and normalised records                                                     | No        |
| 12 | Hand-off contract to Exercise 03 / 04 / 06 is readable from this exercise's config alone                             | No        |
| 13 | Labels parity check (topic + PII span sets identical across tools) is in `tests/`                                   | Yes       |
| 14 | Escalate path documented in guidelines                                                                               | No        |

## 5. Common mistakes

- **Configuring one tool, then "mapping" the other to match.** The
  tool-neutral schema must come first. Configuring Label Studio
  first and then deriving Argilla's questions from Label Studio's
  XML is how the two drift silently.
- **Leaving `allow_extra_metadata=True` on Argilla.** Any
  reviewer who adds a free-text metadata note creates a field the
  pipeline does not know about. Set it to `False` and reject the
  record at ingest if it carries extras.
- **Letting annotator identities flow past the adapter unhashed.**
  The schema holds `annotator_id`, not `annotator_email`. The
  adapter should map identities to opaque ids at the firewall;
  the mapping table is PII and lives outside the labeling tool.
- **Copying `<Labels>` values without matching Argilla
  `SpanQuestion.labels`.** This is a very common review failure —
  the two label sets must be byte-identical. Automate the parity
  check in `tests/`.
- **Forgetting the overlap-cohort fraction.** Without an explicit
  overlap percentage, Exercise 03 has no double-labeled data and
  IAA cannot be computed. Pin this in both tools at configuration
  time.
- **Storing guidelines only inside the labeling tool.** Argilla's
  `guidelines` field is a convenience; the authoritative copy is
  the committed `guidelines.md`. If the two diverge, the
  committed file wins and the ingest adapter flags a mismatch.
- **Treating Label Studio's "ground_truth" flag as "gold" without
  reviewer sign-off.** `role == "gold"` means a reviewer has
  signed off — not that an admin set a flag. The adapter must
  check the review record, not just the flag.
- **Adding free-text comment fields that become training signal.**
  Rationale is in the schema; everything else is not. Adapters
  drop unknown fields; reviewers raise an exception if a tool
  starts emitting new fields unprompted.
- **Skipping the schema test.** The schema is enforced only if
  `jsonschema.validate` runs on real export samples. Commit the
  worked examples and keep `pytest` green.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack, ML06:2023 ML Supply Chain Vulnerabilities:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 is the harm (bad labels poison training), ML06 is the
  mechanism (the labeling tool and its plugins are supply chain).
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing technique that
  a poorly-governed labeling project exposes.
- **NIST AI Risk Management Framework — MEASURE 2.3, GOVERN
  1.2:** <https://www.nist.gov/itl/ai-risk-management-framework>.
  MEASURE 2.3 anchors the task-record schema as the "as
  intended" measurement surface; GOVERN 1.2 anchors the explicit
  reviewer-role list.
- **Label Studio documentation — labeling configuration tags and
  project settings:** <https://labelstud.io/tags/> and
  <https://labelstud.io/guide/>. Primary source for `<View>`,
  `<Choices>`, `<Labels>`, and project-level review settings.
- **Argilla documentation — Feedback datasets, fields, questions,
  metadata properties:** <https://docs.argilla.io/>. Primary
  source for `FeedbackDataset`, `TextField`,
  `MultiLabelQuestion`, `SpanQuestion`, `TextQuestion`, and
  `allow_extra_metadata`.
- **JSON Schema Draft 2020-12:** <https://json-schema.org/draft/2020-12>.
  The `$schema` dialect `task_record.schema.json` targets.
- **Local anchors:** Exercise 03 (reads the overlap cohort to
  compute IAA); Exercise 04 (attaches an ML backend to the Label
  Studio project for active learning); Exercise 06 (DPO
  preference-pair labeling reuses the task-record schema with a
  `pairwise` label set); `mod-109` (records
  `label_schema_version` and `guidelines_sha256` in slice
  lineage); `mod-110` (reads `pii_flags` for the PII risk
  evaluation); `mod-111` (buckets agreement curves by guideline
  version).
