# Exercise 05 — Curse of Recursion Ablation Run (Solution)

> **Exercise-prompt note.** The learning file for this exercise
> (`exercise-05-curse-of-recursion-ablation-run.md`) is a scaffolded
> stub. This solution is grounded in Chapter 07
> (`07-curse-of-recursion-and-model-collapse-defence.md`), which is
> fully written, and in the exercise contract from `README.md` plus
> Chapter 07 §"How to run the ablation (Exercise 05)". Any future
> divergence from the exercise prompt should re-ground against that
> chapter.
> <!-- pending-verify: confirm the exercise prompt once it lands; the
> scaffolded stub does not yet pin base model, dataset, or
> generation count. -->

## 1. Solution overview

This is an **implementation + measurement** exercise. The learner
reproduces the Shumailov et al. 2024 collapse experiment at small
scale: train a tiny base model on real data, sample it to produce
synthetic data, train the next generation on that synthetic data,
repeat for N generations, and measure the three drift diagnostics
(tail perplexity, KL from gen-0, qualitative tail review) at each
generation. Then rerun with anchor ratios `α = 0.5` and `α = 0.9`
to show the anchoring defence works.

Deliverables:

- `recursion_defence/iteration_log.yaml` — one entry per generation,
  with anchor ratio, upstream identities, and the three diagnostics.
- `recursion_defence/kill_switch.yaml` — thresholds pinned in advance;
  the ablation is specifically designed to breach them.
- `outputs/gen_{k}_model/` — the trained checkpoint per generation.
- `outputs/gen_{k}_synthetic.parquet` — the synthetic data used for
  training gen `k+1`.
- `reports/ablation_report.md` — the plots (tail perplexity, KL,
  qualitative review verdicts) and the anchor-ratio sweep.
- `reports/kill_switch_breach_log.md` — one row per breach, with the
  generation, the diagnostic that breached, and the decision (halt
  / override with sign-off).

Governance framing. The ablation is the single most convincing
artifact to show a stakeholder arguing "we can loop the generator
forever". Chapter 07 §"How to run the ablation (Exercise 05)" calls
it out explicitly. The recursion-defence log is one of the seven
artifacts of the program package and is a formal release gate.

- OWASP Machine Learning Security Top 10 anchors: **ML02:2023 Data
  Poisoning Attack** (recursive training on generated data is a
  poisoning vector through omission of real-distribution tails);
  **ML05:2023 Model Theft** / adjacent (a collapsed model is a
  corrupted artifact in supply-chain terms).
- MITRE ATLAS techniques: **AML.T0020 Poison Training Data**
  (synthetic recursion is the poisoning mechanism).
- NIST AI Risk Management Framework controls: **MEASURE 2.3**
  (performance measured as intended — a collapsed model fails this
  on the tail); **MEASURE 4.2** (continuous monitoring); **MANAGE
  2.3** (AI risk tracked and documented over time); **GOVERN 1.2**
  (role and responsibility accountability — kill-switch override is
  a named role).

Cite the framework URLs rather than paraphrasing.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── recipes/
│   └── ablation_config.yaml
├── recursion_defence/
│   ├── iteration_log.yaml
│   └── kill_switch.yaml
├── data/
│   ├── real_corpus.parquet            # the gen-0 training data
│   ├── real_corpus.parquet.sha256
│   ├── held_out_tail.parquet          # tail-category eval split (firewalled)
│   └── held_out_tail.parquet.sha256
├── code/
│   ├── train.py                        # tiny GPT-2 / Pythia trainer
│   ├── sample.py                       # sampling harness → synthetic.parquet
│   ├── diagnostics.py                  # tail perplexity + KL + qual review
│   ├── anchor_mix.py                   # build gen-k training mix at ratio α
│   └── plot_ablation.py                # the 3-panel plot per α setting
├── outputs/
│   ├── gen_0_model/
│   ├── gen_0_synthetic.parquet
│   ├── gen_1_model/
│   ├── gen_1_synthetic.parquet
│   ├── ...
│   └── gen_5_synthetic.parquet
└── reports/
    ├── ablation_report.md
    └── kill_switch_breach_log.md
```

### 2.2 Model and data choice

Pin the model and dataset in the manifest. Chapter 07 §"How to run
the ablation (Exercise 05)" names a small, well-understood model and
a dataset with known tails. The reference setup for this exercise:

- **Base model:** Pythia-70M (`EleutherAI/pythia-70m`,
  <https://huggingface.co/EleutherAI/pythia-70m>). Small enough to
  finetune in a few minutes on a single GPU; the Pythia suite ships
  checkpoints so recursion is reproducible.
- **Dataset with known tails:** a subset of The Pile (Gao et al.
  2020) selected to preserve long-tail entity mentions, or a
  Wikipedia revision-history subset selected the same way. Either
  choice must document how the tail split was separated from the
  head. Pin both the subset's content hash and the split's hash.

### 2.3 `recursion_defence/kill_switch.yaml`

Pin the thresholds *before* the ablation runs. The ablation is
specifically designed to breach them at small α and to pass at large
α; the thresholds themselves are the release-gate shape, not a
research-tunable.

```yaml
# recursion_defence/kill_switch.yaml
# Chapter 07 §"Kill-switch thresholds". A breach is a release block;
# an override is a formally-logged action requiring sign-off.
thresholds:
  tail_perplexity_rise_pct_per_generation: 5.0
  kl_divergence_cumulative_max: 0.3
  qualitative_tail_review:
    sample_size: 50
    reviewer_signoff_required: true
    halt_on_review: true

action_on_breach:
  halt_generation: true
  notify: ai-data-engineer-owner@example.com
  rollback_target: previous_iteration

override:
  permitted: true
  required_signoffs:
    - ai-data-engineer-owner
    - ai-risk-engineer           # for release-scale slices
  override_record_path: reports/kill_switch_breach_log.md
```

### 2.4 `code/sample.py` — sampling harness

```python
# code/sample.py
# Sample from a trained checkpoint to produce synthetic data for
# training the next generation. Chapter 07 is explicit that the
# sampling distribution is what collapses; keep the sampling knobs
# constant across generations so the collapse signal is pure.
from __future__ import annotations

import argparse
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer


def sample(model_dir: Path, prompts: list[str], output: Path,
            temperature: float = 1.0, max_new_tokens: int = 256,
            seed: int = 42) -> None:
    torch.manual_seed(seed)
    tok = AutoTokenizer.from_pretrained(model_dir)
    model = AutoModelForCausalLM.from_pretrained(model_dir)
    model.eval()

    rows: list[dict] = []
    for i, prompt in enumerate(prompts):
        ids = tok(prompt, return_tensors="pt").input_ids
        out = model.generate(
            ids, do_sample=True, temperature=temperature,
            max_new_tokens=max_new_tokens, top_p=1.0,
            pad_token_id=tok.eos_token_id,
        )
        text = tok.decode(out[0], skip_special_tokens=True)
        rows.append({
            "synth_id": f"synth-{i:06d}",
            "prompt": prompt,
            "text": text,
            "source_model_dir": str(model_dir),
            "sample_seed": seed,
            "temperature": temperature,
        })
    pq.write_table(pa.Table.from_pylist(rows), output)
```

Design notes defensible at review:

- **Sampling is at the same temperature across generations.**
  Varying temperature across generations confounds the collapse
  signal with a sampling-strategy change. Pin the sampling knobs
  in the manifest; they do not move during the ablation.
- **Prompts for sampling are a fixed held-out set.** A generation
  that gets a different prompt distribution has a confounding
  difference. The prompt set is a parquet file with its own hash.
- **Seeds are per-generation but recorded.** Reproducibility
  matters; pin the seed in the manifest and log it on each row.

### 2.5 `code/diagnostics.py` — the three diagnostics

```python
# code/diagnostics.py
# Chapter 07 §"Lever 3 — round-over-round drift measurement".
# Three diagnostics, each tracked across iterations:
#   1. Perplexity on a held-out real-data tail.
#   2. KL divergence against the gen-0 reference distribution.
#   3. Qualitative tail-category review.
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path

import math
import pyarrow.parquet as pq
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer


@dataclass(slots=True)
class DiagnosticResult:
    tail_perplexity: float
    kl_from_gen_0: float
    qualitative_review: str   # "passed" | "flagged" | "failed"
    qualitative_sample_size: int


def tail_perplexity(model_dir: Path, tail_split: Path) -> float:
    """Mean per-token perplexity on the held-out tail split."""
    tok = AutoTokenizer.from_pretrained(model_dir)
    model = AutoModelForCausalLM.from_pretrained(model_dir)
    model.eval()
    total_nll = 0.0
    total_tokens = 0
    for row in pq.read_table(tail_split).to_pylist():
        ids = tok(row["text"], return_tensors="pt").input_ids
        with torch.no_grad():
            out = model(ids, labels=ids)
        n_tokens = ids.shape[1]
        total_nll += out.loss.item() * n_tokens
        total_tokens += n_tokens
    return math.exp(total_nll / total_tokens)


def kl_from_reference(model_dir: Path, gen_0_dir: Path,
                        reference_prompts: list[str]) -> float:
    """Mean per-token KL(gen_0 || gen_N) over a fixed prompt set."""
    tok = AutoTokenizer.from_pretrained(gen_0_dir)
    m0 = AutoModelForCausalLM.from_pretrained(gen_0_dir).eval()
    mN = AutoModelForCausalLM.from_pretrained(model_dir).eval()

    kls = []
    for prompt in reference_prompts:
        ids = tok(prompt, return_tensors="pt").input_ids
        with torch.no_grad():
            p = torch.log_softmax(m0(ids).logits, dim=-1)
            q = torch.log_softmax(mN(ids).logits, dim=-1)
        # KL(p || q) = sum p * (log p - log q)
        kls.append(torch.sum(p.exp() * (p - q), dim=-1).mean().item())
    return sum(kls) / len(kls)


def qualitative_tail_review(model_dir: Path, tail_prompts: list[str],
                              reviewer_fn) -> tuple[str, int]:
    """Human-reviewer-in-the-loop over a hand-authored long-tail prompt
    catalogue. reviewer_fn takes (prompt, response) → "pass"|"flag"|"fail".
    """
    tok = AutoTokenizer.from_pretrained(model_dir)
    model = AutoModelForCausalLM.from_pretrained(model_dir).eval()
    verdicts: list[str] = []
    for prompt in tail_prompts:
        ids = tok(prompt, return_tensors="pt").input_ids
        with torch.no_grad():
            out = model.generate(ids, do_sample=False, max_new_tokens=256,
                                   pad_token_id=tok.eos_token_id)
        response = tok.decode(out[0], skip_special_tokens=True)
        verdicts.append(reviewer_fn(prompt, response))
    if any(v == "fail" for v in verdicts):
        return "failed", len(verdicts)
    if any(v == "flag" for v in verdicts):
        return "flagged", len(verdicts)
    return "passed", len(verdicts)
```

Two design choices worth defending at review:

- **Tail perplexity is tracked separately from head perplexity.**
  Chapter 07 §"Diagnostic 1" is explicit: the *relative* rise of
  tail vs head perplexity is the signal. Tracking only aggregate
  perplexity hides the collapse until it is severe.
- **KL is `KL(gen_0 || gen_N)`, not the reverse.** The direction
  matters. `KL(gen_0 || gen_N)` is the right choice because it
  weights by the *reference* (real) distribution — mass that gen_0
  assigns to a token and gen_N does not is heavily penalised. The
  reverse direction weights by gen_N's distribution, which collapses
  along with the model.

### 2.6 `code/anchor_mix.py` — the anchor-ratio builder

```python
# code/anchor_mix.py
# Build a gen-k training mix at anchor ratio α. α tokens come from
# real_corpus; (1 - α) come from the previous generation's synthetic
# corpus. Chapter 07 §"Lever 1 — real-data anchoring" is the anchor.
from __future__ import annotations

import argparse
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq


def build_mix(real: Path, synthetic: Path, output: Path,
               alpha: float, total_tokens: int, tokenizer) -> dict:
    real_rows = _sample_by_token_budget(
        pq.read_table(real).to_pylist(),
        budget_tokens=int(total_tokens * alpha),
        tokenizer=tokenizer,
    )
    synth_rows = _sample_by_token_budget(
        pq.read_table(synthetic).to_pylist() if synthetic.exists() else [],
        budget_tokens=int(total_tokens * (1.0 - alpha)),
        tokenizer=tokenizer,
    )
    mix = real_rows + synth_rows
    pq.write_table(pa.Table.from_pylist(mix), output)
    return {
        "anchor_ratio_requested": alpha,
        "real_tokens": sum(len(tokenizer(r["text"]).input_ids) for r in real_rows),
        "synth_tokens": sum(len(tokenizer(r["text"]).input_ids) for r in synth_rows),
        "total_rows": len(mix),
    }


def _sample_by_token_budget(rows, budget_tokens, tokenizer):
    kept, used = [], 0
    for r in rows:
        if used >= budget_tokens:
            break
        n = len(tokenizer(r["text"]).input_ids)
        kept.append(r)
        used += n
    return kept
```

### 2.7 `recursion_defence/iteration_log.yaml`

```yaml
# recursion_defence/iteration_log.yaml
# One entry per generation per α setting. Shape follows Chapter 07
# §"The defence log".
pipeline: synthetic_program/exercise-05-ablation
alpha_setting: 0.0     # one of: 0.0, 0.5, 0.9
generations:
  - iteration: 0
    anchor_ratio: 1.0
    upstream_identity: [real_corpus/pile_subset_sha_xxxxxx]
    diagnostics:
      tail_perplexity: 32.4
      kl_from_gen_0: 0.0
      qualitative_review: passed
      qualitative_sample_size: 50
  - iteration: 1
    anchor_ratio: 0.0
    upstream_identity:
      - real_corpus/pile_subset_sha_xxxxxx    # not used; alpha=0
      - synth/gen_0_sha_yyyyy
    diagnostics:
      tail_perplexity: 35.1          # +8.3%, breaches 5% threshold
      kl_from_gen_0: 0.12
      qualitative_review: flagged
      qualitative_sample_size: 50
    kill_switch_breach:
      diagnostic: tail_perplexity_rise_pct
      value: 8.3
      threshold: 5.0
      action: halt_or_override
      override_record: reports/kill_switch_breach_log.md
budget:
  max_iteration: 5
  kill_switch_breached: true
```

Three per-α-setting logs ship: `iteration_log_alpha_0.0.yaml`,
`iteration_log_alpha_0.5.yaml`, `iteration_log_alpha_0.9.yaml`. The
α=0 log is expected to breach at every generation past 0; α=0.5
should breach eventually; α=0.9 should not breach within 5
generations. The *expected* failure is the point — if α=0 does not
breach, the ablation is misconfigured.

### 2.8 `recipes/ablation_config.yaml`

```yaml
# recipes/ablation_config.yaml
base_model: EleutherAI/pythia-70m
tokenizer: EleutherAI/pythia-70m

training:
  epochs: 1
  batch_size: 16
  learning_rate: 5.0e-5
  total_tokens_per_generation: 2_000_000
  seed: 42

sampling:
  prompts_path: data/sampling_prompts.parquet
  prompts_sha256: <64-hex-digest>
  temperature: 1.0
  top_p: 1.0
  max_new_tokens: 256
  samples_per_generation: 8000

real_corpus:
  path: data/real_corpus.parquet
  sha256: <64-hex-digest>

tail_eval:
  path: data/held_out_tail.parquet
  sha256: <64-hex-digest>
  firewall_note: >
    This split is explicitly excluded from every generation's
    training mix. See mod-106 ex-01 §"Firewall policy" for the
    firewall contract that enforces the exclusion.

qualitative_review:
  prompts_path: data/tail_prompts.jsonl
  sample_size: 50
  reviewer_signoff_required: true

anchor_ratios: [0.0, 0.5, 0.9]
generations: 5
```

### 2.9 `reports/ablation_report.md`

Required content:

1. **Setup summary.** Base model, dataset, token budget per
   generation, sampling knobs, random seed. All pinned from the
   manifest.
2. **Three-panel plot per α setting.** Tail perplexity vs generation,
   KL from gen_0 vs generation, qualitative verdict (pass / flag /
   fail per generation).
3. **Anchor-ratio sweep comparison.** One row per α: at which
   generation did the kill-switch first breach? what was the mean
   cumulative KL at generation N?
4. **Interpretation.** Chapter 07's failure mode is reproduced at
   α = 0; the anchoring defence is shown to work at α = 0.9; the
   intermediate α = 0.5 shows the trade-off.
5. **Stakeholder message.** The one-paragraph version of the
   anchoring argument for a non-ML stakeholder.

Any claim that cannot be verified at authoring time (e.g., a vendor
throughput number) is marked `<!-- pending-verify: ... -->` and
blocks auto-merge.

## 3. Validation steps

1. **Pythia / base-model identity is pinned.** `yq '.base_model'
   recipes/ablation_config.yaml` returns a specific HF repo.
2. **Real corpus and tail split hashes match the sidecars.**
3. **Tail split is firewalled.** A grep of each generation's
   training mix for text present in `data/held_out_tail.parquet`
   returns zero hits. Chapter 07 §"Diagnostic suite drift" is the
   failure-mode anchor.
4. **Sampling prompt set is fixed across generations.** The
   `sampling.prompts_sha256` in the manifest is byte-equal across
   all generations and all α settings.
5. **Diagnostics run on every generation.** Three entries per
   generation in the iteration log.
6. **Kill-switch breaches are logged.** Every breach has a
   corresponding row in `reports/kill_switch_breach_log.md` with
   the diagnostic value, the threshold, and the decision
   (halt / override with sign-off).
7. **Anchor-ratio sweep is complete.** Three logs ship
   (`alpha=0.0`, `0.5`, `0.9`), each with ≥ 5 generations.
8. **Expected failure at α=0.** The α=0 log breaches by generation
   1 or 2. A run where α=0 does not breach is a misconfigured
   ablation (likely cause: the sampling temperature is too low to
   produce meaningful drift).
9. **Expected pass at α=0.9.** The α=0.9 log stays within the
   kill-switch thresholds across 5 generations.
10. **`gen_0` diagnostics are the reference.** `kl_from_gen_0` is
    0.0 at iteration 0 by construction; `tail_perplexity` at
    iteration 0 is the baseline the later iterations are
    measured against.
11. **Rows in every synthetic.parquet record the source model
    directory.** Lineage is unbroken even if files are reorganised.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Base model (e.g., Pythia-70M) and dataset (with explicit tail split) are pinned in the manifest                      | Yes       |
| 2  | Tail-eval split is firewalled from every generation's training mix                                                   | Yes       |
| 3  | Sampling prompt set and random seed are fixed across generations and α settings                                      | Yes       |
| 4  | `recursion_defence/kill_switch.yaml` is pinned before the ablation runs                                              | Yes       |
| 5  | All three diagnostics run every generation (tail perplexity, KL from gen_0, qualitative review)                      | Yes       |
| 6  | `recursion_defence/iteration_log.yaml` exists per α setting with ≥ 5 generations                                      | Yes       |
| 7  | Kill-switch breaches are logged with diagnostic value, threshold, and decision                                       | Yes       |
| 8  | α=0 run breaches at least one threshold by generation 1 or 2                                                         | Yes       |
| 9  | α=0.9 run stays within kill-switch thresholds across 5 generations                                                   | Yes       |
| 10 | Each synthetic.parquet records `source_model_dir` for lineage                                                        | Yes       |
| 11 | `reports/ablation_report.md` includes a three-panel plot per α setting                                                | No        |
| 12 | Anchor-ratio sweep comparison summarises first-breach generation per α                                               | No        |
| 13 | KL is computed as `KL(gen_0 || gen_N)`, not the reverse direction                                                     | Yes       |
| 14 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF MEASURE 2.3 / MEASURE 4.2 / MANAGE 2.3 / GOVERN 1.2) cited | No |
| 15 | Unverifiable claims marked `<!-- pending-verify: ... -->`                                                           | Yes       |

## 5. Common mistakes

- **Tail split leaks into training.** A generation that memorises
  the tail split has flattering perplexity; the diagnostic lies.
  Chapter 07 §"Diagnostic suite drift" is the anchor; firewall the
  tail split like a benchmark.
- **Sampling knobs drift across generations.** A temperature change
  mid-ablation confounds the collapse signal. Pin temperature,
  top_p, and max_new_tokens in the manifest and keep them constant.
- **Measuring only aggregate perplexity.** A collapsed model can
  look fine on average until it's extreme; the *tail* perplexity
  rise is the signal.
- **Wrong KL direction.** `KL(gen_N || gen_0)` weights by the
  collapsed distribution; mass gen_0 covered that gen_N does not
  gets under-weighted. Use `KL(gen_0 || gen_N)`.
- **No qualitative review.** Perplexity and KL automate; both can
  stay below threshold while a specific tail category silently
  disappears. The hand-authored tail-prompt catalogue catches this.
- **Skipping the α sweep.** The α=0 single run proves collapse.
  The α=0.9 run proves the defence. Without both, the ablation
  does not justify the anchoring argument.
- **Kill-switch override without sign-off.** Chapter 07 §"Where the
  defence breaks" §4 is explicit — an override is a formally-logged
  action with sign-off from `ai-data-engineer-owner` and (for a
  release-scale slice) `ai-risk-engineer`. A silent flag flip is
  a rubric failure.
- **Treating gen_0 synthetic as "real" for gen_2's α computation.**
  Chapter 07 §"Lever 1 — What counts as real" is explicit — a model
  that shares ancestry with your pipeline does not count as real.
  gen_0 synthetic is synthetic for every later generation's α.
- **Using a too-low temperature.** At `T < 0.5` the sampler barely
  emits mass away from the mode; the drift signal shrinks and α=0
  may not breach. Pin `T = 1.0` for the ablation unless you've
  measured that a lower T still produces visible drift.

## 6. References

- **Shumailov et al. 2024, Nature 631:**
  <https://www.nature.com/articles/s41586-024-07566-y>. The primary
  reference for the collapse failure mode and the three-error-source
  model (learned-approximation, functional-approximation, statistical-
  sampling). Earlier preprint: *The Curse of Recursion: Training on
  Generated Data Makes Models Forget.* arXiv 2305.17493 (2023):
  <https://arxiv.org/abs/2305.17493>.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Governance anchor: synthetic recursion poisons the training surface
  through omission of real-distribution tails.
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. Adversary-facing anchor.
- **NIST AI Risk Management Framework — MEASURE 2.3, MEASURE 4.2,
  MANAGE 2.3, GOVERN 1.2:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  2.3 anchors the "performance evaluated as intended" surface that
  tail perplexity measures; MEASURE 4.2 anchors continuous monitoring;
  MANAGE 2.3 anchors the kill-switch and override log trail.
- **Pythia suite (Biderman et al. 2023), arXiv 2304.01373:**
  <https://arxiv.org/abs/2304.01373>. The reference model family for
  the ablation; ships many sizes at matching training budgets, so
  recursion is reproducible at small scale.
- **The Pile (Gao et al. 2020), arXiv 2101.00027:**
  <https://arxiv.org/abs/2101.00027>. The reference pretraining
  corpus; the Pile's tail-heavy slices (ArXiv, PubMed, StackExchange)
  are the exercise's tail-eval candidates.
- **Hugging Face Transformers / Datasets:**
  <https://huggingface.co/docs/transformers>. The infrastructure for
  training and sampling the small base model.
- **Local anchors:** `mod-108` Chapter 07 (the full defence and the
  ablation recipe); `mod-106` exercise-01 (the firewall convention
  used to exclude the tail-eval split from training mixes);
  `mod-109` (the recursion-defence log signed as part of the program
  package); `mod-111` (how the drift diagnostics feed the quality-
  regression monitor).
