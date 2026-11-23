# Exercise 06 — Preference Labelling Program for DPO Pairs (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-06-preference-labelling-program-for-dpo-pairs.md`)
> is not visible from the solutions repo at generation time.
> This solution is grounded in the exercise title, the module
> README, and the official sources cited in §6 (Rafailov et al.
> DPO paper, Label Studio / Argilla docs, NIST AI RMF, OWASP ML
> Top 10). Any divergence from the exercise prompt once it is
> pinned should re-ground against those sources.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm the pair count and
> hand-off format expected by `mod-108` exercise-04. -->

## 1. Solution overview

This is a **design + implementation** exercise. The learner
builds a **preference-labelling program** that produces DPO
training pairs. "Program" is the operative word: the exercise
is not "label some pairs" but "stand up the reusable program a
team of human labellers runs every week to produce reviewable,
diversity-balanced, bias-controlled preference data that
downstream fine-tuning can trust".

Preference labelling has specific failure modes that ordinary
classification labelling does not:

- **Position bias** — labellers favour the side shown first (or
  second) regardless of content.
- **Length bias** — labellers favour the longer response because
  it looks "more thorough".
- **Judge-class drift** — multiple labellers resolve toward a
  dominant house style, not a quality signal.
- **Tie suppression** — labellers are asked to pick A or B even
  when neither answer is adequate; the forced choice becomes
  training signal.

The program must control all four.

Deliverables:

- `PROGRAM.md` — the program design document. Six sections,
  signed by ai-data-engineer and the fine-tuning-engineer who
  consumes the pairs.
- `label_studio/pair_project_config.xml` — Label Studio labeling
  interface for a position-swap pairwise comparison with a
  `tie` choice and a free-text rationale.
- `argilla/pair_dataset_settings.py` — Argilla equivalent.
- `schema/preference_pair.schema.json` — hand-off schema the
  pair-labelled records are normalised into (extends Exercise 01's
  `task_record` with a `pairwise` label set).
- `code/sample_prompts.py` — prompt sampling for the pool (de-
  duplication, diversity filters, stratification).
- `code/assign_pairs.py` — position-swap assignment: every
  (prompt, A, B) task is replicated as (prompt, B, A) and
  routed to a different labeller.
- `code/aggregate.py` — merges the two position-swap verdicts
  into one normalised pair (or flags position-bias disagreement).
- `code/qa_metrics.py` — position-bias, length-bias, and tie-
  rate metrics per labeller and per batch.
- `code/export_dpo.py` — emits the DPO `.parquet` / `.jsonl`
  the fine-tuning module consumes.
- `outputs/pair_qa_report.yaml` — §1 identity, §2 program
  summary, §3 per-labeller metrics, §4 pair-balance report, §5
  recommendation, §6 pending-review.
- `tests/` — determinism tests, schema validation, position-swap
  round-trip test.

Governance framing. Preference pairs are the training signal the
policy is literally optimised against. Any bias in the pair
corpus — position, length, house style — becomes a bias the
policy learns and that cannot be removed after training without
replaying DPO. Downstream training (`mod-108` exercise-04) depends
on this program to deliver a signal that measures what the team
believes it measures.

- OWASP Machine Learning Security Top 10 anchor: **ML02:2023
  Data Poisoning Attack** — biased preference labels poison
  the policy by rewarding surface artifacts.
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data**
  (downstream harm), **AML.T0018.000 Manipulate AI Model
  (Modify AI Model)** (DPO is the modification; the pair corpus
  is the modification surface).
- NIST AI RMF controls: **MEASURE 2.3** (performance as
  intended) — position/length bias is measured as the primary
  measurement surface; **MEASURE 2.11** (fairness and bias) —
  per-labeller and per-slice bias is the review signal;
  **GOVERN 1.2** (roles and responsibilities) — the hand-off
  between the labelling program and fine-tuning is role-
  delineated.

Cite the framework URLs rather than paraphrasing the control text.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-06/
├── README.md
├── requirements.lock
├── Makefile
├── PROGRAM.md
├── schema/
│   └── preference_pair.schema.json
├── label_studio/
│   └── pair_project_config.xml
├── argilla/
│   └── pair_dataset_settings.py
├── code/
│   ├── sample_prompts.py
│   ├── assign_pairs.py
│   ├── aggregate.py
│   ├── qa_metrics.py
│   └── export_dpo.py
├── configs/
│   ├── program.yaml
│   └── labellers.yaml
├── tests/
│   ├── test_schema.py
│   ├── test_position_swap.py
│   └── test_aggregate.py
└── outputs/
    ├── pairs.jsonl
    ├── pair_qa_report.yaml
    └── dpo_training.parquet
```

### 2.2 `PROGRAM.md` — the program design document

```markdown
# Preference labelling program — v1

| Field               | Value                                                     |
|---------------------|-----------------------------------------------------------|
| Program version     | 1.0.0                                                     |
| Effective           | 2026-11-17                                                |
| Signatory (producer)| ai-data-engineer (program owner)                          |
| Signatory (consumer)| fine-tuning-engineer                                      |
| Supersedes          | n/a                                                       |
| Review cadence      | every 60 days                                             |

## 1. Scope

This program produces DPO training pairs from a configurable prompt
pool. Each pair has fields:

- `prompt`
- `chosen` (preferred response)
- `rejected`
- `rationale_sha256` (anchors the labeller's rationale off-line)
- `labeller_set` (which labellers saw the pair)
- `position_swap_agreement` (true/false — see §4)

Out of scope: safety-only labels (routing a response as
"unsafe" rather than comparing two), format pairs where both
responses are identical, pairs with model-generated rationales.

## 2. Prompt pool

(a) Source. The prompt pool is seeded by the `prompt_pool.parquet`
output of a prior module (e.g., `mod-103` curation output or an
earlier-stage labelled SFT prompt set).
(b) Deduplication. Near-duplicate prompts are removed via the
MinHash recipe from `mod-105`. Pair data is extremely sensitive
to prompt duplication because the same prompt labelled twice
amplifies any bias.
(c) Diversity filter. The program caps at most 2% of pairs per
intent cluster (clusters derived from an off-the-shelf embedding
model — document the model and version). This prevents the
corpus from collapsing onto a popular topic.
(d) Stratification. Each batch is stratified across difficulty
buckets (easy / medium / hard) and across prompt-source buckets.

## 3. Response pair generation

(a) Response A and Response B are produced by distinct generators
(or the same generator at distinct temperatures). This program
does NOT prescribe the generators; it prescribes that A and B
differ in model / temperature / decoding / prompt variant, and
that the difference is recorded in the task record.
(b) The pair record carries `response_a.generator`,
`response_b.generator`, `response_a.params`, `response_b.params`.
(c) No labeller sees the generator identity. Blinding is
enforced at the labeling-tool level.

## 4. Labelling contract

(a) Each (prompt, A, B) is labelled twice: once in the original
order, once with positions swapped (A↔B). The two instances are
routed to different labellers.
(b) A labeller picks one of `A_better | B_better | tie | reject`.
`reject` means the pair is unusable (both answers wrong,
off-topic, unsafe, or unlabellable as a pair).
(c) A one-sentence rationale is required on every non-`tie`
verdict.
(d) `tie` is a first-class answer. Hidden ties — labellers
forced to break a tie to meet a quota — are what poison DPO.
(e) IAA target. The program commits to Cohen's kappa ≥ 0.50 on
`A_better | B_better` (ignoring `tie` and `reject`) on a
double-labelled cohort, measured per labeller-pair, using the
code from Exercise 03.

## 5. Bias controls

(a) Position swap. Position bias is measurable and must be
reported per labeller per batch. The pairwise verdict is only
used in training when the two position-swap verdicts agree (or
both are `tie`).
(b) Length-bias check. For each labeller, report correlation
between "picked longer response" and "picked response A".
Labeller-level drift above a documented threshold is flagged.
(c) Judge-class drift. The program periodically re-labels a
"canary" set with known answers (jointly agreed by the program
owner and the fine-tuning-engineer); drop in canary accuracy
triggers a calibration session.

## 6. Hand-off

(a) Format. `dpo_training.parquet` with fields `prompt`,
`chosen`, `rejected`, `source_pair_id`, `labeller_set`,
`difficulty_bucket`.
(b) The parquet is signed with cosign; the hand-off record in
`mod-109` references the signature.
(c) Dispute path. If `mod-108` exercise-04's calibration run
finds the pairs degrade the policy, a joint program-owner /
fine-tuning-engineer review within five business days.

## 7. Signatures

<cosign signature paths>
```

The program document is the normative artifact. The code and
schemas enforce what it promises; the QA report measures it.

### 2.3 `schema/preference_pair.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.org/mod-107/preference_pair-v1",
  "title": "Preference Pair Record",
  "type": "object",
  "required": [
    "pair_id", "prompt", "response_a", "response_b",
    "verdicts", "created_at_utc", "program_version"
  ],
  "properties": {
    "pair_id":           { "type": "string" },
    "program_version":   { "type": "string",
                            "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "prompt":            { "type": "string" },
    "prompt_metadata": {
      "type": "object",
      "properties": {
        "intent_cluster":  { "type": "string" },
        "difficulty":      { "type": "string",
                              "enum": ["easy", "medium", "hard"] },
        "source_bucket":   { "type": "string" }
      }
    },
    "response_a": { "$ref": "#/$defs/response" },
    "response_b": { "$ref": "#/$defs/response" },
    "verdicts": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["labeller_id", "position_order", "verdict",
                     "submitted_at_utc"],
        "properties": {
          "labeller_id":     { "type": "string" },
          "position_order":  { "type": "string", "enum": ["AB", "BA"] },
          "verdict":         { "type": "string",
                                "enum": ["A_better", "B_better",
                                         "tie", "reject"] },
          "rationale_sha256":{ "type": "string",
                                "pattern": "^[a-f0-9]{64}$" },
          "time_spent_seconds":{ "type": "integer", "minimum": 0 },
          "submitted_at_utc":{ "type": "string", "format": "date-time" }
        }
      }
    },
    "position_swap_agreement": { "type": "boolean" },
    "final_verdict":   { "type": "string",
                          "enum": ["A_better", "B_better", "tie",
                                   "reject", "disputed"] },
    "created_at_utc":  { "type": "string", "format": "date-time" },
    "pii_flags":       { "type": "array",
                          "items": { "type": "string" } }
  },
  "$defs": {
    "response": {
      "type": "object",
      "required": ["text", "generator"],
      "properties": {
        "text":      { "type": "string" },
        "generator": {
          "type": "object",
          "required": ["model_id"],
          "properties": {
            "model_id":    { "type": "string" },
            "temperature": { "type": "number", "minimum": 0 },
            "max_tokens":  { "type": "integer", "minimum": 1 },
            "params_sha256":{ "type": "string",
                                "pattern": "^[a-f0-9]{64}$" }
          }
        }
      }
    }
  },
  "additionalProperties": false
}
```

### 2.4 `label_studio/pair_project_config.xml`

Label Studio's `<Choices>` plus side-by-side `<Text>` panels
implement the pairwise task. Generator identity and side
identity (A vs B) are deliberately not shown to the labeller.

```xml
<View>
  <Header value="Which response better satisfies the prompt?"/>
  <View style="display: flex; gap: 16px;">
    <View style="flex: 1 1 50%;">
      <Header value="Response 1"/>
      <Text name="response_left" value="$response_left"/>
    </View>
    <View style="flex: 1 1 50%;">
      <Header value="Response 2"/>
      <Text name="response_right" value="$response_right"/>
    </View>
  </View>
  <Header value="Original prompt"/>
  <Text name="prompt" value="$prompt"/>

  <Choices name="verdict" toName="prompt" choice="single" required="true">
    <Choice value="left_better"/>
    <Choice value="right_better"/>
    <Choice value="tie"/>
    <Choice value="reject"/>
  </Choices>

  <TextArea name="rationale" toName="prompt"
            placeholder="One sentence. Required unless verdict is 'tie'."
            maxSubmissions="1"/>
</View>
```

Design notes:

- **Side labels are `left` / `right`, not `A` / `B`.** The
  adapter translates to `A_better` / `B_better` using the task-
  level `position_order` field so the labeller cannot see the
  authoritative-order mapping.
- **`reject` is first-class.** A pair where both responses are
  wrong or unsafe is a `reject`; forcing a choice contaminates
  the training signal.
- **`tie` does not require a rationale.** Rationales on ties are
  often "both are fine" and contribute no value; making them
  optional reduces labeller cost without losing signal.

### 2.5 `argilla/pair_dataset_settings.py`

```python
# argilla/pair_dataset_settings.py
import argilla as rg


FIELDS = [
    rg.TextField(name="response_left", use_markdown=False),
    rg.TextField(name="response_right", use_markdown=False),
    rg.TextField(name="prompt", use_markdown=False),
]

QUESTIONS = [
    rg.LabelQuestion(
        name="verdict",
        title="Which response is better?",
        labels=["left_better", "right_better", "tie", "reject"],
        required=True,
    ),
    rg.TextQuestion(
        name="rationale",
        title="One-sentence rationale (optional for ties)",
        use_markdown=False,
        required=False,
    ),
]

METADATA = [
    rg.TermsMetadataProperty(name="position_order"),
    rg.TermsMetadataProperty(name="difficulty",
                              values=["easy", "medium", "hard"]),
    rg.TermsMetadataProperty(name="intent_cluster"),
    rg.IntegerMetadataProperty(name="length_delta_tokens",
                                min=-10_000, max=10_000),
]


def build_dataset(guidelines: str) -> rg.FeedbackDataset:
    return rg.FeedbackDataset(
        fields=FIELDS,
        questions=QUESTIONS,
        metadata_properties=METADATA,
        guidelines=guidelines,
        allow_extra_metadata=False,
    )
```

### 2.6 `code/sample_prompts.py`

```python
# code/sample_prompts.py
# Sample prompts for the pair-labelling pool. Deduplicate, cluster,
# stratify.
from __future__ import annotations

import argparse
import random
from pathlib import Path

import pandas as pd


def sample(prompts: pd.DataFrame, *, n_per_cluster_cap: int,
           per_bucket_counts: dict[str, int], seed: int) -> pd.DataFrame:
    rng = random.Random(seed)
    rows: list[dict] = []
    for bucket, count in per_bucket_counts.items():
        sub = prompts[prompts["difficulty"] == bucket]
        picked_per_cluster: dict[str, int] = {}
        order = list(sub.index)
        rng.shuffle(order)
        taken = 0
        for idx in order:
            if taken >= count:
                break
            cluster = str(sub.loc[idx, "intent_cluster"])
            if picked_per_cluster.get(cluster, 0) >= n_per_cluster_cap:
                continue
            picked_per_cluster[cluster] = picked_per_cluster.get(cluster, 0) + 1
            rows.append(sub.loc[idx].to_dict())
            taken += 1
    return pd.DataFrame(rows)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pool",    required=True, type=Path)
    ap.add_argument("--seed",    type=int, required=True)
    ap.add_argument("--cap",     type=int, required=True,
                    help="Max prompts per intent cluster")
    ap.add_argument("--easy",    type=int, required=True)
    ap.add_argument("--medium",  type=int, required=True)
    ap.add_argument("--hard",    type=int, required=True)
    ap.add_argument("--out",     required=True, type=Path)
    args = ap.parse_args()
    prompts = pd.read_parquet(args.pool)
    picked = sample(prompts, n_per_cluster_cap=args.cap,
                     per_bucket_counts={"easy": args.easy,
                                         "medium": args.medium,
                                         "hard": args.hard},
                     seed=args.seed)
    picked.to_parquet(args.out, index=False)
```

### 2.7 `code/assign_pairs.py`

```python
# code/assign_pairs.py
# Replicate each (prompt, A, B) task as (prompt, B, A) and route
# the two instances to different labellers.
from __future__ import annotations

import argparse
import json
import random
from pathlib import Path

import pandas as pd


def assign(pairs: pd.DataFrame, labellers: list[str], seed: int) -> list[dict]:
    rng = random.Random(seed)
    rows: list[dict] = []
    for _, p in pairs.iterrows():
        pair_id = str(p["pair_id"])
        # Draw two distinct labellers per pair, no replacement.
        pool = labellers[:]
        rng.shuffle(pool)
        lab_ab, lab_ba = pool[0], pool[1]
        rows.append({
            "pair_id": pair_id, "position_order": "AB",
            "response_left": p["response_a"],
            "response_right": p["response_b"],
            "prompt": p["prompt"], "labeller_id": lab_ab,
        })
        rows.append({
            "pair_id": pair_id, "position_order": "BA",
            "response_left": p["response_b"],
            "response_right": p["response_a"],
            "prompt": p["prompt"], "labeller_id": lab_ba,
        })
    return rows


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pairs",    required=True, type=Path)
    ap.add_argument("--labellers", required=True, type=Path)
    ap.add_argument("--seed",     type=int, required=True)
    ap.add_argument("--out",      required=True, type=Path)
    args = ap.parse_args()
    pairs = pd.read_parquet(args.pairs)
    labellers = json.loads(args.labellers.read_text())["labellers"]
    rows = assign(pairs, labellers, args.seed)
    args.out.write_text("\n".join(json.dumps(r) for r in rows))
```

Design notes:

- **Two distinct labellers per pair.** If the same labeller
  sees both position orders, position bias is undetectable.
- **Deterministic under seed.** Reruns with the same seed route
  to the same labellers, which is important for QA
  reproducibility.
- **`position_order` is carried on the task**. The adapter at
  ingest reads it and maps `left/right` to `A/B` in the
  normalised record.

### 2.8 `code/aggregate.py`

```python
# code/aggregate.py
# Merge the two position-swap verdicts into one normalised pair.
# Drops into {A_better, B_better, tie, reject, disputed}.
from __future__ import annotations

from typing import Literal


LEFT, RIGHT, TIE, REJECT = "left_better", "right_better", "tie", "reject"
Verdict = Literal["A_better", "B_better", "tie", "reject", "disputed"]


def _translate(verdict: str, order: str) -> str:
    """Translate a side-oriented verdict (left/right) into A/B under
    the known position_order."""
    if verdict in (TIE, REJECT):
        return verdict
    if order == "AB":
        return "A_better" if verdict == LEFT else "B_better"
    # order == "BA": left was B
    return "B_better" if verdict == LEFT else "A_better"


def aggregate(verdicts: list[dict]) -> tuple[Verdict, bool]:
    """Returns (final_verdict, position_swap_agreement).

    Rule set:
    - Any `reject` collapses the pair to `reject`.
    - If both verdicts agree (same A_better / B_better / tie), that's the
      final verdict; position_swap_agreement = True.
    - A_better vs tie (or symmetric cases) → disputed; peer review.
    - A_better vs B_better → disputed (position bias detected or genuine
      disagreement); peer review required.
    """
    if any(v["verdict"] == REJECT for v in verdicts):
        return "reject", False
    translated = [_translate(v["verdict"], v["position_order"])
                  for v in verdicts]
    if len(translated) == 2 and translated[0] == translated[1]:
        return translated[0], True
    return "disputed", False
```

Design notes defensible at review:

- **Disputed is not silently resolved.** A pair whose two
  position-swap verdicts disagree is peer-reviewed; the program
  does not automatically pick one side. This is what prevents
  position bias from leaking into training.
- **`reject` is sticky.** If any labeller says the pair is
  unusable, that's authoritative — overriding a `reject` with
  a `A_better` ingested elsewhere lets unsafe or wrong pairs
  into training.

### 2.9 `code/qa_metrics.py`

```python
# code/qa_metrics.py
# Per-labeller and per-batch QA metrics: position bias, length bias,
# tie rate, reject rate.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pandas as pd


def position_bias(verdicts: pd.DataFrame) -> pd.DataFrame:
    # Fraction of non-tie/reject verdicts where the labeller picked `left_better`.
    sub = verdicts[verdicts["verdict"].isin(["left_better", "right_better"])]
    out = []
    for lab, g in sub.groupby("labeller_id"):
        total = len(g)
        left = int((g["verdict"] == "left_better").sum())
        out.append({"labeller_id": lab, "picked_left_rate": left / total,
                     "n": total})
    return pd.DataFrame(out)


def length_bias(verdicts: pd.DataFrame) -> pd.DataFrame:
    # Fraction where the labeller picked the LONGER of the two.
    sub = verdicts[verdicts["verdict"].isin(["left_better", "right_better"])]
    sub = sub.assign(
        picked_longer=lambda d: (
            (d["verdict"] == "left_better") &
            (d["left_tokens"] > d["right_tokens"])
        ) | (
            (d["verdict"] == "right_better") &
            (d["right_tokens"] > d["left_tokens"])
        )
    )
    out = []
    for lab, g in sub.groupby("labeller_id"):
        total = len(g)
        out.append({"labeller_id": lab,
                     "picked_longer_rate": float(g["picked_longer"].mean()),
                     "n": total})
    return pd.DataFrame(out)


def tie_and_reject_rates(verdicts: pd.DataFrame) -> pd.DataFrame:
    out = []
    for lab, g in verdicts.groupby("labeller_id"):
        total = len(g)
        tie = int((g["verdict"] == "tie").sum())
        reject = int((g["verdict"] == "reject").sum())
        out.append({"labeller_id": lab,
                     "tie_rate": tie / total if total else 0.0,
                     "reject_rate": reject / total if total else 0.0,
                     "n": total})
    return pd.DataFrame(out)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--verdicts", required=True, type=Path,
                    help="Parquet with columns: labeller_id, verdict, "
                         "left_tokens, right_tokens")
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()
    v = pd.read_parquet(args.verdicts)
    pb = position_bias(v)
    lb = length_bias(v)
    tr = tie_and_reject_rates(v)
    merged = pb.merge(lb, on="labeller_id", suffixes=("_pos", "_len"))
    merged = merged.merge(tr, on="labeller_id")
    args.out.write_text(json.dumps(merged.to_dict(orient="records"),
                                    indent=2, sort_keys=True))
```

Flag thresholds (document in `configs/program.yaml`, not hard-
coded here):

- `picked_left_rate` outside `[0.40, 0.60]` → position-bias
  flag.
- `picked_longer_rate` above 0.70 → length-bias flag.
- `tie_rate` under 0.02 on a non-trivial batch → suspected tie-
  suppression (labeller is forcing choices).
- `reject_rate` above 0.15 → labeller is rejecting too liberally;
  calibration session triggered.

### 2.10 `code/export_dpo.py`

```python
# code/export_dpo.py
# Export the final DPO training file. Includes only non-disputed,
# non-reject, non-tie pairs where position_swap_agreement == True.
from __future__ import annotations

import argparse
from pathlib import Path

import pandas as pd


def export(pairs: pd.DataFrame) -> pd.DataFrame:
    usable = pairs[(pairs["final_verdict"].isin(["A_better", "B_better"]))
                    & (pairs["position_swap_agreement"] == True)].copy()
    usable["chosen"] = usable.apply(
        lambda r: r["response_a"] if r["final_verdict"] == "A_better"
                                  else r["response_b"], axis=1)
    usable["rejected"] = usable.apply(
        lambda r: r["response_b"] if r["final_verdict"] == "A_better"
                                   else r["response_a"], axis=1)
    return usable[["pair_id", "prompt", "chosen", "rejected",
                    "labeller_set", "difficulty"]]


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pairs", required=True, type=Path)
    ap.add_argument("--out",   required=True, type=Path)
    args = ap.parse_args()
    pairs = pd.read_parquet(args.pairs)
    export(pairs).to_parquet(args.out, index=False)
```

### 2.11 Tests

`tests/test_schema.py` validates one worked pair against the
JSON Schema.

`tests/test_position_swap.py` round-trips a known pair through
`assign_pairs` and `aggregate`:

```python
# tests/test_position_swap.py
from code.aggregate import aggregate


def test_agreement_translates_correctly() -> None:
    # Labeller A saw AB, chose left_better → translates to A_better.
    # Labeller B saw BA, chose right_better → also translates to A_better.
    # Expected: A_better, position_swap_agreement=True.
    verdicts = [
        {"verdict": "left_better",  "position_order": "AB"},
        {"verdict": "right_better", "position_order": "BA"},
    ]
    assert aggregate(verdicts) == ("A_better", True)


def test_disagreement_flags_disputed() -> None:
    verdicts = [
        {"verdict": "left_better",  "position_order": "AB"},
        {"verdict": "left_better",  "position_order": "BA"},
    ]
    assert aggregate(verdicts) == ("disputed", False)


def test_any_reject_wins() -> None:
    verdicts = [
        {"verdict": "left_better", "position_order": "AB"},
        {"verdict": "reject",      "position_order": "BA"},
    ]
    assert aggregate(verdicts) == ("reject", False)


def test_tie_tie_agrees() -> None:
    verdicts = [
        {"verdict": "tie", "position_order": "AB"},
        {"verdict": "tie", "position_order": "BA"},
    ]
    assert aggregate(verdicts) == ("tie", True)
```

`tests/test_aggregate.py` verifies that `export_dpo` drops ties,
rejects, and disputes:

```python
import pandas as pd

from code.export_dpo import export


def test_export_only_keeps_agreed_non_tie() -> None:
    df = pd.DataFrame([
        {"pair_id": "1", "prompt": "p",
          "response_a": "A", "response_b": "B",
          "final_verdict": "A_better",
          "position_swap_agreement": True,
          "labeller_set": "x,y", "difficulty": "easy"},
        {"pair_id": "2", "prompt": "p",
          "response_a": "A", "response_b": "B",
          "final_verdict": "disputed",
          "position_swap_agreement": False,
          "labeller_set": "x,y", "difficulty": "easy"},
        {"pair_id": "3", "prompt": "p",
          "response_a": "A", "response_b": "B",
          "final_verdict": "tie",
          "position_swap_agreement": True,
          "labeller_set": "x,y", "difficulty": "easy"},
    ])
    out = export(df)
    assert list(out["pair_id"]) == ["1"]
    assert out.iloc[0]["chosen"] == "A"
    assert out.iloc[0]["rejected"] == "B"
```

### 2.12 `Makefile`

```makefile
.PHONY: all test sample assign aggregate qa export clean

all: test sample assign aggregate qa export

test:
	pytest -q tests/

sample:
	python code/sample_prompts.py \
	  --pool data/prompt_pool.parquet \
	  --seed 20261117 --cap 20 \
	  --easy 200 --medium 400 --hard 200 \
	  --out outputs/sampled_prompts.parquet

assign:
	python code/assign_pairs.py \
	  --pairs outputs/sampled_prompts.parquet \
	  --labellers configs/labellers.yaml \
	  --seed 20261117 \
	  --out outputs/assignments.jsonl

aggregate:
	python code/aggregate.py \
	  --verdicts data/verdicts.parquet \
	  --out outputs/pairs.parquet

qa:
	python code/qa_metrics.py \
	  --verdicts data/verdicts.parquet \
	  --out outputs/pair_qa_report.json

export:
	python code/export_dpo.py \
	  --pairs outputs/pairs.parquet \
	  --out outputs/dpo_training.parquet

clean:
	rm -rf outputs/
```

### 2.13 Reading `pair_qa_report.yaml`

Three signals a reviewer reads before signing off:

1. **Per-labeller position bias.** Any labeller outside `[0.40,
   0.60]` on `picked_left_rate` is a calibration target. One
   labeller does not break the batch; a biased cohort does.
2. **Per-labeller length bias.** `picked_longer_rate` above
   0.70 means the labeller is using length as a proxy for
   quality. This is the single most insidious preference-bias
   failure.
3. **Position-swap disagreement rate.** `(n_disputed / n_total)`.
   A rate above ~20% means the pair pool is not labellable with
   this guideline; fix the guideline before shipping the batch.

## 3. Validation steps

1. **Schema validates a worked example.** `pytest -q tests/`
   exits 0.
2. **Position-swap aggregation is correct.**
   `tests/test_position_swap.py` covers the four aggregation
   cases: agreement, disagreement, any-reject, tie-tie.
3. **Export drops disputed, tie, and reject pairs.**
   `tests/test_aggregate.py` passes.
4. **Every pair is double-labelled.** `wc -l
   outputs/assignments.jsonl` equals twice the number of
   sampled prompts.
5. **Two distinct labellers per pair.** For every pair_id, the
   two `labeller_id`s differ. A match is a routing bug.
6. **Position-bias threshold enforced.** Any labeller flagged by
   `qa_metrics` has an action in the report (calibration
   scheduled, batch held, etc.).
7. **No generator identity in the labeling tool.** Grep the
   Label Studio config and Argilla settings for
   `generator`/`model_id`; they must not appear as visible
   fields. The generator identity is on the pair record, not
   on the task record shown to labellers.
8. **Tie rate reported, not hidden.** The QA report has
   `tie_rate` per labeller. A tie rate near zero on a non-
   trivial batch is a review-blocking flag.
9. **DPO export is signed.** `outputs/dpo_training.parquet`
   has a cosign signature, verifiable with the program's
   public key.
10. **`peer_review.status` defaults to `pending_review`.** The
    fine-tuning-engineer signs the hand-off separately.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero
zeros on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `PROGRAM.md` has §1 scope, §2 pool, §3 generation, §4 labelling, §5 bias controls, §6 hand-off, §7 signatures       | Yes       |
| 2  | Position-swap doubling: every (prompt, A, B) labelled twice in distinct orders by distinct labellers                | Yes       |
| 3  | `tie` and `reject` are first-class verdicts                                                                          | Yes       |
| 4  | Disputes are peer-reviewed; the program does not auto-resolve them                                                   | Yes       |
| 5  | Generator identity is NOT shown to the labeller                                                                     | Yes       |
| 6  | QA report publishes per-labeller position-bias, length-bias, tie, and reject rates                                  | Yes       |
| 7  | DPO export drops disputed, tie, reject, and position-swap-disagreement pairs                                        | Yes       |
| 8  | Hand-off format pinned (`dpo_training.parquet` with named fields) and signed                                         | Yes       |
| 9  | Schema has `additionalProperties: false`                                                                             | Yes       |
| 10 | IAA kappa target documented in PROGRAM.md and measured against Exercise 03 code                                     | Yes       |
| 11 | Length-bias threshold and position-bias threshold declared, not implied                                             | Yes       |
| 12 | Canary / calibration set referenced in §5(c) and has a known owner                                                  | No        |
| 13 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020 / T0018.000, NIST AI RMF MEASURE 2.3 / 2.11 / GOVERN 1.2) cited | No |
| 14 | `peer_review.status` defaults to `pending_review`                                                                    | Yes       |

## 5. Common mistakes

- **Shipping without the position swap.** A one-sided labelling
  of (prompt, A, B) measures the labeller's position
  preference, not response quality. Doubling is non-negotiable.
- **Resolving disputes with a tie-breaker vote.** A third
  labeller breaking a disagreement smears the position bias
  across three labellers instead of exposing it. Peer-review
  the disputed pair; if it can't be resolved cleanly, drop it.
- **Forcing labellers off `tie`.** Quotas or "no ties, please"
  guidance converts hard pairs into coin flips that train DPO
  on noise. `tie` is a first-class answer, always.
- **Showing generator identity to labellers.** Labellers who
  know which side is "the new model" and which is "the baseline"
  label the brand, not the content.
- **Reporting only aggregate bias.** Per-labeller is the unit
  of calibration; an aggregate average hides one biased
  labeller behind three balanced ones.
- **Treating length bias as a secondary concern.** It is the
  dominant failure mode of LLM-assisted and human preference
  labelling. Measure it in every batch.
- **Using the same labeller for both position orders of a pair.**
  Position bias is invisible. Routing must prevent this; the
  tests cover it.
- **Not stratifying the prompt pool.** A pool dominated by easy
  prompts trains DPO on easy wins and tells the policy nothing
  about hard cases. Stratify explicitly.
- **Omitting the `reject` path.** Pairs that are both wrong,
  both unsafe, or off-topic should be rejected, not labelled.
  Shipping them as tie pollutes training with "nothing to
  learn here" signal.
- **Signing the hand-off without `mod-108` exercise-04's
  calibration gate.** The pair corpus and the DPO run belong
  to different roles; the program owner cannot self-sign that
  the pairs train well.

## 6. References

- **Rafailov, R. et al. (2023). "Direct Preference Optimization:
  Your Language Model is Secretly a Reward Model." _NeurIPS
  2023_.** <https://arxiv.org/abs/2305.18290>. Primary source
  for DPO's data contract: pairs of `(prompt, chosen, rejected)`
  are the training unit.
- **Label Studio documentation — labeling configuration tags
  and ML backend:** <https://labelstud.io/tags/> and
  <https://labelstud.io/guide/>. Primary source for the pair-
  task interface.
- **Argilla documentation — Feedback datasets, questions,
  metadata:** <https://docs.argilla.io/>. Primary source for
  the `LabelQuestion` / `TextQuestion` / metadata-property
  primitives used in the pair dataset.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Governance anchor — biased preference labels poison the
  policy.
- **MITRE ATLAS — AML.T0020 Poison Training Data,
  AML.T0018.000 Manipulate AI Model (Modify AI Model):**
  <https://atlas.mitre.org/>. Adversary-facing anchors.
- **NIST AI Risk Management Framework — MEASURE 2.3,
  MEASURE 2.11, GOVERN 1.2:**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
  Governance anchors for the measurement surface,
  bias/fairness, and roles.
- **JSON Schema Draft 2020-12:**
  <https://json-schema.org/draft/2020-12>. The
  `preference_pair.schema.json` dialect.
- **Local anchors:** Exercise 01 (`task_record.schema.json`
  is the parent schema; `preference_pair` is a specialisation);
  Exercise 02 (vendor RFP's §3 quality commitments extend to
  this program if the labelling workforce is external);
  Exercise 03 (the IAA code that measures the §4(e) kappa
  target); Exercise 04 (active learning on the prompt pool is
  an alternative sampling strategy); `mod-108` exercise-04
  (consumes `dpo_training.parquet` and verifies the pair corpus
  via a calibration run); `mod-109` (signs the hand-off);
  `mod-111` (trends position-bias, length-bias, and tie rate
  across weekly reruns).
