# Exercise 04 — Model Card Data Slice Authoring (Solution)

## 1. Solution overview

A Model Card (Mitchell et al., 2019) is the human-readable contract that
accompanies a trained model into production. Of its many sections, the one
that most directly supports fairness, robustness, and incident response is
the **Quantitative Analyses — Disaggregated Evaluation** section (often
called "data slices" or "subgroup results"). This section reports the
model's primary metric not only overall, but **per subpopulation** along
axes the team considers relevant. The exercise asks the learner to author
this section for one concrete model, commit to a defensible choice of
axes and metrics, produce the table, and write the honest-caveats
narrative that sits beside it.

The model we commit to in this reference solution is **`search-reranker-v3`**,
a cross-encoder re-ranker that consumes the `features.search_reranker_v3`
feature table introduced in mod-111 ex-01 and emits a ranking score and a
calibrated click-probability head. We pick this model because (a) ranking
is the exercise case where disaggregated evaluation has the most teeth —
a locale with a 15 nDCG@10 gap is obviously broken; (b) it ties the
module's exercises together (feature contract from mod-111, PII posture
from mod-110, dataset card from mod-112 ex-02, GPAI framing from ex-03);
and (c) a ranker's metric (nDCG@10) is unambiguous enough that a reviewer
can argue about *what the gaps mean* rather than *what the numbers are*.

Disaggregated evaluation is **necessary but not sufficient** for fairness.
It tells you *where* the model is weakest among the slices you thought to
measure. It does not tell you: whether the slice definitions themselves
encode harm; whether the labels inside each slice are equally reliable;
whether the slices you did not measure (intersectional, unobserved
demographics, long-tail dialects) hide worse gaps; or whether equal
performance is even the right fairness target for your product. The
"Known limitations" subsection of the deliverable is where these caveats
live, and reviewers should weight it as heavily as the table itself.

The deliverables for this exercise are three artifacts in one Model Card
Data-Slices section: (A) the Markdown prose + disaggregated table that
would appear in the public model card, (B) a Hugging Face model-card YAML
frontmatter block with per-slice `model-index` entries so downstream
tooling can parse it, and (C) a reproducible Python snippet that
regenerates the table from a held-out test set. Each artifact must agree
with the others: axes in the YAML match axes in the table, bucket names
in the code match bucket names in the YAML, and the overall value in the
table matches the `type: overall` entry in the YAML.

Governance framing. In the NIST AI RMF, disaggregated evaluation is the
archetypal **MEASURE** artifact — it is the "measure" of risk the rest of
the framework (MANAGE, GOVERN) acts on; the choice of *which* axes to
include is a **MAP** activity (you can only measure what you've mapped as
a context of use). In OWASP ML Top 10, per-slice performance gaps are the
precondition that makes **ML04:2023 Membership Inference** and **ML01:2023
Input Manipulation Attack** exploitable at scale — an adversary who finds
your worst-performing slice has a cheap attack surface. In MITRE ATLAS,
per-slice performance is a **reconnaissance surface**: an attacker runs
probes to find the worst slice, then concentrates abuse there. The
correct mitigation for all three is not to *hide* the per-slice numbers;
it is to publish them, monitor them, and set thresholds that trigger a
rollback. Hiding the gap does not close the gap — it just means the
adversary finds it before the defender does.

Finally — and this is the signal of a high-quality card — the solution
explicitly lists the axes we **did not** measure and **why**. The two most
common honest absences in a search re-ranker card are user-level
demographics (we do not store age, gender, or inferred ethnicity, see
mod-110) and voice-vs-text input modality (the re-ranker sees a tokenised
query string and cannot distinguish the upstream source). Naming these
absences is not an evasion — it is the card telling the reader *what the
card does not know*, which is the most important thing a card can say.

## 2. Worked answer or implementation

### 2.1 Artifact A — "Data Slices" section of the model card (Markdown)

```markdown
## Quantitative Analyses — Disaggregated Evaluation

### What this section is for

This section reports the primary ranking quality metric (nDCG@10) and a
calibration check (expected calibration error, ECE) for `search-reranker-v3`
on a held-out evaluation set, disaggregated by subpopulation axes. It
exists so that a reader — product owner, SRE, auditor, or an affected
end user's advocate — can see **where the model is weakest** rather than
only its aggregate score.

What this section does **not** prove:
- It does not prove that `search-reranker-v3` is "fair". Per-slice parity
  on nDCG@10 is one input to a fairness argument, not the argument
  itself.
- It does not cover **intersectional** slices (e.g. `query_locale ×
  device`). Those are deliberately deferred — see "Known limitations".
- It does not cover axes we do not store (see "Axes not measured").
- The numbers are from **one** held-out evaluation window; see the
  monitoring plan in mod-111 ex-01 for the ongoing live measurement.

> **Note — illustrative numbers.** The specific values in the table
> below are illustrative placeholders chosen to demonstrate the shape of
> the artifact. Reviewers MUST regenerate the table against a real
> held-out set before this card is published externally. See the code
> snippet in §2.3 and the regeneration validator in §3.

### Subpopulation axes measured

We report three axes, each with 3–5 buckets. Axes were selected by three
criteria: (1) business impact (a regression here is a P1), (2) we actually
store the signal and can join it to judgments, and (3) the bucket
boundaries are reproducible from logs without manual labelling.

| Axis               | Buckets                                                           | Rationale                                                                   |
| ------------------ | ----------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `query_locale`     | en-US, en-GB, de-DE, ja-JP, es-MX                                 | Covers the five locales that account for >90% of traffic; cross-script coverage ensures tokeniser issues surface. |
| `query_vertical`   | shopping, local, informational, media                              | Vertical determines which retrieval leg feeds the re-ranker and therefore which score distribution it sees. |
| `query_class`      | head, torso, tail (by log-frequency of the normalized query)       | Head queries dominate training data; tail queries are where click-sparsity bites nDCG estimation. |
| `device`           | desktop, mobile-web, mobile-app                                    | Mobile sessions have shorter viewports — nDCG@10 can be misleading if users only see top-5. |
| `session_position` | first-query, follow-up                                             | Follow-up queries carry reformulation signal the ranker should exploit; regressions here indicate state-leakage bugs. |

### Axes NOT measured (honest absence)

- **User-level demographics** (age, gender, inferred ethnicity, income
  bracket). We do not store these fields; see mod-110 PII posture. We
  cannot slice by what we do not record, and we deliberately choose not
  to infer them from behavioural proxies.
- **Query input modality** (voice vs. text vs. image-to-text). By the
  time a query reaches the re-ranker it is a tokenised string; the
  upstream modality is not available in the re-ranker's feature table.
  Upstream teams own this slice.
- **Accessibility-tech users** (screen reader, keyboard-only navigation).
  Not stored in the ranking feature set; served by a separate
  front-end-level evaluation we do not own.
- **Logged-out vs. logged-in users**. We hash both, and the hashing
  policy (mod-110) deliberately makes the two indistinguishable at the
  feature-table layer.

Naming these absences is intentional: a reviewer should treat *missing
axes* as a signal of what the card does not cover, not as an implicit
claim that the model performs equally on them.

### Primary and secondary metrics

| Metric                     | Role       | Why                                                                                       |
| -------------------------- | ---------- | ----------------------------------------------------------------------------------------- |
| **nDCG@10**                | Primary    | Standard ranking quality metric; sensitive to position; well-understood by stakeholders.  |
| Recall@10                  | Secondary  | Guards against the "nDCG looks fine but the relevant doc dropped out of top-10" failure.  |
| ECE (click-prob head)      | Calibration| The click-probability head feeds downstream bidding/ordering; miscalibration = silent revenue loss. |
| p95 scoring latency        | Operational| Reported but not a fairness signal; included so the reader sees the full operating point. |

Metric choice is defended in §2.4.

### Disaggregated performance table (illustrative)

Overall: **nDCG@10 = 0.741** (95% CI [0.736, 0.746], n_queries = 48,210,
n_judged = 482,100 query-doc pairs). All deltas below are **percentage
points vs. overall nDCG@10 = 0.741** (so −3.2 pp = 0.741 → 0.709).

Bootstrap: 1,000 iterations, resampled at the **query** level (not the
query-doc level) to preserve within-query correlation. Random seed 20260301.

#### Axis: `query_locale`

| Bucket  | nDCG@10 | 95% CI (bootstrap) | n_queries | n_judged | delta_vs_overall_pp |
| ------- | ------- | ------------------ | --------- | -------- | ------------------- |
| en-US   | 0.763   | [0.757, 0.769]     | 21,044    | 210,440  | +2.2                |
| en-GB   | 0.751   | [0.742, 0.760]     | 8,112     | 81,120   | +1.0                |
| de-DE   | 0.729   | [0.719, 0.740]     | 6,880     | 68,800   | −1.2                |
| es-MX   | 0.718   | [0.707, 0.729]     | 5,901     | 59,010   | −2.3                |
| ja-JP   | 0.681   | [0.666, 0.695]     | 4,004     | 40,040   | −6.0                |
| (other) | —       | low-n — not reported | 2,269   | 22,690   | —                   |

#### Axis: `query_vertical`

| Bucket         | nDCG@10 | 95% CI (bootstrap) | n_queries | n_judged | delta_vs_overall_pp |
| -------------- | ------- | ------------------ | --------- | -------- | ------------------- |
| shopping       | 0.772   | [0.765, 0.778]     | 17,110    | 171,100  | +3.1                |
| informational  | 0.748   | [0.741, 0.755]     | 15,988    | 159,880  | +0.7                |
| local          | 0.731   | [0.721, 0.741]     | 8,702     | 87,020   | −1.0                |
| media          | 0.701   | [0.689, 0.713]     | 6,410     | 64,100   | −4.0                |

#### Axis: `query_class`

| Bucket  | nDCG@10 | 95% CI (bootstrap) | n_queries | n_judged | delta_vs_overall_pp |
| ------- | ------- | ------------------ | --------- | -------- | ------------------- |
| head    | 0.801   | [0.796, 0.806]     | 19,284    | 192,840  | +6.0                |
| torso   | 0.747   | [0.741, 0.753]     | 19,284    | 192,840  | +0.6                |
| tail    | 0.672   | [0.662, 0.683]     | 9,642     | 96,420   | −6.9                |

#### Axis: `device`

| Bucket      | nDCG@10 | 95% CI (bootstrap) | n_queries | n_judged | delta_vs_overall_pp |
| ----------- | ------- | ------------------ | --------- | -------- | ------------------- |
| desktop     | 0.754   | [0.748, 0.760]     | 20,730    | 207,300  | +1.3                |
| mobile-web  | 0.733   | [0.726, 0.740]     | 15,427    | 154,270  | −0.8                |
| mobile-app  | 0.739   | [0.732, 0.746]     | 12,053    | 120,530  | −0.2                |

#### Axis: `session_position`

| Bucket      | nDCG@10 | 95% CI (bootstrap) | n_queries | n_judged | delta_vs_overall_pp |
| ----------- | ------- | ------------------ | --------- | -------- | ------------------- |
| first-query | 0.749   | [0.744, 0.754]     | 36,158    | 361,580  | +0.8                |
| follow-up   | 0.719   | [0.711, 0.727]     | 12,052    | 120,520  | −2.2                |

#### Calibration (ECE on click-probability head, lower is better)

| Axis / bucket         | ECE   |
| --------------------- | ----- |
| overall               | 0.023 |
| query_locale = ja-JP  | 0.041 |
| query_class = tail    | 0.047 |
| query_vertical = media| 0.038 |

> All values in the four tables above are **illustrative**. Do not quote
> them externally. The regeneration code in §2.3 is the source of truth;
> see §3 for the validator that enforces "numbers in this card came from
> the current code on the current eval set".

### Gap analysis

Three gaps exceed our internal "investigate" threshold of 3 pp below
overall on nDCG@10:

1. **`query_locale = ja-JP`: −6.0 pp.** Working hypothesis: tokeniser
   under-segments Japanese morphology; head-tail skew is worse in ja-JP
   because the training-data mix under-weights it. **Commitment: fix.**
   Owner: ranking modelling. Target: close to within 2.0 pp of overall in
   v3.1. Rollback plan: if a v3.1 candidate improves ja-JP by >=3 pp but
   regresses en-US by >=1 pp, we hold the launch and escalate to the
   steering committee for a conscious trade-off decision. Not a silent
   call.
2. **`query_class = tail`: −6.9 pp.** Expected direction (tail queries
   have sparse clicks, so both the model and the metric estimate are
   noisier). **Commitment: monitor, do not fix in v3.** The structural
   remediation is better judgments for tail queries, which is an
   annotation-program change (see mod-107), not a model change. We will
   track the gap in the monitoring dashboard (mod-111) with an alert at
   −8.5 pp.
3. **`query_vertical = media`: −4.0 pp.** Working hypothesis: the media
   retrieval leg emits a score distribution that the re-ranker was not
   co-trained on. **Commitment: fix (feature-level).** Owner: feature
   platform (mod-111 ex-01 feature table owner). Target: re-emit the
   `media_retrieval_score_v2` field with corrected scaling in Q2; we
   expect this to close roughly half the gap.

The `session_position = follow-up` (−2.2 pp) gap is below the
investigate threshold but close; we add it to the monitoring dashboard
with an alert at −3.5 pp and revisit at the next review.

### Known limitations

- **Intersectional slices not shown.** We report axes marginally; the
  worst real-world gap is very likely at an intersection
  (e.g. `ja-JP × mobile-web × tail`). We do not publish the full
  intersectional cube because many cells would fall below the
  `n_queries >= 100` floor and reporting them would be misleading. See
  §2.4 design rationale.
- **Sample-size floor.** Any bucket with fewer than 100 queries is
  reported as "low-n — not reported" rather than with a noisy point
  estimate. This hides some long-tail locales (see the `(other)` row).
- **Judgment-set skew.** The held-out judgments were sampled from the
  same distribution as training traffic; underserved populations are
  underserved in the evaluation too, which *under-estimates* the gap.
- **Necessary not sufficient.** Equal nDCG@10 across these axes does not
  mean the model is fair; it means the model is *equally good at the
  thing we chose to measure, on the slices we chose to report, using
  labels we chose to trust.* Every one of those choices is contestable.
```

### 2.2 Artifact B — Hugging Face model-card YAML frontmatter

```yaml
---
# Hugging Face model-card frontmatter. Field names follow the public
# model-card schema on huggingface.co/docs/hub/model-cards. Some nested
# fields are marked research-note where the exact key spelling or
# allowed values are not confirmed from first-party docs.
language:
  - en
  - de
  - ja
  - es
license: other
license_name: internal-proprietary
license_link: LICENSE
library_name: transformers  # cross-encoder architecture
tags:
  - ranking
  - search
  - cross-encoder
  - reranker
  - information-retrieval
datasets:
  - internal/search-reranker-eval-2026q1   # see mod-112 ex-02 dataset card
metrics:
  - ndcg
  - recall
  - expected_calibration_error  # <!-- research-note: exact HF metric slug -->
pipeline_tag: text-ranking     # <!-- research-note: confirm this is the canonical tag -->
base_model: internal/search-reranker-v2
model-index:
  - name: search-reranker-v3
    results:
      # Overall
      - task:
          type: text-ranking           # <!-- research-note: confirm -->
          name: Learning to Rank
        dataset:
          type: internal/search-reranker-eval-2026q1
          name: Search Reranker Eval 2026 Q1
          split: test
        metrics:
          - type: ndcg_at_10
            name: nDCG@10
            value: 0.741
            verified: false
          - type: recall_at_10
            name: Recall@10
            value: 0.812
            verified: false
          - type: ece
            name: ECE (click-prob head)
            value: 0.023
            verified: false

      # Slice: query_locale = ja-JP (worst locale)
      - task:
          type: text-ranking
          name: Learning to Rank
        dataset:
          type: internal/search-reranker-eval-2026q1
          name: Search Reranker Eval 2026 Q1
          split: test
          config: query_locale=ja-JP      # <!-- research-note: HF uses `config` for split sub-variant; confirm for slices -->
        metrics:
          - type: ndcg_at_10
            name: nDCG@10 (ja-JP)
            value: 0.681
            verified: false
          - type: ece
            name: ECE (ja-JP)
            value: 0.041
            verified: false

      # Slice: query_class = tail
      - task:
          type: text-ranking
          name: Learning to Rank
        dataset:
          type: internal/search-reranker-eval-2026q1
          name: Search Reranker Eval 2026 Q1
          split: test
          config: query_class=tail
        metrics:
          - type: ndcg_at_10
            name: nDCG@10 (tail)
            value: 0.672
            verified: false
          - type: ece
            name: ECE (tail)
            value: 0.047
            verified: false

      # Slice: query_vertical = media
      - task:
          type: text-ranking
          name: Learning to Rank
        dataset:
          type: internal/search-reranker-eval-2026q1
          name: Search Reranker Eval 2026 Q1
          split: test
          config: query_vertical=media
        metrics:
          - type: ndcg_at_10
            name: nDCG@10 (media)
            value: 0.701
            verified: false
---
```

Notes on the YAML:

- `verified: false` on every entry is deliberate — these are
  self-reported numbers, not independently verified by a benchmark
  provider. Keep it false unless/until you plug in an external
  leaderboard.
- The `config:` sub-key is the mechanism we use to encode a slice
  without inventing a new split per slice; the spelling is marked
  `research-note` because first-party HF docs on model-card YAML are
  looser about this than one would like.
- We publish only the overall + the three worst slices in the
  `model-index` to keep the YAML scannable. The full table lives in the
  Markdown body (Artifact A); the YAML is the machine-readable summary.

### 2.3 Artifact C — Reproducible evaluation snippet (`snippet.py`)

```python
"""Regenerate the disaggregated nDCG@10 table for search-reranker-v3.

Dependencies (pinned):
  pandas==2.2.*
  numpy==1.26.*
  scipy==1.13.*     # for bootstrap CI
  scikit-learn==1.4.* # for ndcg_score
"""
from __future__ import annotations

import numpy as np
import pandas as pd
from scipy.stats import bootstrap
from sklearn.metrics import ndcg_score

SLICING_AXES = ["query_locale", "query_vertical", "query_class",
                "device", "session_position"]
PRIMARY_K = 10
N_BOOTSTRAP = 1000
MIN_N_QUERIES = 100
RNG = np.random.default_rng(20260301)


def load_eval_set(path: str) -> pd.DataFrame:
    """Columns: query_id, doc_id, relevance (0..4), score,
    query_locale, query_vertical, query_class, device, session_position."""
    return pd.read_parquet(path)


def score_model(df: pd.DataFrame) -> pd.DataFrame:
    """In a real pipeline this calls the served model. Here we assume
    `score` is already populated on df (e.g. produced by an offline
    scoring job against the frozen v3 checkpoint)."""
    assert "score" in df.columns, "score column must be prepopulated"
    return df


def ndcg_at_k_per_query(group: pd.DataFrame, k: int = PRIMARY_K) -> float:
    y_true = group["relevance"].to_numpy().reshape(1, -1)
    y_score = group["score"].to_numpy().reshape(1, -1)
    if y_true.shape[1] < 2:
        return float("nan")
    return float(ndcg_score(y_true, y_score, k=k))


def mean_ndcg_with_ci(per_query: np.ndarray, n_boot: int = N_BOOTSTRAP
                     ) -> tuple[float, float, float]:
    per_query = per_query[~np.isnan(per_query)]
    if per_query.size == 0:
        return (float("nan"), float("nan"), float("nan"))
    point = float(per_query.mean())
    res = bootstrap(
        (per_query,), np.mean, n_resamples=n_boot,
        confidence_level=0.95, method="percentile",
        random_state=RNG,
    )
    return point, float(res.confidence_interval.low), float(res.confidence_interval.high)


def disaggregate(df: pd.DataFrame, axes: list[str]) -> pd.DataFrame:
    per_q = (df.groupby("query_id")
               .apply(ndcg_at_k_per_query, include_groups=False)
               .rename("ndcg_at_10"))
    per_q_meta = df.drop_duplicates("query_id").set_index("query_id")[axes]
    per_q_df = per_q_meta.join(per_q)

    overall_point, overall_lo, overall_hi = mean_ndcg_with_ci(per_q_df["ndcg_at_10"].to_numpy())
    rows: list[dict] = [{
        "axis": "overall", "bucket": "overall",
        "ndcg_at_10": overall_point, "ci_lo": overall_lo, "ci_hi": overall_hi,
        "n_queries": int(per_q_df["ndcg_at_10"].notna().sum()),
        "delta_vs_overall_pp": 0.0,
    }]
    for axis in axes:
        for bucket, grp in per_q_df.groupby(axis):
            n_q = int(grp["ndcg_at_10"].notna().sum())
            if n_q < MIN_N_QUERIES:
                rows.append({"axis": axis, "bucket": bucket, "ndcg_at_10": None,
                             "ci_lo": None, "ci_hi": None, "n_queries": n_q,
                             "delta_vs_overall_pp": None, "note": "low-n — not reported"})
                continue
            pt, lo, hi = mean_ndcg_with_ci(grp["ndcg_at_10"].to_numpy())
            rows.append({
                "axis": axis, "bucket": str(bucket),
                "ndcg_at_10": round(pt, 3),
                "ci_lo": round(lo, 3), "ci_hi": round(hi, 3),
                "n_queries": n_q,
                "delta_vs_overall_pp": round((pt - overall_point) * 100, 1),
            })
    return pd.DataFrame(rows)


def to_markdown(table: pd.DataFrame) -> str:
    out = ["| axis | bucket | nDCG@10 | 95% CI | n_queries | delta_vs_overall_pp |",
           "| --- | --- | --- | --- | --- | --- |"]
    for _, r in table.iterrows():
        ci = f"[{r['ci_lo']}, {r['ci_hi']}]" if r.get("ci_lo") is not None else "—"
        ndcg = r["ndcg_at_10"] if r["ndcg_at_10"] is not None else "low-n"
        delta = r["delta_vs_overall_pp"] if r["delta_vs_overall_pp"] is not None else "—"
        out.append(f"| {r['axis']} | {r['bucket']} | {ndcg} | {ci} | {r['n_queries']} | {delta} |")
    return "\n".join(out)


if __name__ == "__main__":
    df = score_model(load_eval_set("eval/search-reranker-eval-2026q1.parquet"))
    table = disaggregate(df, SLICING_AXES)
    print(to_markdown(table))
```

The snippet is deliberately short and single-file. It does **not** fetch
a model checkpoint, which keeps the artifact safe to run in a shared
notebook without triggering an unintended download; the `score_model`
hook expects the scoring step to have been done separately (so the eval
is reproducible even when the serving binary has moved on).

### 2.4 Design rationale (hardest decisions)

**Why these five axes and not others.** The five axes
(`query_locale`, `query_vertical`, `query_class`, `device`,
`session_position`) were picked because each one meets three criteria
simultaneously: (1) the signal is already present in the re-ranker
feature table (we do not need a new join), (2) a regression of >=3 pp on
that axis is one we would act on, and (3) bucket boundaries are
reproducible from logs. Axes that pass (2) but fail (1) — e.g. user-level
demographics, accessibility-tech users — are listed as honest absences
in the "Axes NOT measured" subsection rather than silently dropped. This
is important: readers who scan only the table see five axes; readers who
read the prose see twelve, five measured and seven consciously not.

**Why nDCG@10 as primary.** For a search re-ranker, the alternatives are
MRR, MAP, Recall@k, and raw click-through rate. nDCG@10 wins because:
(a) it is **graded** — our judgments are 0..4, and MRR/MAP collapse
graded relevance to binary; (b) it is **position-weighted** — a bad doc
at rank 1 hurts more than a bad doc at rank 10, which matches user
behaviour; (c) `k=10` matches the typical first-page user viewport on
desktop (mobile viewports see ~5, which is a weakness of this choice
that we flag in "Known limitations" rather than hide); and (d) it is
stakeholder-legible — product owners can reason about "nDCG dropped 3
pp" without a tutorial. We report Recall@10 as a secondary to catch the
"nDCG looks fine but the key doc dropped out of top-10" failure mode,
and we report ECE on the click-probability head because the downstream
bidding stack *consumes the probability*, not the ranking, and
miscalibration there is silent revenue loss — the kind of regression
that would not show up in nDCG at all.

**Why bootstrap CIs and not asymptotic normal.** Three reasons. (1) The
per-query nDCG@10 distribution is **not** normal — it has a hard ceiling
at 1.0, a floor at 0.0, and a heavy left tail on tail queries. The
normal-approximation CI would under-cover the left tail. (2) Resampling
*at the query level* (not the query-doc level) is the only way to
preserve within-query correlation; bootstrap handles this naturally by
resampling the unit of independence, while asymptotic formulas would
require us to derive a cluster-robust variance, which is error-prone.
(3) With 1,000 iterations on ~50k queries we can afford the compute;
the regenerator takes about two minutes on commodity hardware. The
trade-off is that the CIs are **non-deterministic** without a pinned
seed; we pin `20260301` so a reviewer can rerun and get the same
numbers.

**What intersectional slicing was skipped and why.** The full
intersectional cube for our five axes would be 5 x 4 x 3 x 3 x 2 = 360
cells. Most of those cells have fewer than 100 queries and would be
dropped under our `MIN_N_QUERIES = 100` floor, leaving a sparse
unreadable table. More importantly, "add every intersection" is a
*presentation* fix that would make the card technically more complete
and practically less read. We keep the table marginal and add a
monitoring task (owner: ranking SRE) to compute a two-way interaction
heatmap (the ten most-populated intersections of `query_locale ×
device`) that is reviewed quarterly but not published in the card.
Reviewers who want the full cube can regenerate with
`SLICING_AXES = [...]` expanded to pairs. This is the single most common
critique of model cards (Mitchell et al. 2019 flag it directly) and
hiding it in prose rather than fixing it is a conscious choice we
document, not one we pretend does not exist.

## 3. Validation steps

A reviewer should be able to run all of these in under ten minutes.
Numbered so a checklist tool can parse them.

1. **Markdown lint.** Run `markdownlint-cli2 SOLUTION.md` (or equivalent).
   No `MD007`, `MD013`, `MD024`, or `MD040` violations at defaults, or
   explicit per-file disables for the ones we accept.
2. **Every slice has sample size and CI.** For every row in every
   disaggregated table in Artifact A: verify the `n_queries` column is
   populated with an integer >= 0 and the `95% CI` column is populated
   with `[lo, hi]` where lo < pt < hi — or the row is explicitly marked
   `low-n — not reported`. No blank cells.
3. **Low-n floor enforced.** No row with `n_queries < 100` carries a
   numeric `nDCG@10`. Every low-n row carries the exact string
   `low-n — not reported`.
4. **Overall anchor matches YAML.** The `overall` nDCG@10 in the
   Markdown table equals the value in the first `model-index` entry of
   the YAML (Artifact B). If they diverge the YAML is wrong.
5. **Axis coverage YAML <-> Markdown.** Every slice `config:` in the
   YAML `model-index` (e.g. `query_locale=ja-JP`) corresponds to exactly
   one row in one of the per-axis Markdown tables. Converse not
   required — YAML can publish a subset.
6. **Snippet parses.** `python -c "import ast; ast.parse(open('snippet.py').read())"`
   exits 0. (We do not require the snippet to execute without an eval
   parquet file; it is the regenerator, not a self-contained demo.)
7. **Snippet pins bootstrap iterations.** Grep `snippet.py` for
   `N_BOOTSTRAP` and confirm it is set to 1000 (matches the figure
   stated in Artifact A). A card that says "1000 iterations" and a
   regenerator that uses 100 is lying.
8. **Snippet pins the seed.** Grep `snippet.py` for `default_rng(` and
   confirm the seed matches the seed stated in Artifact A
   (`20260301`).
9. **Delta arithmetic.** For every numeric row, verify
   `delta_vs_overall_pp == round((row_ndcg - overall_ndcg) * 100, 1)`
   to within 0.1 pp. A card where the deltas do not arithmetic is
   embarrassing and the fastest tell of a copy-paste error.
10. **Gap analysis covers the worst three.** The three worst rows by
    `delta_vs_overall_pp` must each be discussed in the "Gap analysis"
    narrative with an explicit fix/accept/monitor commitment and an
    owner.
11. **Honest absence present.** The "Axes NOT measured" subsection
    exists and names at least three axes we do not measure, each with a
    one-line "why". A card without this subsection is below quality bar.
12. **No roles are named humans.** Grep for first-last name patterns
    (`[A-Z][a-z]+ [A-Z][a-z]+`) in the owner fields — all owners should
    be team/role names (e.g. "ranking modelling", "feature platform"),
    never individuals.

## 4. Rubric or review checklist

Scored 0 (missing/wrong), 1 (partial), 2 (solid). Pass threshold: **>= 28/40**,
no critical (**C**) row below 1.

| # | Criterion                                                                                 | Critical | 0–2 |
| - | ----------------------------------------------------------------------------------------- | -------- | --- |
| 1 | One concrete model picked and committed to throughout (no drift)                          | C        |     |
| 2 | At least three subpopulation axes defined with reproducible bucket boundaries             | C        |     |
| 3 | Rationale given for each axis (business impact + signal availability + reproducibility)   |          |     |
| 4 | Axes-NOT-measured subsection names >=3 unmeasured axes with reasons                        | C        |     |
| 5 | Primary metric named with defence vs. at least two alternatives                           | C        |     |
| 6 | Secondary/calibration metrics named and defended                                          |          |     |
| 7 | Disaggregated table has nDCG, CI, n_queries, n_judged, delta — no blank cells             | C        |     |
| 8 | CI method named (bootstrap, percentile) with iteration count and seed pinned              | C        |     |
| 9 | Sample-size floor (`n_queries >= 100`) stated and enforced in the table                   |          |     |
|10 | Illustrative-numbers disclaimer present inline and in common mistakes                     | C        |     |
|11 | Gap analysis covers the worst >=3 slices with fix/accept/monitor + owner                   | C        |     |
|12 | Intersectional-slicing limitation explicitly discussed and defended                        |          |     |
|13 | Hugging Face YAML frontmatter present with model-index >=3 entries                        | C        |     |
|14 | YAML `verified: false` on self-reported entries                                           |          |     |
|15 | YAML slice entries correspond to rows in the Markdown table                                | C        |     |
|16 | Code snippet parses (ast.parse succeeds)                                                  | C        |     |
|17 | Snippet computes per-query metric then aggregates (resamples at the query level)          | C        |     |
|18 | Snippet dependencies pinned with versions                                                 |          |     |
|19 | Governance framing cites NIST AI RMF (MEASURE+MAP), OWASP ML Top 10, MITRE ATLAS          |          |     |
|20 | No invented sub-item numbers for governance frameworks; URLs listed in §6                 | C        |     |

Any critical row scoring 0 fails the exercise regardless of total.

## 5. Common mistakes

1. **Publishing only the overall number.** The entire point of the
   data-slices section is per-slice reporting. A card with a single
   nDCG@10 figure and no disaggregation does not have a data-slices
   section — it has a headline.
2. **Reporting a point estimate with no CI.** A bucket with 112 queries
   has a 95% CI roughly ±0.03 wide; without the CI, a 0.9-pp difference
   looks meaningful when it is noise. Every numeric cell gets a CI.
3. **Resampling at the query-doc level, not the query level.** Doc-level
   bootstrap under-estimates the variance because query-docs within a
   query are correlated. Resample the unit of independence (the query).
4. **Over-claiming on illustrative numbers.** The whole exercise rests
   on using plausible placeholders. If the solution does not mark them
   as illustrative, a reader may quote them externally. Say "illustrative"
   in prose, in a callout, and in the common-mistakes list.
5. **Treating "no demographic slices" as "fair".** Not measuring is not
   a result. The honest answer is "we do not slice on demographic axes
   because we do not store them" — which is both an honest absence and
   an input to a separate conversation about whether we should.
6. **Hiding bad slices as "low-n".** The `low-n — not reported` tag is
   for cells that are genuinely too noisy to publish, not for cells
   whose numbers look embarrassing. Apply the floor rule mechanically
   (`n_queries >= 100`) and document it.
7. **Pasting numbers from the previous version's card.** Numbers must
   regenerate from the current snippet on the current eval set.
   Validator step §3.4 (overall anchor matches YAML) is the first thing
   that catches copy-paste drift.
8. **Publishing the full intersectional cube.** Tempting (it looks
   thorough), but most cells will fall below the low-n floor and the
   table becomes unreadable. Keep the published table marginal; put
   intersections on the monitoring dashboard.
9. **Citing NIST AI RMF sub-item numbers from memory.** The function
   names (GOVERN, MAP, MEASURE, MANAGE) are stable; the sub-item codes
   are not, and inventing them makes the card look authoritative on a
   thing it is not. Cite by function name and link the framework URL.
10. **Omitting calibration when a probability head feeds downstream.**
    If the re-ranker emits a click probability that another system
    consumes, nDCG alone can be perfect while calibration is broken.
    Report ECE per slice when a probability head exists.
11. **Naming individual humans as owners.** Owners should be roles or
    teams; individual names become stale, leak personnel information,
    and make the card feel like an internal ticket.
12. **Treating the card as write-once.** The card must be regenerated
    every time the model or the eval set changes. The snippet is the
    regenerator; the validator (§3) is the gate. Without the loop, the
    card decays into decoration.

## 6. References

Primary sources (verified):

- Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L.,
  Hutchinson, B., Spitzer, E., Raji, I. D., Gebru, T. (2019). *Model
  Cards for Model Reporting.* FAT* 2019.
  <https://arxiv.org/abs/1810.03677>
- Hugging Face. *Model Cards documentation.*
  <https://huggingface.co/docs/hub/model-cards>
- NIST. *AI Risk Management Framework (AI RMF 1.0).*
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP. *Machine Learning Security Top 10.*
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE. *ATLAS — Adversarial Threat Landscape for Artificial
  Intelligence Systems.* <https://atlas.mitre.org/>

Academic and methodological:

- Efron, B. (1979). *Bootstrap Methods: Another Look at the Jackknife.*
  Annals of Statistics, 7(1), 1–26. The original bootstrap paper;
  justifies percentile CIs used in §2.3.
- Buolamwini, J. & Gebru, T. (2018). *Gender Shades: Intersectional
  Accuracy Disparities in Commercial Gender Classification.*
  FAT* 2018. The canonical example of why *intersectional*
  disaggregation matters and why marginal slices under-report gaps.
- Barocas, S., Hardt, M., Narayanan, A. *Fairness and Machine Learning:
  Limitations and Opportunities.* The textbook treatment of why equal
  per-slice performance is not the same as fairness. Plain-name citation.
- NIST Special Publication 1270. *Towards a Standard for Identifying
  and Managing Bias in Artificial Intelligence.*
  <!-- research-note: confirm the SP-1270 guidance on disaggregated
  evaluation and the exact section to cite. -->
- Järvelin, K. & Kekäläinen, J. (2002). *Cumulated gain-based
  evaluation of IR techniques.* ACM TOIS. Justifies the choice of nDCG.

Local anchors (other modules this card depends on):

- `../../mod-111-data-quality-monitoring-and-regression/exercise-01-dataset-monitoring-contract-for-one-pipeline/SOLUTION.md`
  — the `features.search_reranker_v3` feature table and its monitoring
  contract.
- `../../mod-109-dataset-versioning-and-provenance/` — dataset
  versioning strategy for the eval split used in Artifact C.
- `../../mod-110-pii-and-privacy-risk-for-training-data/` — the PII
  posture that explains why user-level demographics are an honest
  absence rather than an oversight.
- `../exercise-02-dataset-card-across-three-formats/` — the dataset
  card that this model card's `datasets:` field in the YAML points to.
- `../exercise-03-eu-ai-act-annex-xi-training-data-summary-draft/` — the
  GPAI framing for `acme-text-base-7b`, the alternative model we
  considered and did not pick.
