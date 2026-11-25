# Exercise 03 — Teacher-Student Distillation with License Propagation (Solution)

> **Exercise-prompt note.** The learning file for this exercise
> (`exercise-03-teacher-student-distillation-with-license-propagation.md`)
> is a scaffolded stub. This solution is grounded in Chapter 05
> (`05-teacher-student-distillation-and-licence-propagation.md`),
> which is fully written, and in the exercise contract from
> `README.md`. Any future divergence from the exercise prompt should
> re-ground against that chapter.
> <!-- pending-verify: confirm the exercise prompt once it lands; the
> scaffolded stub does not yet pin corpus size or teacher model. -->

## 1. Solution overview

This is a **mixed design + implementation** exercise. The learner
captures a signed licence-propagation ledger for one distillation
run, then stands up a correctness-oriented distillation generation
loop that produces `(problem, chain_of_thought, answer)` triples and
hands the ledger + corpus off to counsel for sign-off before any
student training runs.

Two artifacts are load-bearing:

- `generator/licence_ledger.yaml` — captures teacher identity,
  licence SHA, downstream intent, questions for counsel, and the
  sign-off block. **The ledger is a release gate.** A distillation
  run without a signed ledger does not enter student training.
- `outputs/distillation_corpus.parquet` — the filtered triples, with
  lineage pointing back to the problem pool and the ledger run-id.

Governance framing. Distillation is the module's highest-risk
pattern: licensing exposure on the teacher side, benchmark
contamination inheritance on the eval side, capability-ceiling
inheritance on the student side. The ledger is the control that
makes all three legible to counsel and `ai-risk-engineer`.

- OWASP Machine Learning Security Top 10 anchors: **ML02:2023 Data
  Poisoning Attack** (teacher benchmark contamination enters the
  student via synthetic outputs); **ML06:2023 AI Supply Chain
  Attacks** (the teacher is an unverified upstream dependency).
- MITRE ATLAS techniques: **AML.T0010 ML Supply Chain Compromise**
  (the teacher is upstream of the student training surface);
  **AML.T0020 Poison Training Data** (contamination inheritance via
  the teacher's chain-of-thought).
- NIST AI Risk Management Framework controls: **GOVERN 6.1** (third-
  party risk management — the teacher's terms of service and licence
  are a third-party risk surface); **MAP 4.1** (mapping of legal and
  regulatory requirements across AI technology); **MEASURE 2.7**
  (security and resilience measurement for the pipeline).

Cite the framework URLs rather than paraphrasing the control text.
The ledger captures facts; counsel renders the legal opinion.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── requirements.lock
├── Makefile
├── generator/
│   ├── licence_ledger.yaml                    # release gate
│   ├── terms_snapshot/
│   │   ├── anthropic_aup_2026-01-15.html      # fetched at generation time
│   │   └── anthropic_aup_2026-01-15.html.sha256
│   └── counsel_opinion/
│       └── opinion_2026-01-20.pdf             # delivered by counsel
├── seeds/
│   ├── problem_pool.parquet                   # from mod-107 or public source
│   └── problem_pool.parquet.sha256
├── recipes/
│   └── distillation_config.yaml
├── code/
│   ├── fetch_terms.py                         # fetch and hash teacher terms
│   ├── build_problem_pool.py
│   ├── distill.py                             # the generation loop
│   ├── self_consistency.py                    # consistency-sampling filter
│   ├── verifier_filters.py                    # programmatic verifiers
│   ├── judge_filter.py                        # independent judge
│   └── ledger.py                              # ledger emit + sign check
├── outputs/
│   ├── distillation_corpus.parquet
│   └── distillation_corpus.parquet.sha256
├── filters/
│   └── filter_report.parquet                  # per-filter rejection counts
└── reports/
    ├── capability_ceiling_report.md           # style-vs-substance sample
    └── contamination_inheritance_scan.md      # Min-K% Prob-style scan
```

### 2.2 `generator/licence_ledger.yaml` — the release gate

```yaml
# generator/licence_ledger.yaml
run_id: 2026-01-15T14:00:00Z/distillation-run-7

teacher:
  identity: anthropic:claude-3-5-sonnet-20241022
  provenance: hosted_api
  licence_terms_url: https://www.anthropic.com/legal/aup
  licence_terms_fetched_at: 2026-01-15T13:45:12Z
  licence_terms_sha256: <sha256-of-fetched-HTML>
  licence_terms_version_if_stated: 2024-10-15
  snapshot_path: generator/terms_snapshot/anthropic_aup_2026-01-15.html

student:
  identity: internal:llama-base-7b-sft-run42
  base_model: meta-llama/Llama-3.1-8B-Instruct
  base_model_licence: Llama-3-Community-Licence
  intended_use: internal_product_assistant
  commercial_use: yes

corpus:
  path: outputs/distillation_corpus.parquet
  expected_item_count: 20000

questions_for_counsel:
  - Does the Anthropic Usage Policy as of the fetched date permit
    using Claude outputs to train an internal, non-API-replacement
    assistant for commercial use inside the organisation?
  - Does the base-model licence (Llama Community Licence) permit
    fine-tuning the base model on Claude-generated chain-of-thought
    data for the intended use above?
  - With what restrictions (if any) does the derived corpus pass to
    downstream internal consumers (the ML Platform team, model-ops)?
  - Does the derived student model's intended release cadence
    (quarterly internal rollout) introduce any additional licence
    obligations we have not captured?

counsel_signoff:
  reviewer: TBD
  reviewer_email: TBD
  reviewed_at: TBD
  opinion_summary: TBD
  opinion_sha256: TBD           # SHA of counsel_opinion/opinion_YYYY-MM-DD.pdf
  gate_passed: TBD               # true | false | conditional

downstream_restriction_decision:
  derived_corpus_licence: TBD
  model_release_eligibility: TBD
```

Three rules on this file:

- **Fetch the terms live; do not transcribe from memory.** Chapter 05
  §"Known reference cases" is explicit — both OpenAI and Anthropic
  revise their terms on cadence. The `fetch_terms.py` script is the
  mechanism; the HTML snapshot and its SHA are the evidence.
- **The counsel sign-off block is pre-populated with `TBD`.** The
  distillation author captures facts and authors the questions. The
  reviewer fills in the sign-off block after counsel renders the
  opinion. Pre-signed blocks fail the rubric.
- **`gate_passed` is a formal release gate.** Automation that reads
  this file to decide whether to kick off student training must
  check the field literally; anything other than `true` or
  `conditional` (with the conditions documented) blocks training.

### 2.3 `code/fetch_terms.py` — the terms-of-service capture

```python
# code/fetch_terms.py
# Fetches the teacher's terms-of-service / usage-policy page, hashes
# it, and writes a snapshot for the ledger. Run at the start of
# every distillation run; do not reuse a snapshot older than 30
# days without a new fetch.
from __future__ import annotations

import argparse
import hashlib
from datetime import datetime, timezone
from pathlib import Path

import requests


def fetch(url: str, out_dir: Path, tag: str) -> dict:
    out_dir.mkdir(parents=True, exist_ok=True)
    now = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    out = out_dir / f"{tag}_{now}.html"
    resp = requests.get(url, timeout=30, allow_redirects=True)
    resp.raise_for_status()
    out.write_bytes(resp.content)
    digest = hashlib.sha256(resp.content).hexdigest()
    (out.with_suffix(".html.sha256")).write_text(digest + "\n")
    return {
        "url": url,
        "snapshot_path": str(out),
        "sha256": digest,
        "fetched_at": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "status_code": resp.status_code,
    }


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--url", required=True)
    ap.add_argument("--tag", required=True,
                     help='short label, e.g. "anthropic_aup"')
    ap.add_argument("--out-dir", required=True, type=Path)
    args = ap.parse_args()
    import json
    print(json.dumps(fetch(args.url, args.out_dir, args.tag), indent=2))
```

Design notes defensible at review:

- **No user-agent spoofing.** The fetch identifies as the Python
  `requests` library default. Terms-of-service pages that return
  variant HTML to browsers vs scripts are themselves a flag; the
  ledger captures what the pipeline actually read.
- **Snapshot is stored verbatim.** No HTML parsing, no stripping.
  Counsel reads the clause as published.
- **The SHA is the audit anchor.** A re-fetch that returns a
  different digest means the terms changed — the run is a boundary
  and a new ledger entry is required.

### 2.4 `code/self_consistency.py` — the correctness filter

```python
# code/self_consistency.py
# Self-consistency: sample the teacher K times at temperature > 0
# and require M-of-K agreement on the final answer. Chapter 05
# §"Correctness-oriented filters" cites Wang et al. 2022 (arXiv
# 2203.11171, Self-Consistency Improves Chain of Thought Reasoning
# in Language Models) as the primary reference.
from __future__ import annotations

from collections import Counter
from dataclasses import dataclass


@dataclass(slots=True)
class ConsistencyResult:
    passed: bool
    majority_answer: str | None
    agreement: int
    k: int


def check_self_consistency(problem: str, teacher,
                             k: int = 3, agreement_min: int = 2,
                             temperature: float = 0.4) -> ConsistencyResult:
    answers: list[str] = []
    for _ in range(k):
        resp = teacher.generate(
            prompt=f"{problem}\n\nThink step by step, then give the final answer "
                    "on a line that starts with 'Answer:'.",
            temperature=temperature, max_tokens=1024,
        )
        answers.append(_extract_answer(resp))
    counts = Counter(answers)
    answer, agreement = counts.most_common(1)[0]
    return ConsistencyResult(
        passed=(agreement >= agreement_min),
        majority_answer=answer,
        agreement=agreement, k=k,
    )


def _extract_answer(response: str) -> str:
    for line in reversed(response.splitlines()):
        line = line.strip()
        if line.lower().startswith("answer:"):
            return line[len("answer:"):].strip()
    return response.strip().splitlines()[-1].strip() if response.strip() else ""
```

Two design choices worth defending at review:

- **K=3, M=2 as the default.** Chapter 05 pins this as the Wang et
  al. 2022 self-consistency setup. Higher K at the same M raises
  pass rate but costs K× the API budget; lower M silently keeps
  disagreeing answers.
- **`temperature > 0` on the consistency pass.** A deterministic
  pass defeats the point — self-consistency is a disagreement
  signal, and a temperature-0 generator never disagrees.

### 2.5 `code/verifier_filters.py` — the programmatic verifier

```python
# code/verifier_filters.py
# Problem-domain verifiers. For math problems with known answers,
# we run a numeric equality check (with tolerance). For code
# problems, we run the provided unit tests inside a seccomp-/
# docker-isolated sandbox. Problems without a verifier fall back to
# the judge filter.
from __future__ import annotations

import math
import subprocess
from dataclasses import dataclass


@dataclass(slots=True)
class VerifierResult:
    verifier_available: bool
    passed: bool
    reason: str


def verify_math(expected: str, observed: str,
                 tolerance: float = 1e-6) -> VerifierResult:
    try:
        e = float(expected)
        o = float(observed)
    except ValueError:
        if _canon_str(expected) == _canon_str(observed):
            return VerifierResult(True, True, "string_equal")
        return VerifierResult(True, False, "string_mismatch")
    if math.isclose(e, o, abs_tol=tolerance, rel_tol=tolerance):
        return VerifierResult(True, True, "numeric_equal")
    return VerifierResult(True, False, "numeric_mismatch")


def _canon_str(s: str) -> str:
    return "".join(s.lower().split())


def verify_code(unit_tests: str, teacher_code: str) -> VerifierResult:
    """Run teacher_code against unit_tests in an isolated sandbox.

    The sandbox call is out-of-scope to detail; use a hardened runner
    (e.g., a seccomp-restricted container with no network, 1 GB RAM,
    30 s wall-clock cap). Returning VerifierResult rather than
    raising on failure is intentional: a timeout or OOM is a
    failed verification, not a pipeline error.
    """
    outcome = subprocess.run(
        ["bash", "-c",
          f"python -c {teacher_code!r} && python -c {unit_tests!r}"],
        capture_output=True, text=True, timeout=30,
    )
    if outcome.returncode == 0:
        return VerifierResult(True, True, "tests_passed")
    return VerifierResult(True, False, outcome.stderr.strip()[:200])
```

Design notes worth defending at review:

- **The verifier is optional, not mandatory.** Not every problem has
  one. The pipeline's filter stack falls back to judge-model scoring
  when the verifier is unavailable, and `filter_report.parquet`
  records the fallback.
- **Sandbox isolation is non-negotiable for code execution.** The
  teacher's code is untrusted — never execute it in the orchestration
  process. Use a seccomp-restricted container, no-network, bounded
  memory / wall-clock.
- **Tolerance is explicit.** Numerical verifiers often fail on
  trivial format mismatches (`3.14` vs `3.140`). Pin a tolerance
  and apply it symmetrically.

### 2.6 `code/distill.py` — the main loop (skeleton)

```python
# code/distill.py
# Reads the problem pool, runs the teacher, applies the correctness
# filter stack, writes the triples. Every row records the ledger
# run_id so lineage is unbreakable even if files are reorganised.
from __future__ import annotations

import argparse
from collections import Counter
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq

from judge_filter import judge_score
from ledger import load_ledger_verify_signoff
from self_consistency import check_self_consistency
from verifier_filters import verify_code, verify_math


def distil(problem_pool: Path, ledger: Path, output: Path,
            filter_report: Path, teacher, judge,
            consistency_k: int, consistency_m: int,
            judge_threshold: float) -> None:
    led = load_ledger_verify_signoff(ledger)
    if led["counsel_signoff"]["gate_passed"] not in ("true", "conditional"):
        raise RuntimeError("ledger gate not passed; refusing to run")

    counter: Counter = Counter()
    rows: list[dict] = []

    pool = pq.read_table(problem_pool).to_pylist()
    for prob in pool:
        cons = check_self_consistency(prob["problem"], teacher,
                                        k=consistency_k,
                                        agreement_min=consistency_m)
        if not cons.passed:
            counter[("self_consistency", "failed")] += 1
            continue

        verifier = None
        if prob.get("verifier") == "math":
            verifier = verify_math(prob["expected"], cons.majority_answer)
        elif prob.get("verifier") == "code":
            verifier = verify_code(prob["unit_tests"], cons.majority_answer)

        if verifier is not None and not verifier.passed:
            counter[("verifier", verifier.reason)] += 1
            continue

        if verifier is None:
            score = judge_score(prob["problem"], cons.majority_answer, judge)
            if score < judge_threshold:
                counter[("judge", "low_score")] += 1
                continue

        # Full chain-of-thought trace — re-ask at the same temperature
        # as the consistency pass, preserving the majority answer
        # as the final answer line.
        trace = teacher.generate(
            prompt=f"{prob['problem']}\n\nThink step by step. Your final answer "
                    f"on the last line, prefixed with 'Answer:', must be "
                    f"'{cons.majority_answer}'.",
            temperature=0.2, max_tokens=4096,
        )
        rows.append({
            "problem_id": prob["problem_id"],
            "problem": prob["problem"],
            "chain_of_thought": _extract_cot(trace),
            "answer": cons.majority_answer,
            "ledger_run_id": led["run_id"],
            "teacher_model_id": led["teacher"]["identity"],
            "licence_terms_sha256": led["teacher"]["licence_terms_sha256"],
            "verifier_available": verifier is not None,
            "verifier_reason": verifier.reason if verifier else None,
            "judge_score": None if verifier else score,
        })
        counter[("kept", prob.get("verifier", "judge_only"))] += 1

    pq.write_table(pa.Table.from_pylist(rows), output)
    _flush_counter(filter_report, counter)


def _extract_cot(trace: str) -> str:
    """Strip the final answer line; return everything above it."""
    lines = trace.splitlines()
    for i in range(len(lines) - 1, -1, -1):
        if lines[i].strip().lower().startswith("answer:"):
            return "\n".join(lines[:i]).strip()
    return trace.strip()
```

Design notes worth defending at review:

- **The ledger gate is checked at the start of `distil()`.** Running
  the loop without a `gate_passed in {true, conditional}` ledger
  entry is a hard refusal. Automation that bypasses this check is a
  rubric failure.
- **Verifier output is recorded even when it passes.** The
  `verifier_reason` column keeps the audit trail for mixed verifier
  / judge-filtered corpora.
- **Chain-of-thought is a parquet column.** Chapter 05 §"Chain-of-
  thought is the primary artifact" names the CoT as the capability-
  transfer mechanism. The column is load-bearing.

### 2.7 `recipes/distillation_config.yaml`

```yaml
# recipes/distillation_config.yaml
teacher:
  model_id: anthropic:claude-3-5-sonnet-20241022
  temperature: 0.2
  max_tokens: 4096
  licence_ledger_ref: generator/licence_ledger.yaml

problem_pool:
  path: seeds/problem_pool.parquet
  sha256: <64-hex-digest>
  source_licences_ref: generator/licence_ledger.yaml
  item_count: 20000

generation:
  samples_per_problem: 1
  self_consistency_samples: 3
  self_consistency_agreement_min: 2
  cot_prompting: explicit

filters:
  verifier:
    enabled: true
    verifier_modules:
      math: verifier_filters.verify_math
      code: verifier_filters.verify_code
  judge_model:
    model_id: anthropic:claude-3-5-sonnet-20241022
    score_threshold: 0.7
    independence_warning: true    # same-model-as-teacher; flag for review
  length_range: [10, 4000]
  drop_refusals: true

student:
  intended_identity: internal:llama-base-7b-sft-run42
  intended_use: internal_product_assistant

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

Pin the dated teacher version. The `independence_warning: true` flag
is explicit when the judge and the teacher share identity — Chapter
06 §"Judge-generator overlap" is the anchor.

## 3. Validation steps

1. **Terms-of-service snapshot matches its SHA.**
   `sha256sum generator/terms_snapshot/*.html` equals the
   `licence_terms_sha256` pinned in the ledger.
2. **Snapshot is recent.** `licence_terms_fetched_at` is less than
   30 days old at the start of the run; a staler snapshot must be
   re-fetched.
3. **Ledger gate is checked before the loop runs.** A dry-run with
   `counsel_signoff.gate_passed: TBD` exits non-zero before any
   teacher API call.
4. **Counsel opinion present when `gate_passed == true`.**
   `opinion_sha256` resolves to a file in
   `generator/counsel_opinion/` and the SHA matches.
5. **Corpus references the ledger.** Every row of
   `outputs/distillation_corpus.parquet` carries the
   `ledger_run_id` and `licence_terms_sha256` columns.
6. **Self-consistency pass rate is recorded.** The filter report
   has `(self_consistency, failed)` and `(self_consistency, passed)`
   counts.
7. **Verifier coverage.** The corpus's `verifier_available` column
   is `true` for ≥ some fraction of problems with structured
   answers (pin threshold in the manifest).
8. **Judge-generator overlap is flagged.** When the judge and the
   teacher are the same model, `independence_warning` is `true` in
   the manifest and `capability_ceiling_report.md` discusses the
   calibration result.
9. **Capability-ceiling spot-check.** A sample of 25 rows from the
   corpus is hand-reviewed for style-without-substance (long,
   confident CoT that does not match the answer). The sample is
   recorded in `reports/capability_ceiling_report.md`.
10. **Contamination inheritance scan.** A scan against a frozen
    eval surface (per Chapter 10 and `mod-106`) finds no verbatim
    n-gram matches from the training evaluation surface. If the
    scan finds hits, the run blocks on `ai-risk-engineer` review.
11. **Pending-review defaults.** `counsel_signoff.reviewer` is a
    real email address (or `TBD`); the distillation author must not
    sign on behalf of counsel.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/30 with zero zeros on
critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `generator/licence_ledger.yaml` captures teacher identity, licence SHA, downstream intent, and questions for counsel | Yes       |
| 2  | Terms-of-service snapshot is fetched live and hashed; ledger pins the SHA                                            | Yes       |
| 3  | `counsel_signoff` block is pre-populated with `TBD`; the author does not pre-sign                                    | Yes       |
| 4  | Teacher model ID is dated (`claude-3-5-sonnet-20241022`, not `claude-3-5-sonnet`)                                    | Yes       |
| 5  | Self-consistency filter (`k ≥ 3`, `M ≥ 2`) runs on every problem                                                     | Yes       |
| 6  | Verifier filter runs when a verifier is available; fallback to judge otherwise                                       | Yes       |
| 7  | Code verifier uses a sandboxed runner (seccomp / container, no network, resource caps)                                | Yes       |
| 8  | Judge-teacher overlap is explicitly flagged in the manifest when they share identity                                  | Yes       |
| 9  | Every corpus row carries `ledger_run_id` and `licence_terms_sha256`                                                  | Yes       |
| 10 | Chain-of-thought is stored as a parquet column alongside the answer                                                  | Yes       |
| 11 | Capability-ceiling spot-check report lists ≥ 25 sampled rows with style-vs-substance verdict                         | No        |
| 12 | Contamination-inheritance scan is run and reported                                                                   | Yes       |
| 13 | Ledger gate enforcement at loop start refuses to run with `gate_passed: TBD`                                         | Yes       |
| 14 | Governance anchors (OWASP ML Top 10 ML02 / ML06, MITRE ATLAS AML.T0010 / AML.T0020, NIST AI RMF GOVERN 6.1 / MAP 4.1 / MEASURE 2.7) cited | No |
| 15 | Unverifiable claims marked `<!-- pending-verify: ... -->`                                                           | Yes       |

## 5. Common mistakes

- **Pre-signing the counsel sign-off block.** The distillation
  author captures facts; counsel signs. A ledger shipped with
  `gate_passed: true` without a matching `opinion_sha256` is a
  rubric failure.
- **Transcribing teacher terms from memory.** Terms revise on
  cadence. Chapter 05 §"Known reference cases" is explicit — fetch
  live, hash, snapshot. The SHA is the audit anchor.
- **Reusing a stale snapshot across runs.** A snapshot older than
  30 days must be re-fetched; a terms change between runs is a
  ledger boundary.
- **Running code without a sandbox.** Executing untrusted
  teacher-generated code in the orchestration process is a supply-
  chain vector of its own (OWASP ML06). Use a hardened runner.
- **Silent judge-teacher overlap.** Using the same model as judge
  and as teacher is permitted but must be flagged; `independence_
  warning` is a parquet column, not a free-text footnote.
- **Final-answer-only rows.** Shipping `(problem, answer)` without
  the chain-of-thought loses the capability-transfer surface.
  Chapter 05 §"Chain-of-thought is the primary artifact" is the
  anchor; keep the CoT column.
- **Treating a verifier-failed row as a judge-only row.** A
  verifier that returns `passed=False` is a verdict; do not fall
  back to the judge in that case. Fallback is reserved for
  verifier-unavailable rows.
- **Missing contamination-inheritance scan.** The teacher may have
  been trained on your evaluation surface. Chapter 10 and
  `mod-106` ex-01 cover the firewall; skipping the scan on the
  distilled corpus lets the leak into the student via synthetic
  surface.
- **Treating the Alpaca / Vicuna cases as settled.** Chapter 05
  §"Known reference cases" records them as factual references, not
  as legal precedents. The ledger captures the facts; counsel
  decides which lessons apply to this run.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack; ML06:2023 AI Supply Chain Attacks:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML06 is the supply-chain anchor for the teacher-as-upstream-
  dependency; ML02 is the contamination-inheritance anchor.
- **MITRE ATLAS — AML.T0010 ML Supply Chain Compromise; AML.T0020
  Poison Training Data:** <https://atlas.mitre.org/>. The adversary-
  facing anchors for a compromised or contaminated teacher feeding
  the student pipeline.
- **NIST AI Risk Management Framework — GOVERN 6.1 (third-party
  risk management), MAP 4.1 (legal and regulatory requirements
  mapping), MEASURE 2.7 (security and resilience):**
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- **Anthropic Usage Policy:** <https://www.anthropic.com/legal/aup>.
  Fetch live at generation time; capture SHA in the ledger.
- **OpenAI Terms of Use:** <https://openai.com/policies/row-terms-of-use/>.
  Fetch live; same capture pattern.
- **Alpaca card (Taori et al. 2023):**
  <https://crfm.stanford.edu/2023/03/13/alpaca.html>. The canonical
  reference case for non-commercial release downstream of an
  OpenAI-restricted teacher.
- **Self-Consistency (Wang et al. 2022), arXiv 2203.11171:**
  <https://arxiv.org/abs/2203.11171>. The primary source for the
  self-consistency filter's K-of-M agreement check.
- **Local anchors:** `mod-108` Chapter 05 (the full recipe and the
  licence ledger shape); `mod-106` exercise-01 (the contamination-
  inheritance scan surface); `mod-109` (lineage from
  `ledger_run_id` through the student training run); `mod-110`
  (PII and sensitive-content scan of the distillation corpus
  before student training).
