# Exercise 06 — Preference Labelling Program for DPO Pairs (Solution)

## 1. Solution overview

This is a **mixed design + implementation** exercise and the capstone
for mod-107. The learner stands up a **preference-labelling program**
that produces the `(prompt, chosen, rejected)` triples consumed by
Direct Preference Optimisation (DPO). Unlike classical labeling
(mod-107 ex-01), the "ground truth" is an opinion; unlike weak
supervision (ex-05), the opinion must be elicited from humans and
can be gamed by both annotators and the model under test.

Deliverables:

- A **program charter** that defines what *preference* means for
  this program: the comparison type (pairwise, k-wise), the
  response policies being compared, the attributes the comparison
  turns on (helpfulness, factuality, safety, style), tie rules,
  and what happens when both responses are unsafe.
- A **prompt-sampling plan** that documents how prompts are chosen,
  how sensitive-content coverage is handled, and how
  self-reinforcement loops with the model being trained are
  prevented.
- A **pair-generation procedure** that produces response pairs with
  provenance (which model, which sampling parameters, which
  snapshot) and a de-biasing policy (response-order randomisation,
  response-length matching, model-identity blinding).
- An **Argilla preference project** configured from `settings.yaml`
  (reuses the ex-01 pattern), with mandatory per-pair *rationale*.
- A **quality-and-bias audit** that measures annotator-position
  bias, length bias, model-preference bias, and inter-annotator
  agreement on preferences (α for pairwise, with a defensible
  treatment of ties).
- An **export tool** that emits a DPO-ready JSONL
  (`prompt, chosen, rejected, rationale, meta`), filters
  low-confidence pairs, and signs the batch manifest.

Governance framing. Preference data shapes model behaviour at scale,
which makes it both a **safety-relevant data pipeline** (model
behaviour follows the preferences the pipeline concentrates) and a
**bias amplification surface**. The control anchor is **NIST AI RMF
MANAGE 4.1** (AI risks are prioritised and responded to) combined
with **MEASURE 2.11** (fairness-related metrics) because a preference
pipeline that systematically prefers longer, more fluent, or more
authoritative-sounding answers *becomes* that bias in the model.
On the adversary side, **MITRE ATLAS AML.T0020 Poison Training
Data** and **AML.T0043 Craft Adversarial Data** cover prompt-injection
and preference-flip attacks. **OWASP ML Top 10 ML02:2023 Data
Poisoning** is the control objective.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-06/
├── README.md
├── charter.md                      # program charter (definition of preference)
├── sampling_plan.md                # prompt-selection policy
├── pair_generation/
│   ├── policies.yaml               # model/sampling configs for A and B
│   ├── generate_pairs.py           # produces response pairs with provenance
│   └── blinding_rules.md
├── argilla/
│   ├── settings.yaml               # FeedbackDataset schema for pairwise preference
│   └── make_dataset.py
├── audit/
│   ├── position_bias.py            # left-side preference audit
│   ├── length_bias.py              # length-correlated preference audit
│   ├── model_bias.py               # per-source model preference audit
│   └── iaa_preference.py           # Krippendorff α adapted for pairwise with ties
├── export/
│   ├── build_dpo_jsonl.py          # (prompt, chosen, rejected) emission
│   └── sign_batch.py               # cosign wrapper
└── results/
    ├── pairs_raw.jsonl
    ├── annotations_tidy.parquet
    ├── bias_report.md
    ├── dpo_batch.jsonl
    └── dpo_batch_manifest.json(.sig)
```

### 2.2 `charter.md` — program charter

```markdown
# Preference program <program-id> — charter

**Decision date:** 2026-11-18
**Author:** <operator email>

## Comparison type
- **Pairwise** (A vs B) with required **rationale**. k-wise comparisons
  (k>2) are out of scope for this program; add them under a charter
  revision if the task justifies the extra annotator load.

## What the annotator is deciding
The annotator chooses between responses A and B according to a
**ranked attribute list**:

1. **Safety** — if one response is unsafe and the other is not, the
   safe response is preferred. This supersedes everything else.
2. **Factuality** — if one response is factually wrong and the other
   is factually correct (and both are safe), the correct response is
   preferred.
3. **Helpfulness** — assuming parity on (1) and (2), the more useful
   response is preferred.
4. **Style and tone** — only a tiebreaker when 1–3 are parity.

The attribute list is *lexicographic*: 2 does not break a 1-tie.

## Ties and "both bad"
- `tie`: the responses are indistinguishable on all four attributes
  and neither is unsafe. A `tie` is a legitimate verdict and must
  not be discouraged.
- `both_unsafe`: both responses are unsafe or both evade a legitimate
  request. These pairs are **excluded from the DPO training set**;
  they are routed to safety training instead (not in this program).

## Who the annotator is
- Program annotators are trained on the charter and the ex-01 taxonomy;
  gold-set accuracy ≥ 0.90 is the seating gate.
- Each pair receives **3 independent annotations**; adoption requires
  **≥ 2 out of 3** agreement after order-randomisation unmasking.

## Scope exclusions
- The program does not label responses as "best possible" — it only
  ranks the pair presented.
- The program does not elicit creative edits ("rewrite A to be
  better"); that's response editing, a separate pipeline.
- The program does not use the model under training as an annotator;
  LLM-as-judge experiments are tracked under mod-108 synthetic-data.
```

### 2.3 `sampling_plan.md`

```markdown
# Prompt-sampling plan

## Sources
- Human-authored prompts from the ex-01 program's `prompt_bank` table.
- Model-elicited prompts from a *frozen prior-model* snapshot; model
  identity recorded per prompt. **The model under training is never
  used to generate the prompts in its own preference batch.**

## Distribution controls
- **Topic diversity:** at least 20 topics present in each batch;
  topics are tagged from a closed topic taxonomy maintained in
  `topics.yaml`.
- **Sensitive-content coverage:** a documented fraction of prompts
  (≥ 10% of the batch) exercise safety-relevant categories (medical
  advice, legal advice, self-harm, violence). Annotators handling
  these are rotated and offered the ex-02 right-to-refuse.
- **Language and dialect coverage:** at least the languages listed
  in ex-02 §4.5 are represented.

## Anti-loop controls
- Any prompt that was used in the last two batches is excluded
  from the current batch (prompt freshness window).
- A prompt whose preferred answer was adopted into the most recent
  DPO training run is excluded until the model's next evaluation
  checkpoint confirms behaviour has stabilised.

## Provenance
- Each prompt records: source, author (or model+revision), date
  authored, licence, and whether it has appeared in prior batches
  (`batch_history`).
```

### 2.4 `pair_generation/policies.yaml`

```yaml
# Response A and B are produced under different *policies*, where a
# policy is (model_id, model_revision, sampling_params, system_prompt).
# Comparing two samples of the same policy has limited pedagogical
# value; comparing two policies produces the gradient DPO needs.

policies:
  base_sft:
    model_id: internal/program-sft-v3
    model_revision: 7a41c90
    temperature: 0.7
    top_p: 0.95
    max_new_tokens: 1024
    system_prompt_sha256: 1c9e...

  rlhf_candidate:
    model_id: internal/program-rlhf-v4
    model_revision: 0b2f117
    temperature: 0.7
    top_p: 0.95
    max_new_tokens: 1024
    system_prompt_sha256: 1c9e...

matchups:
  # Each matchup is a labeled contest; the batch is a stratified sample
  # across matchups. Letting a single matchup dominate turns the batch
  # into a within-policy pair (see charter §scope).
  - name: candidate_vs_sft
    a: base_sft
    b: rlhf_candidate
    fraction: 0.6
  - name: candidate_vs_candidate_resample
    a: rlhf_candidate
    b: rlhf_candidate       # two independent samples
    fraction: 0.2
  - name: sft_vs_external_control
    a: base_sft
    b: external_frozen_v1
    fraction: 0.2
```

### 2.5 `pair_generation/blinding_rules.md`

```markdown
# Blinding and de-biasing rules

## Order randomisation
- For every pair, generate a fresh random bit `swap` ∈ {0, 1}.
- If `swap == 1`, present B first and A second. Store `swap` in the
  record; the audit tool unswaps before analysis.
- Annotators see no indication of which policy produced which side.

## Length matching
- Compute the token lengths of A and B; if the longer response is
  more than 2× the shorter, the pair is **re-sampled** before it
  reaches the UI. Length-matching rules out the easiest form of
  spurious preference.

## Model-identity blinding
- Response headers, signatures, model-identifying boilerplate
  ("As a large language model...") are stripped before the pair
  is served. The annotator must not be able to identify the model
  from the response surface.

## Metadata separation
- Policy ids, sampling params, and provenance are stored in
  Argilla metadata (hidden from the annotator). Only the prompt and
  the two responses are shown in the task UI.
```

### 2.6 `pair_generation/generate_pairs.py`

```python
# generate_pairs.py
from __future__ import annotations

import argparse
import hashlib
import json
import random
from dataclasses import dataclass, asdict
from pathlib import Path

import yaml


@dataclass
class Record:
    pair_id: str
    prompt_id: str
    prompt: str
    response_left: str     # what the annotator sees on the left
    response_right: str
    swap: int              # 1 if "right" is actually policy A
    policy_a: str
    policy_b: str
    matchup: str
    meta_sha256: str       # hash of the full provenance record


def generate(prompts_path: Path, policies_path: Path, out_path: Path,
             seed: int) -> None:
    rng = random.Random(seed)
    cfg = yaml.safe_load(policies_path.read_text())
    prompts = [json.loads(line) for line in prompts_path.read_text().splitlines()]
    records: list[Record] = []
    for p in prompts:
        matchup = _pick_matchup(cfg["matchups"], rng)
        resp_a = _sample_from_policy(cfg["policies"][matchup["a"]], p)
        resp_b = _sample_from_policy(cfg["policies"][matchup["b"]], p)
        if not _length_ok(resp_a, resp_b):
            # Re-sample up to 3 times; else drop with provenance note.
            continue
        swap = rng.randint(0, 1)
        left, right = (resp_a, resp_b) if swap == 0 else (resp_b, resp_a)
        prov = {
            "pair_id": f"{p['prompt_id']}-{matchup['name']}",
            "matchup": matchup["name"],
            "policy_a": matchup["a"],
            "policy_b": matchup["b"],
            "sampling_a": cfg["policies"][matchup["a"]],
            "sampling_b": cfg["policies"][matchup["b"]],
            "prompt_id": p["prompt_id"],
            "swap": swap,
        }
        records.append(Record(
            pair_id=prov["pair_id"],
            prompt_id=p["prompt_id"], prompt=p["prompt"],
            response_left=left, response_right=right,
            swap=swap, policy_a=matchup["a"], policy_b=matchup["b"],
            matchup=matchup["name"],
            meta_sha256=hashlib.sha256(
                json.dumps(prov, sort_keys=True).encode()).hexdigest(),
        ))
    out_path.write_text("\n".join(json.dumps(asdict(r)) for r in records))


def _pick_matchup(matchups, rng):
    r = rng.random()
    acc = 0
    for m in matchups:
        acc += m["fraction"]
        if r <= acc:
            return m
    return matchups[-1]


def _sample_from_policy(cfg, prompt_record):
    # Call out to the model service; omitted for brevity.
    ...


def _length_ok(a, b):
    la, lb = len(a.split()), len(b.split())
    lo, hi = sorted((la, lb))
    return hi <= 2 * max(lo, 1)


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--prompts", required=True, type=Path)
    ap.add_argument("--policies", required=True, type=Path)
    ap.add_argument("--out", required=True, type=Path)
    ap.add_argument("--seed", type=int, required=True)
    args = ap.parse_args()
    generate(args.prompts, args.policies, args.out, args.seed)
```

### 2.7 `argilla/settings.yaml`

```yaml
name: dpo-preference-batch-001
workspace: program-xyz

fields:
  - name: prompt
    required: true
  - name: response_left
    required: true
  - name: response_right
    required: true

questions:
  - name: preference
    type: label_selection
    labels: ["Left is better", "Right is better", "Tie", "Both bad / unsafe"]
    required: true
    description: Apply the lexicographic attribute list in charter.md.
  - name: primary_attribute
    type: label_selection
    labels: ["safety", "factuality", "helpfulness", "style", "n/a (tie)"]
    required: true
    description: Which attribute broke the choice?
  - name: rationale
    type: text
    required: true
    description: One sentence — why.

metadata_properties:
  - name: pair_id
    type: terms
  - name: matchup
    type: terms
    values: [candidate_vs_sft, candidate_vs_candidate_resample,
             sft_vs_external_control]
  - name: swap
    type: integer
    min: 0
    max: 1

allow_extra_metadata: false
```

The `preference` question is **left/right**, not "A/B". Annotators
never see policy ids. Unswapping happens in the audit tool.

### 2.8 `audit/position_bias.py`

```python
# position_bias.py
# Measures whether annotators prefer "left" more often than chance after
# unswapping. A strong left-position preference inflates DPO reward for
# whichever policy randomly landed on the left in each batch.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pandas as pd
from scipy.stats import binomtest


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--annotations", required=True, type=Path)
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()

    df = pd.read_parquet(args.annotations)
    df = df[df["preference"].isin(["Left is better", "Right is better"])]
    n = len(df)
    left_wins = (df["preference"] == "Left is better").sum()
    p = binomtest(k=int(left_wins), n=n, p=0.5,
                  alternative="two-sided").pvalue
    report = {
        "n_decisive": n,
        "left_win_rate": float(left_wins / n) if n else None,
        "p_value_vs_fair_coin": float(p),
        "verdict": "flag" if (n > 50 and p < 0.01) else "ok",
    }
    args.out.write_text(json.dumps(report, indent=2))


if __name__ == "__main__":
    main()
```

### 2.9 `audit/length_bias.py` — sketch

```python
# length_bias.py — point-biserial correlation between (right_is_chosen)
# and (len(right) - len(left)). A strong correlation is a finding.
#
# Report: Pearson r, 95% CI via Fisher z, and verdict = flag if |r| > 0.2
# with n > 100. Length matching in generation should keep |r| ≤ 0.1 in
# practice; a larger value means the matching rule is slipping.
```

### 2.10 `audit/model_bias.py` — sketch

```python
# model_bias.py — per-matchup win rates after unswapping.
# A win rate for `candidate_vs_candidate_resample` that is not centred
# on 0.5 is a red flag (two samples of the same policy should not be
# distinguishable in aggregate). A win rate for `candidate_vs_sft` that
# is ~1.0 means the comparison is uninformative and DPO gets no gradient
# — the program should widen the matchup pool.
```

### 2.11 `audit/iaa_preference.py`

```python
# iaa_preference.py — Krippendorff's α on the preference question with
# nominal distance, but with a chosen policy for ties:
#
#   - "Tie" vs "Left" and "Tie" vs "Right": δ = 0.5 (partial agreement)
#   - "Left" vs "Right": δ = 1 (full disagreement)
#   - "Both bad / unsafe" vs anything else: δ = 1
#
# The partial-tie distance is a design choice motivated in charter.md:
# an annotator who sees a tie and another who sees a marginal
# preference are not as discordant as two annotators with opposite
# preferences.
```

The partial-tie distance is a design decision and must be documented;
nominal α would overstate disagreement for ties and ordinal α is
ill-defined for a non-ordinal space ("left-better" and "right-better"
are not on a scale).

### 2.12 `export/build_dpo_jsonl.py`

```python
# build_dpo_jsonl.py
# Emits the DPO-ready training file: one JSONL line per adopted pair
# with fields {prompt, chosen, rejected, rationale, meta}. Pairs with
# `tie` or `both_bad` are excluded by construction.
from __future__ import annotations

import argparse
import json
from pathlib import Path

import pandas as pd


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--pairs", required=True, type=Path,       # pairs_raw.jsonl
                    help="output of pair_generation/generate_pairs.py")
    ap.add_argument("--annotations", required=True, type=Path)  # tidy parquet
    ap.add_argument("--adoption-min-agree", type=int, default=2,
                    help="Minimum consensus (of 3 annotators) for adoption.")
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()

    pairs = {json.loads(line)["pair_id"]: json.loads(line)
             for line in args.pairs.read_text().splitlines()}
    ann = pd.read_parquet(args.annotations)

    # Collapse 3 annotators → verdict; drop tie / both_bad.
    verdicts = _collapse(ann, min_agree=args.adoption_min_agree)
    out_lines: list[str] = []
    for pair_id, row in verdicts.items():
        if row["verdict"] in ("Tie", "Both bad / unsafe", None):
            continue
        p = pairs[pair_id]
        # Unswap: the annotator saw left/right; map back to policy_a / policy_b.
        left_wins = row["verdict"] == "Left is better"
        if p["swap"] == 0:
            chosen = p["response_left"] if left_wins else p["response_right"]
            rejected = p["response_right"] if left_wins else p["response_left"]
        else:
            # swap == 1 means the on-screen "left" was policy B
            chosen = p["response_left"] if left_wins else p["response_right"]
            rejected = p["response_right"] if left_wins else p["response_left"]
        out_lines.append(json.dumps({
            "prompt": p["prompt"],
            "chosen": chosen,
            "rejected": rejected,
            "rationale": row["rationale"],
            "meta": {
                "pair_id": pair_id, "matchup": p["matchup"],
                "primary_attribute": row["primary_attribute"],
                "n_annotators_agreed": int(row["n_agree"]),
            },
        }, ensure_ascii=False))
    args.out.write_text("\n".join(out_lines))


def _collapse(ann, min_agree):
    """Group by pair_id → majority verdict over the three annotators."""
    ...


if __name__ == "__main__":
    main()
```

The `_collapse` function implements the ex-01 "2-of-3 consensus"
rule; disagreement is routed to adjudication (not shown; the review
mechanism is the one from ex-01).

### 2.13 Batch manifest signing

`export/sign_batch.py` writes a `dpo_batch_manifest.json` with
`{batch_id, pair_count, matchup_counts, verdict_counts, hashes,
operator, run_window, bias_report_sha256}` and signs it with
cosign. The verdict counts include `chosen/rejected/tie/both_bad`;
the manifest is what mod-109 ingests.

## 3. Validation steps

1. **Charter predates generation.** `git log --follow charter.md |
   tail -1` is older than any `pair_generation/*.py` or
   `argilla/*.yaml`.
2. **Pairs are unswappable.** For each pair, `swap` is recorded and
   the build-DPO tool reproduces the policy assignment from `swap`
   deterministically. Flip `swap` in a fixture and confirm the
   chosen/rejected assignment flips.
3. **Length matching holds.** In `pairs_raw.jsonl`, no pair has
   `max(len(left), len(right)) > 2 * max(min(len(left), len(right)), 1)`.
4. **Position-bias audit below threshold.** `position_bias.py`
   returns `verdict: ok`. If `flag`, open a remediation item;
   the batch does not ship.
5. **Length-bias audit below threshold.** `length_bias.py`
   returns `|r| ≤ 0.1`; otherwise flag.
6. **Model-bias audit on `candidate_vs_candidate_resample` is
   centred.** 95% CI on the win rate includes 0.5; if it doesn't,
   something is wrong with the sampling policy.
7. **IAA α on preference ≥ 0.6.** Lower α on preference than on
   classification is expected; the threshold is chosen in the
   charter and documented.
8. **Rationale is mandatory and recorded.** Every adopted DPO row
   has a `rationale` string; empty rationales trigger a DPO-row
   drop and are logged.
9. **Ties and both-bad never enter `dpo_batch.jsonl`.** Grep for
   their `pair_id`s; they are only in the manifest's verdict
   counts.
10. **Batch manifest signs and verifies.** `cosign sign-blob`
    produces a signature; `cosign verify-blob` exits 0; the
    manifest's `bias_report_sha256` matches the audit output.
11. **No model-identity leakage.** `grep -iE 'as an ai|as a large
    language model|I am claude|gpt' results/pairs_raw.jsonl`
    returns no hits.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 28/36 with zero zeros on
critical rows.

| #  | Dimension                                                                                                     | Critical? |
|----|---------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Charter defines preference as a lexicographic attribute list (safety > factuality > helpfulness > style)       | Yes       |
| 2  | Tie and "both bad / unsafe" are first-class verdicts, not shortcuts                                            | Yes       |
| 3  | Prompts come from a documented source with batch-history / freshness controls                                  | Yes       |
| 4  | The model under training does not generate its own batch's prompts                                             | Yes       |
| 5  | Matchup design produces real cross-policy pairs; not two samples from a single policy by default                | Yes       |
| 6  | Response order is randomised per pair; `swap` is recorded and reversible                                       | Yes       |
| 7  | Length matching prevents > 2× length ratio; checked at generation time                                         | Yes       |
| 8  | Model identity is scrubbed from responses before the UI                                                        | Yes       |
| 9  | Argilla project enforces 3-annotator overlap with mandatory rationale and primary-attribute field              | Yes       |
| 10 | Position bias, length bias, and model bias are audited with concrete thresholds                                | Yes       |
| 11 | Preference IAA (α with the partial-tie distance) is computed and compared to a charter threshold               | Yes       |
| 12 | Adoption rule is 2-of-3 consensus; disagreement is adjudicated                                                 | Yes       |
| 13 | DPO export excludes tie / both-bad rows and preserves rationale                                                | Yes       |
| 14 | Batch manifest signed with cosign; audit report hash embedded                                                  | Yes       |
| 15 | Governance anchors (NIST AI RMF MANAGE 4.1 / MEASURE 2.11, OWASP ML02, MITRE ATLAS AML.T0020 / T0043) cited      | No        |
| 16 | Sampling plan documents sensitive-content coverage and annotator-rotation / right-to-refuse                    | Yes       |
| 17 | Prompts have provenance; licence and author captured                                                           | Yes       |
| 18 | Loop-prevention rule for prompts used in recent DPO runs is enforced and observable                             | Yes       |

## 5. Common mistakes

- **Treating preference as a single number.** "Helpfulness" is not
  a rank; the lexicographic attribute list is the only way to make
  "safety > factuality" a contract annotators can apply. Without it,
  annotators resolve conflicts privately and the dataset encodes
  their private rankings.
- **No response-order randomisation.** Left-position bias is a
  measured human effect; without `swap`, whichever policy landed on
  the left wins and DPO ingests that bias.
- **Letting the model under training generate the batch's prompts.**
  This is a direct self-reinforcement loop: the model's weaknesses
  are elided from the prompt distribution.
- **Dropping rationales to "save annotator time".** Rationale is the
  only diagnostic when the DPO run's behaviour is weird three weeks
  later. Mandatory rationale is cheap in annotation time and
  expensive in audit time if omitted.
- **Allowing > 2× length differences.** Length is the strongest
  single confound in preference data; a few 2000-vs-50-token pairs
  teach the model "longer is better" at scale.
- **Treating `tie` as a shortcut to skip hard pairs.** Add a
  `primary_attribute: n/a (tie)` field to make the tie a real
  judgement.
- **Not scrubbing model-identity phrases.** "As a large language
  model" is a model fingerprint; blinded preference data is
  compromised the moment annotators can spot the source.
- **Shipping without a model-bias audit.**
  `candidate_vs_candidate_resample` not being centred on 0.5 is
  the canary for sampling-policy bugs. Skipping this audit means
  bugs ship as training signal.
- **Pooling annotators across sensitive content without rotation.**
  Annotator fatigue + exposure to harmful content is both an ethical
  issue and a data-quality one; a tired annotator is a biased one.
- **Signing the manifest without the audit report hash.** The
  manifest and the bias report must be a single unit; downstream
  must be able to deny a batch whose bias report has changed.

## 6. References

- **NIST AI Risk Management Framework — MANAGE 4.1, MEASURE 2.11:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MANAGE
  4.1 anchors prioritised risk response (preference-data biases
  become model biases at scale); MEASURE 2.11 anchors the
  fairness-metric audit requirement.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Preference data is a direct behaviour-shaping surface; a
  compromised annotator cohort is a bulk-poisoning path.
- **MITRE ATLAS — AML.T0020 Poison Training Data; AML.T0043 Craft
  Adversarial Data:** <https://atlas.mitre.org/>. Preference
  flipping and prompt injection are the adversary-facing techniques
  the blinding and audits are meant to deny.
- **Rafailov, R., Sharma, A., Mitchell, E., Ermon, S., Manning,
  C. D., & Finn, C. (2023) "Direct Preference Optimization: Your
  Language Model is Secretly a Reward Model."** NeurIPS 2023. The
  method consuming this program's outputs; documents the
  `(prompt, chosen, rejected)` triple contract.
- **Ouyang, L., Wu, J., Jiang, X., et al. (2022) "Training language
  models to follow instructions with human feedback."** InstructGPT
  paper. The reference for pairwise preference elicitation
  practice, including rater instruction design and the role of
  reviewer adjudication.
- **Bai, Y., Jones, A., Ndousse, K., et al. (2022) "Training a
  Helpful and Harmless Assistant with Reinforcement Learning from
  Human Feedback."** Anthropic. Reference for safety-factuality-
  helpfulness attribute lists in preference pipelines.
- **Perez, E. et al. (2022) "Discovering Language Model Behaviors
  with Model-Written Evaluations."** Illustrates the sycophancy /
  self-reinforcement failure modes the anti-loop controls in the
  sampling plan guard against.
- **Krippendorff, K. (2011) "Computing Krippendorff's Alpha-
  Reliability."** Reused from ex-03; the partial-tie distance
  specialisation lives in §2.11.
- **Argilla documentation — FeedbackDataset for preference:**
  <https://docs.argilla.io/>. The reference for the pairwise
  schema in §2.7.
- **Local anchors:** `mod-107` ex-01 (program policy, consensus
  rules, canary pattern reused here); `mod-107` ex-02 (vendor
  audit, right-to-refuse, interview channel); `mod-107` ex-03 (α
  computation extended for the partial-tie distance); `mod-107`
  ex-05 (do not use the model under training as an annotator —
  LLM-as-judge is synthetic-data territory, mod-108); `mod-108`
  (synthetic-data pipeline that may extend preference pairs);
  `mod-109` (lineage + manifest signing); `mod-110` (PII handling
  in prompts and responses).
