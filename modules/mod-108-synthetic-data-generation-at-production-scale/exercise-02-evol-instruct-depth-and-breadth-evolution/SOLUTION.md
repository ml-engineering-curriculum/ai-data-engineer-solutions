# Exercise 02 — Evol-Instruct Depth and Breadth Evolution (Solution)

> **Exercise-prompt note.** The learning file for this exercise
> (`exercise-02-evol-instruct-depth-and-breadth-evolution.md`) is a
> scaffolded stub at authoring time. This solution is grounded in
> Chapter 03 (`03-evol-instruct-depth-and-breadth-evolution.md`) of
> the module, which is fully written, and in the exercise contract
> from `README.md`. Any future divergence from the exercise prompt
> should re-ground against that chapter.
> <!-- pending-verify: confirm the exercise prompt once it lands; the
> scaffolded stub does not yet pin round count or evolution budget. -->

## 1. Solution overview

This is an **implementation** exercise. The learner takes the
Self-Instruct corpus from Exercise 01 (or an equivalent SFT corpus)
and runs the Evol-Instruct operator set over it to produce a
difficulty-stratified corpus with explicit lineage. The chapter
covers five In-Depth operators (add constraints, deepening,
concretising, increased reasoning steps, complicate input) and one
In-Breadth operator, plus the Elimination Evolving filter that the
paper names as load-bearing.

Deliverables:

- A **seed corpus hash** pinned in the manifest (the Exercise 01
  SFT corpus or equivalent).
- A **per-round evolution log** that tags every evolved instruction
  with its operator, its seed lineage, its depth count, and its
  breadth count.
- A **four-round evolved corpus** (`outputs/evol_corpus.parquet`)
  with lineage columns intact.
- A **per-operator pass-rate report**
  (`filters/elimination_report.parquet`) that shows how many
  evolutions each operator produced and how many survived the
  elimination filter.
- A **round-level diversity trace** that shows task-family counts
  per round so topic narrowing is visible before it ships.

Governance framing. The Elimination Evolving filter is the control
that stops the pool from collapsing onto whatever shape the generator
enjoys evolving. OWASP Machine Learning Security Top 10 anchor:
**ML02:2023 Data Poisoning Attack** — operator-prompt leakage and
information-poor patterns both poison the training surface by
imitating instruction shape without carrying substantive difficulty.
MITRE ATLAS technique: **AML.T0020 Poison Training Data**. NIST AI
RMF anchors: **GOVERN 1.2** (role-and-responsibility accountability
for the pipeline), **MEASURE 2.11** (fairness-and-bias measurement —
topic-narrowing is a bias signal), and **MEASURE 4.2** (continuous
monitoring for performance drift — round-over-round task-family drift
is drift). Cite the exact framework URLs rather than paraphrasing.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── evol_instruct_config.yaml       # depth:breadth ratio, thresholds
├── code/
│   ├── operators.py                     # 5 in-depth + 1 in-breadth prompts
│   ├── evolve.py                        # per-round driver
│   ├── eliminate.py                     # the Elimination Evolving filter
│   ├── lineage.py                       # depth/breadth counting
│   └── manifest.py
├── inputs/
│   ├── seed_corpus.parquet              # from exercise-01 (or equivalent)
│   └── seed_corpus.parquet.sha256
├── outputs/
│   ├── evol_corpus.parquet              # final difficulty-stratified corpus
│   └── evol_corpus.parquet.sha256
├── filters/
│   ├── elimination_report.parquet       # per-(round, operator, reason) counts
│   ├── judge_scores.parquet             # judge-model scores per evolved item
│   └── round_diversity.parquet          # per-round task-family counts
└── diversity_trace.md                   # qualitative summary + plots
```

### 2.2 `code/operators.py` — operator prompts

```python
# code/operators.py
# Chapter 03 operator set. Each operator returns a prompt that
# instructs the generator to rewrite the seed instruction.
# The prompts are paraphrased from the WizardLM paper appendix; keep
# them short and keep the "must remain reasonable" constraint.
from __future__ import annotations

from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Operator:
    name: str
    family: str          # "in_depth" | "in_breadth"
    system: str
    user_template: str


IN_DEPTH = [
    Operator(
        name="add_constraints", family="in_depth",
        system="You are a Prompt Rewriter. Rewrite the given prompt into a "
                "more complex version by adding one additional constraint "
                "or requirement. The rewritten prompt must remain reasonable, "
                "must be answerable by a human, and must preserve the non-text "
                "parts of the original prompt verbatim.",
        user_template="Given prompt: {seed}\n\nRewritten prompt:",
    ),
    Operator(
        name="deepening", family="in_depth",
        system="You are a Prompt Rewriter. Rewrite the given prompt by making "
                "an existing requirement more specific or more depth-demanding. "
                "Do not introduce unrelated constraints.",
        user_template="Given prompt: {seed}\n\nRewritten prompt:",
    ),
    Operator(
        name="concretising", family="in_depth",
        system="You are a Prompt Rewriter. Rewrite the given prompt by "
                "replacing one general concept with a specific, named instance. "
                "Keep every other part of the prompt unchanged.",
        user_template="Given prompt: {seed}\n\nRewritten prompt:",
    ),
    Operator(
        name="increased_reasoning_steps", family="in_depth",
        system="You are a Prompt Rewriter. Rewrite the given prompt so that a "
                "correct answer requires more sequential reasoning steps. "
                "The rewritten prompt must remain reasonable and answerable.",
        user_template="Given prompt: {seed}\n\nRewritten prompt:",
    ),
    Operator(
        name="complicate_input", family="in_depth",
        system="You are a Prompt Rewriter. Thicken the input data in the given "
                "prompt without changing the requested output shape. Do not "
                "introduce ambiguity.",
        user_template="Given prompt: {seed}\n\nRewritten prompt:",
    ),
]

IN_BREADTH = [
    Operator(
        name="in_breadth", family="in_breadth",
        system="You are a Prompt Creator. Draw inspiration from the given "
                "prompt and create a brand-new prompt in the same domain. The "
                "new prompt should be rarer and should preserve the length "
                "and difficulty level of the given prompt.",
        user_template="Given prompt: {seed}\n\nNew prompt:",
    ),
]

ALL_OPERATORS = IN_DEPTH + IN_BREADTH


def pick_operator(rng, depth_weight: float = 0.5):
    """Weighted sample. depth_weight = fraction routed to in_depth family."""
    if rng.random() < depth_weight:
        return rng.choice(IN_DEPTH)
    return rng.choice(IN_BREADTH)
```

Design notes defensible at review:

- **The operator prompts preserve the "reasonable" constraint
  explicitly.** Chapter 03 §"The In-Depth evolving operators" names
  this as part of the paper's prompt shape — the generator can emit
  physically-impossible or self-contradictory rewrites if not
  constrained.
- **Depth:breadth is a configurable weight, not a hard-coded ratio.**
  The manifest pins the weight; a 50:50 default is the paper baseline.
- **Operator sampling is per-seed, not per-round.** Each surviving
  seed draws one operator per round. Round-level uniformity of
  operator *selection* (every operator sees roughly `N/6` seeds per
  round) is the diversity guarantee, not round-level uniformity of
  operator *output*.

### 2.3 `code/eliminate.py` — the Elimination Evolving filter

```python
# code/eliminate.py
# Chapter 03 §"The Elimination Evolving filter". Four named failure
# modes + modern judge-model step. Returns an EliminationResult with
# verdict, reason code, and (when applicable) judge score.
from __future__ import annotations

import re
from dataclasses import dataclass

from rouge_score import rouge_scorer

_SCORER = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=False)


SEED_SIMILARITY_THRESHOLD = 0.9    # drop near-identical rewrites
MAX_LENGTH_RATIO = 5.0             # evolved vs seed length ceiling

INFO_POOR_PATTERNS = [
    re.compile(r"^(sure|of course|happy to help|certainly)[,\.! ]", re.I),
    re.compile(r"^(here is|here's) (the|a|an) (rewritten|new) prompt", re.I),
    re.compile(r"^as an? (ai|language model|assistant)", re.I),
]

# Fragments of the operator system prompts; any verbatim echo is a
# leak. Keep this list in sync with operators.py.
OPERATOR_PROMPT_ECHOES = [
    "you are a prompt rewriter",
    "you are a prompt creator",
    "rewritten prompt:",
    "new prompt:",
]


@dataclass(slots=True)
class EliminationResult:
    kept: bool
    reason: str | None       # seed_similar | cannot_answer | info_poor |
                              # operator_leak | length_ratio | judge_low
    judge_score: float | None


def eliminate(seed: str, evolved: str, *, judge,
              judge_threshold: float = 0.6) -> EliminationResult:
    # 1. Near-identical rewrites.
    if _SCORER.score(seed, evolved)["rougeL"].fmeasure > SEED_SIMILARITY_THRESHOLD:
        return EliminationResult(False, "seed_similar", None)

    # 2. Cannot-answer self-check.
    if not judge.can_answer(evolved):
        return EliminationResult(False, "cannot_answer", None)

    # 3. Information-poor patterns.
    if any(p.search(evolved) for p in INFO_POOR_PATTERNS):
        return EliminationResult(False, "info_poor", None)

    # 4. Operator-prompt leakage.
    lowered = evolved.lower()
    if any(frag in lowered for frag in OPERATOR_PROMPT_ECHOES):
        return EliminationResult(False, "operator_leak", None)

    # 5. Length-ratio sanity.
    seed_toks = len(seed.split())
    evolved_toks = len(evolved.split())
    if seed_toks and evolved_toks / seed_toks > MAX_LENGTH_RATIO:
        return EliminationResult(False, "length_ratio", None)

    # 6. Judge-model harder-but-correct score.
    score = judge.score_harder_but_correct(seed=seed, evolved=evolved)
    if score < judge_threshold:
        return EliminationResult(False, "judge_low", score)

    return EliminationResult(True, None, score)
```

Design notes worth defending at review:

- **The cannot-answer check is a generator self-judge.** Chapter 03
  §"The Elimination Evolving filter" names this: the generator is
  asked whether it can produce a correct response, and admissions of
  inability drop the candidate.
- **The judge-model step is the modern-recipe addition.** A second
  independent model scores `(seed, evolved)` for "harder but
  correct"; low scores drop the candidate. This moves the filter
  from "I can answer" to a stricter check and reduces the depth
  illusion failure mode in §"Where Evol-Instruct breaks".
- **Operator-prompt echo list is kept in sync with `operators.py`.**
  Any rewording of the operator prompts must be reflected in the
  echo-leak list or the regression test fails.

### 2.4 `code/lineage.py` — lineage tracking

```python
# code/lineage.py
# Every evolved instruction carries a lineage pointer back to its
# seed and (recursively) through its evolution path. The lineage is
# a parquet column, not a sidecar — Chapter 03 §"Lineage loss under
# reshuffle" is explicit.
from __future__ import annotations

from dataclasses import dataclass, field


@dataclass(slots=True)
class LineageNode:
    id: str
    operator: str | None   # None for the original seed


@dataclass(slots=True)
class Lineage:
    chain: list[LineageNode] = field(default_factory=list)
    round: int = 0

    @property
    def depth(self) -> int:
        from operators import IN_DEPTH
        depth_ops = {op.name for op in IN_DEPTH}
        return sum(1 for n in self.chain if n.operator in depth_ops)

    @property
    def breadth(self) -> int:
        from operators import IN_BREADTH
        breadth_ops = {op.name for op in IN_BREADTH}
        return sum(1 for n in self.chain if n.operator in breadth_ops)

    def extend(self, new_id: str, operator: str) -> "Lineage":
        return Lineage(
            chain=self.chain + [LineageNode(id=new_id, operator=operator)],
            round=self.round + 1,
        )

    def to_pylist(self) -> list[dict]:
        return [{"id": n.id, "operator": n.operator} for n in self.chain]
```

The `depth` and `breadth` counts land as parquet columns next to
`lineage`; `fine-tuning-engineer` reads them to build a curriculum.

### 2.5 `code/evolve.py` — the per-round driver (skeleton)

```python
# code/evolve.py
# Per-round driver. Reads the surviving pool, picks an operator per
# item, invokes the generator, runs the Elimination Evolving filter,
# records lineage and judge score, and appends the survivors to the
# output parquet. The round-dedup step (ROUGE-L > 0.8 within the
# round) runs after per-item elimination.
from __future__ import annotations

import argparse
import random
from collections import Counter
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq

from eliminate import eliminate
from lineage import Lineage, LineageNode
from operators import pick_operator


def run_round(round_idx: int, seeds: list[dict], generator, judge, rng,
              depth_weight: float, round_dedup_rouge_l: float,
              elimination_counter: Counter) -> list[dict]:
    kept: list[dict] = []
    seen_texts: list[str] = []
    for seed in seeds:
        op = pick_operator(rng, depth_weight=depth_weight)
        evolved = generator.rewrite(
            system=op.system,
            user=op.user_template.format(seed=seed["instruction"]),
        )
        result = eliminate(seed["instruction"], evolved, judge=judge)
        elimination_counter[(round_idx, op.name, result.reason or "kept")] += 1
        if not result.kept:
            continue

        # Round-level dedup: ROUGE-L against evolved items already kept
        # in this round. Chapter 03 §"The Elimination Evolving filter"
        # describes this modern-recipe addition.
        if _max_rouge_l(evolved, seen_texts) > round_dedup_rouge_l:
            elimination_counter[(round_idx, op.name, "round_dup")] += 1
            continue
        seen_texts.append(evolved)

        new_id = f"ev-{round_idx:02d}-{len(kept):06d}"
        lineage = Lineage(
            chain=list(seed.get("lineage", [])) + [
                LineageNode(id=new_id, operator=op.name),
            ],
            round=round_idx,
        )
        kept.append({
            "instruction_id": new_id,
            "instruction": evolved,
            "parent_id": seed["instruction_id"],
            "operator": op.name,
            "round": round_idx,
            "depth": lineage.depth,
            "breadth": lineage.breadth,
            "lineage": lineage.to_pylist(),
            "judge_score": result.judge_score,
        })
    return kept


def _max_rouge_l(cand: str, pool: list[str]) -> float:
    from rouge_score import rouge_scorer
    s = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=False)
    return max((s.score(cand, p)["rougeL"].fmeasure for p in pool), default=0.0)
```

Design notes worth defending at review:

- **Lineage grows across rounds.** `round` and `depth` / `breadth`
  are separately tracked. A high-depth low-breadth lineage is a
  hard-but-narrow instruction; `fine-tuning-engineer` uses the two
  counts separately.
- **Round-level dedup runs on evolved items, not seed-vs-evolved.**
  Seed-similarity is already caught by the elimination filter.
  Round-level dedup is the modern-recipe addition that prevents
  cross-seed evolution duplicates from accumulating.
- **Operator identity is a parquet column.** The per-operator pass
  rate is read from the column, not a sidecar — reshuffle-safe.

### 2.6 `recipes/evol_instruct_config.yaml`

```yaml
# recipes/evol_instruct_config.yaml
seed_corpus:
  path: inputs/seed_corpus.parquet
  sha256: <64-hex-digest>
  item_count: 2000

generator:
  model_id: claude-3-5-sonnet-20241022
  temperature: 0.7
  max_tokens: 2048

operators:
  in_depth:
    weight: 0.5
    operators:
      - add_constraints
      - deepening
      - concretising
      - increased_reasoning_steps
      - complicate_input
  in_breadth:
    weight: 0.5
    operators:
      - in_breadth

rounds: 4

elimination:
  seed_similarity_threshold: 0.9
  cannot_answer_check: true
  info_poor_patterns_path: filters/info_poor_patterns.txt
  operator_prompt_leak_check: true
  judge_model: claude-3-5-sonnet-20241022
  judge_threshold: 0.6
  round_dedup_rouge_l: 0.8
  max_length_ratio: 5.0

lineage:
  track_depth: true
  track_breadth: true

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

## 3. Validation steps

1. **Seed-corpus hash matches.** `sha256sum
   inputs/seed_corpus.parquet` equals the pin in the manifest.
2. **Operator balance per round.** Each of the five In-Depth
   operators receives roughly `N / 5 × depth_weight` seeds per round
   (within a 2× tolerance of the mean). The In-Breadth operator
   receives roughly `N × (1 - depth_weight)`.
3. **Lineage referential integrity.** Every `parent_id` in
   `outputs/evol_corpus.parquet` resolves to an `instruction_id`
   in either the seed corpus or an earlier round's rows.
4. **Depth and breadth counts are internally consistent.** For
   every row, `depth + breadth == round`. Mismatches indicate a
   lineage-write bug.
5. **No operator-prompt leakage.** Grep the output corpus's
   `instruction` column for the `OPERATOR_PROMPT_ECHOES` fragments;
   zero matches.
6. **Elimination report shape.** `filters/elimination_report.parquet`
   has one row per `(round, operator, reason)` tuple.
7. **Round-level diversity.** `filters/round_diversity.parquet`
   shows per-round task-family counts. No family loses more than
   X% of its mass per round (pin `X` in the manifest; Chapter 03
   §"Topic narrowing" names X around 25% as a yellow flag).
8. **Length-ratio sanity.** For every row in the output corpus,
   `len(instruction) / len(parent.instruction) ≤ MAX_LENGTH_RATIO`.
9. **Judge-score distribution is non-degenerate.** The judge-score
   column has a visible spread; a distribution concentrated at a
   single score indicates a judge-fidelity failure (Chapter 06
   §"Known reference judges" applies here one level upstream).
10. **Report determinism.** Fixing the generator seed and the
    operator-selection RNG seed reproduces the same lineage graph
    modulo generator non-determinism (hosted models are not
    bitwise-deterministic; expect lineage-shape invariance, not
    byte-equality).

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 22/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Seed corpus hash is pinned in the manifest and verified at run start                                                | Yes       |
| 2  | All five In-Depth operators and the single In-Breadth operator are implemented with the Chapter 03 prompt shape      | Yes       |
| 3  | Depth:breadth weight is a manifest parameter; default 0.5:0.5 is paper-faithful                                     | Yes       |
| 4  | Elimination filter covers all four paper-named failure modes + the judge-model score                                | Yes       |
| 5  | Lineage is a parquet column (not a sidecar); depth and breadth counts are separate columns                          | Yes       |
| 6  | Round-level dedup (ROUGE-L > 0.8 within round) is applied                                                           | Yes       |
| 7  | Operator-prompt leak check matches on the fragment list defined in `operators.py`                                   | Yes       |
| 8  | Length-ratio ceiling is pinned at ≤ 5.0 (or justified in the manifest)                                               | No        |
| 9  | Judge model is independent of the evolution generator OR calibration is documented                                   | No        |
| 10 | Round-level task-family diversity is tracked and plotted                                                            | Yes       |
| 11 | Elimination report has per-(round, operator, reason) counts                                                         | Yes       |
| 12 | Validation step 3 (lineage referential integrity) passes on the final corpus                                        | Yes       |
| 13 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF GOVERN 1.2 / MEASURE 2.11 / MEASURE 4.2) cited | No   |
| 14 | Unverifiable claims marked `<!-- pending-verify: ... -->`                                                           | Yes       |
| 15 | `diversity_trace.md` names at least one topic that narrowed and one that held, with rejection counts                 | No        |

## 5. Common mistakes

- **Operator-prompt leakage silently accepted.** The generator will
  occasionally emit the operator prompt verbatim. The echo-leak
  check in `eliminate.py` catches it; forgetting to update the
  echo list when a prompt changes is the most common regression.
- **Lineage as a sidecar.** A JSON file next to the parquet drifts
  under reshuffle. Chapter 03 §"Lineage loss under reshuffle" is
  explicit — the lineage travels with the row.
- **Only measuring aggregate judge score.** A corpus-wide judge mean
  can look fine while one operator is quietly collapsing. Report
  judge-score distribution per operator, not just overall.
- **Depth without breadth.** A pool that runs only In-Depth evolution
  from a narrow seed produces a very hard corpus on a very narrow
  surface — the opposite of what you want. Keep the breadth operator
  in the rotation.
- **Letting the depth illusion ship.** Evolved instructions look
  harder but responses stay the same length and shape. Compare
  response-length distributions across rounds. If they do not move,
  the depth signal is not transferring — strengthen the judge filter.
- **Running without a round-dedup step.** Different seeds can evolve
  to the same text. The round-dedup catches the duplicates; without
  it the pool bloats with near-identical siblings.
- **Un-pinned judge model.** The judge is upstream of the training
  signal. A silent judge upgrade changes the filter behaviour; pin
  the dated version.
- **Pre-populating the counsel block in the licence ledger.** This
  exercise shares the `licence/` section with Exercises 03 and 05 —
  the counsel sign-off slot defaults to `TBD`.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchor for operator-prompt leakage and info-poor
  patterns entering the training surface.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The adversary-facing anchor for a
  generator that inflates difficulty surface without substance.
- **NIST AI Risk Management Framework — GOVERN 1.2, MEASURE 2.11,
  MEASURE 4.2:** <https://www.nist.gov/itl/ai-risk-management-framework>.
  GOVERN 1.2 anchors pipeline accountability; MEASURE 2.11 anchors
  bias measurement that topic-narrowing would undercut; MEASURE 4.2
  anchors continuous-monitoring for round-over-round drift.
- **WizardLM (Xu et al. 2023, ICLR 2024), arXiv 2304.12244:**
  <https://arxiv.org/abs/2304.12244>. The primary source for the
  In-Depth and In-Breadth operator set, the Elimination Evolving
  filter, and the paper-default 50:50 weight.
- **WizardCoder (Luo et al. 2023), arXiv 2306.08568:**
  <https://arxiv.org/abs/2306.08568>. Code-domain adaptation; the
  operator contract is unchanged.
- **WizardMath (Luo et al. 2023), arXiv 2308.09583:**
  <https://arxiv.org/abs/2308.09583>. Math-domain adaptation; adds
  the RLEIF (reward-filtered Evol-Instruct) step relevant to the
  judge-score filter in `eliminate.py`.
- **NeMo Curator:** <https://github.com/NVIDIA-NeMo/Curator>. Ships
  the `NemotronGenerator` operator family against which the
  Evol-Instruct operators can be built. Pin the library version in
  the manifest; the submodule layout changes across releases.
- **Data-Juicer operators:** <https://datajuicer.github.io/data-juicer/en/main/docs/Operators.html>.
  Ships `GenerateQAFromExamplesMapper` and `OptimizeQAMapper`-style
  refinement; mapper naming has evolved — check the release's
  operator list.
- **Local anchors:** `mod-108` Chapter 03 (operator set and
  Elimination filter); `mod-108` exercise-01 (seed corpus input);
  `mod-109` (lineage column signing in the final parquet);
  `mod-107` Chapter 08 (human preference-labelling that the judge
  step approximates).
