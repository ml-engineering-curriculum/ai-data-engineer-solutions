# Exercise 02 — Dataset Card Across Three Formats (Solution)

## 1. Solution overview

This is a **design + implementation** exercise. The learner authors
exactly one dataset card for exactly one concrete dataset and renders
the same card into three reader-facing formats:

1. A Hugging Face Hub `README.md` dataset card with full Hub YAML
   frontmatter. Audience: practitioners who found the dataset by
   searching the Hub and want to download it in under five minutes.
2. A Google Data Cards long-form card (Pushkarna, Zaldivar, Kjartansson
   2022, "Data Cards: Purposeful and Transparent Dataset Documentation
   for Responsible AI"). Audience: internal reviewers, legal counsel,
   responsible-AI / privacy roles, and executives who need the risk
   story in prose.
3. An MLCommons Croissant JSON-LD record. Audience: downstream
   machine readers — data catalogs, Hub search, ML frameworks that
   mount the dataset directly, and governance tools that index
   provenance.

The point of the exercise is that **the same dataset facts have to
appear in three different formats**, each tuned to a different
audience, and some facts get lost in translation. The learner has to
reason about which format carries which fact losslessly, which
degrades to prose, and which does not travel at all. The deliverable
is not a tool or a framework — it is three artifacts that together
document one dataset, plus a short cross-format diff explaining
what each format preserves.

Deliverables (verbatim from the exercise contract):

- `README.md` — a Hugging Face Hub dataset card with the Hub's YAML
  frontmatter at the top and the required narrative sections below.
- `data_card.md` — a Google Data Cards long-form card using the
  Pushkarna et al. 2022 structured prompt-answer-artifact pattern.
- `croissant.json` — an MLCommons Croissant JSON-LD record that
  validates under `mlcroissant` (the one artifact that is
  machine-checked).
- `cross_format_diff.md` — a short table showing which facts each
  format preserves and which it loses, plus a 3-4 paragraph design
  rationale defending the hardest choices.

Chosen dataset. We reuse `features.search_reranker_v3` from mod-111
ex-01 — a tabular feature-store table for a search re-ranker — so
reviewers can cross-reference the sibling exercise and see the same
dataset documented operationally (the monitoring contract) and
communicatively (these three cards). Reusing one dataset across the
module is deliberate: it is the same reality seen through different
lenses, and the lesson is that each lens drops something the others
keep.

Governance framing. Dataset cards are the first-line documentation
control for a training dataset. The **NIST AI Risk Management
Framework** anchors in scope are the **MAP** function (characterize
the data, its provenance, and intended use in context) and the
**GOVERN** function (documentation discipline across the lifecycle,
including explicit owners and review cadence). The **OWASP Machine
Learning Security Top 10** anchors are **ML02 Data Poisoning Attack**
(the Croissant distribution's `sha256` hashes and the pinned
per-column dtypes are what a reviewer compares against when a
training set appears to be shifting under them) and **ML06 AI Supply
Chain Attacks** (the provenance, license, and curator identity fields
are the supply-chain receipts for the dataset itself). The **MITRE
ATLAS** framing is that a dataset card is an **inspectable artifact**
for both defenders (did the training data have a license to be used
at all?) and red-teamers (what intents, locales, and biases does the
card admit — and therefore where should an adversary probe?). We
cite only the top-level function names; URLs are in §6. We do not
invent sub-item numbers.

What this exercise is not. It is not an attempt to design a
universal dataset-documentation schema. The Hugging Face Hub card,
the Google Data Cards format, and Croissant JSON-LD already exist as
widely-adopted formats serving distinct audiences. The learner's job
is to render **the same facts** into each, not to invent a fourth.

## 2. Worked answer or implementation

### 2.1 Chosen dataset — one-paragraph pin

- **Name:** `features.search_reranker_v3`
- **Shape:** 18 numeric features + 4 categorical features + 1 label
  (`clicked`) + partition key (`event_date`) + timestamp
  (`event_ts_utc`) + two keys (`query_id`, `candidate_doc_id`).
- **Volume:** ~2.4 billion rows across the last 90 days, stored as
  Parquet in `s3://ml-featurestore-prod/search_reranker_v3/`, roughly
  480 GiB compressed.
- **Grain:** one row per `(query_id, candidate_doc_id)` pair for a
  served candidate list.
- **Version in scope for this card:** `2026.11.24.1` (CalVer),
  pinned to the mod-111 ex-01 monitoring contract.
- **License:** internal, non-redistributable. (For a public dataset
  card you would substitute a real OSI license id.)

We will **render this same dataset three times.** Where the formats
disagree on vocabulary, we note it. Where they disagree on scope, we
pick the broader scope (Croissant) as canonical and degrade the
others.

### 2.2 Project layout

```
exercise-02/
├── README.md                 # Hugging Face Hub dataset card
├── data_card.md              # Google Data Cards long-form card
├── croissant.json            # MLCommons Croissant JSON-LD record
├── cross_format_diff.md      # Fact-preservation table + rationale
└── validation/
    ├── validate_croissant.sh # one-line wrapper around `mlcroissant validate`
    └── lint_hub_yaml.py      # 30-line YAML frontmatter lint
```

### 2.3 Artifact A — Hugging Face Hub `README.md`

The Hub expects a YAML frontmatter block at the very top of the file,
delimited by `---`, followed by the dataset-card narrative. Hub
tooling parses the YAML; the narrative is Markdown. The frontmatter
drives Hub search filters (task, language, size, license), so missing
keys silently cost discoverability. Narrative section order is
conventional, not strictly enforced, but the convention is strong
enough that we follow it.

```markdown
---
license: other
license_name: internal-non-redistributable
license_link: LICENSE
pretty_name: "Search Re-ranker v3 Features"
task_categories:
  - tabular-classification
  - feature-extraction
language:
  - en
  - de
  - fr
  - es
  - ja
size_categories:
  - 1B<n<10B
tags:
  - search
  - ranking
  - tabular
  - feature-store
  - internal
  - mod-112-ex-02
annotations_creators:
  - machine-generated
language_creators:
  - found
source_datasets:
  - original
multilinguality:
  - multilingual
pretty_name_slug: search-reranker-v3
configs:
  - config_name: default
    data_files:
      - split: train
        path: data/event_date=2026-08-*/**.parquet
      - split: validation
        path: data/event_date=2026-10-*/**.parquet
      - split: test
        path: data/event_date=2026-11-*/**.parquet
dataset_info:
  features:
    - name: query_id
      dtype: string
    - name: candidate_doc_id
      dtype: string
    - name: event_date
      dtype: date32
    - name: event_ts_utc
      dtype: timestamp[us, tz=UTC]
    - name: clicked
      dtype: bool
    - name: feat_bm25
      dtype: float32
    - name: feat_dense_cosine
      dtype: float32
    - name: feat_title_len
      dtype: float32
    - name: feat_body_len
      dtype: float32
    - name: feat_click_ctr_30d
      dtype: float32
    - name: feat_query_log_freq
      dtype: float32
    - name: feat_candidate_age_days
      dtype: float32
    - name: feat_candidate_quality
      dtype: float32
    - name: feat_user_locale_rank
      dtype: float32
    - name: feat_session_pos
      dtype: float32
    - name: feat_dwell_pred
      dtype: float32
    - name: feat_price_log
      dtype: float32
    - name: feat_stock_flag
      dtype: float32
    - name: feat_domain_authority
      dtype: float32
    - name: feat_image_present
      dtype: float32
    - name: feat_video_present
      dtype: float32
    - name: feat_lang_match
      dtype: float32
    - name: feat_prev_click_indicator
      dtype: float32
    - name: cat_locale
      dtype:
        class_label:
          names:
            "0": en-US
            "1": en-GB
            "2": de-DE
            "3": fr-FR
            "4": es-ES
            "5": ja-JP
            "6": other
    - name: cat_device
      dtype:
        class_label:
          names:
            "0": desktop
            "1": mobile
            "2": tablet
            "3": other
    - name: cat_vertical
      dtype:
        class_label:
          names:
            "0": web
            "1": commerce
            "2": news
            "3": video
            "4": other
    - name: cat_query_class
      dtype:
        class_label:
          names:
            "0": navigational
            "1": transactional
            "2": informational
            "3": other
  splits:
    - name: train
      num_bytes: 322000000000
      num_examples: 1600000000
    - name: validation
      num_bytes: 80000000000
      num_examples: 400000000
    - name: test
      num_bytes: 80000000000
      num_examples: 400000000
  download_size: 482000000000
  dataset_size: 482000000000
---

# Dataset Card for Search Re-ranker v3 Features

<!-- research-note: the HF Hub spec allows several equivalent tag
     shapes for an internal non-OSI license. We use `license: other`
     plus `license_name` + `license_link`; verify against the current
     https://huggingface.co/docs/hub/datasets-cards schema before
     publishing. -->

## Dataset Summary

`features.search_reranker_v3` is the training-time feature-store table
consumed by the inference-time retrieval re-ranker in our product
search stack. Each row is one `(query_id, candidate_doc_id)` pair
that appeared in a served candidate list over a 90-day rolling
window. Columns are 18 numeric features (BM25 and dense cosine
scores, document-quality signals, session positional signals,
commerce features), 4 categorical features (locale, device,
vertical, query class), a binary click label, and keys and
timestamps. The dataset is read by the offline training job and by
the online feature-store sync that publishes features to Redis.

This card documents version `2026.11.24.1`. The corresponding
monitoring contract is maintained as a separate artifact
(`dataset_monitoring_contract.yaml`) referenced in §Additional
Information.

## Supported Tasks and Leaderboards

- **tabular-classification** — binary classification of
  `clicked ∈ {0, 1}` from the 22 feature columns. This is the
  primary task.
- **feature-extraction** — the numeric and categorical features
  are reused as inputs to session-level re-ranker and
  diversification models.

There is no public leaderboard; the dataset is internal.

## Languages

The dataset is multilingual, with locale-tagged rows across five
primary market locales (`en-US`, `en-GB`, `de-DE`, `fr-FR`,
`es-ES`, `ja-JP`) plus an `other` bucket. Locale is recorded
on the row as `cat_locale`; it is derived from the IP-geolocated
market of the request and not from language detection of the
query text.

## Dataset Structure

### Data Instances

A single example:

```json
{
  "query_id": "q_01HMX7V7Z5E8Y3",
  "candidate_doc_id": "d_01HMX7V8KRZ1",
  "event_date": "2026-11-24",
  "event_ts_utc": "2026-11-24T14:03:21.517000Z",
  "clicked": true,
  "feat_bm25": 12.47,
  "feat_dense_cosine": 0.813,
  "feat_title_len": 42.0,
  "feat_body_len": 1183.0,
  "feat_click_ctr_30d": 0.071,
  "feat_query_log_freq": 6.21,
  "feat_candidate_age_days": 19.0,
  "feat_candidate_quality": 0.78,
  "feat_user_locale_rank": 3.0,
  "feat_session_pos": 2.0,
  "feat_dwell_pred": 14.8,
  "feat_price_log": 3.91,
  "feat_stock_flag": 1.0,
  "feat_domain_authority": 0.62,
  "feat_image_present": 1.0,
  "feat_video_present": 0.0,
  "feat_lang_match": 1.0,
  "feat_prev_click_indicator": 0.0,
  "cat_locale": "en-US",
  "cat_device": "mobile",
  "cat_vertical": "commerce",
  "cat_query_class": "transactional"
}
```

### Data Fields

See the `dataset_info.features` block in the YAML frontmatter for
the typed schema. Narrative notes:

- `query_id`, `candidate_doc_id` — opaque ULIDs. Not joinable back
  to raw query text or user identifiers in this release (see
  Personal and Sensitive Information below).
- `event_date`, `event_ts_utc` — the event is the moment the
  candidate list was served, not the click moment.
- `clicked` — `true` if any click on the candidate landed within the
  session's 30-minute window.
- `feat_*` — numeric features, all `float32`. Cold-start candidates
  may have nulls in `feat_click_ctr_30d` and `feat_dwell_pred`; non-
  commerce verticals legitimately null out `feat_price_log` and
  `feat_stock_flag`. See the monitoring contract for the pinned
  per-column null-rate caps.
- `cat_*` — categorical features. Values outside the enumerated set
  are mapped to `other` before storage, so the column is strictly
  bounded.

### Data Splits

- **train** — `event_date` in August 2026, ~1.6B rows.
- **validation** — `event_date` in October 2026, ~400M rows.
- **test** — `event_date` in November 2026, ~400M rows.

Splits are **temporal**, not random. Random splits on this data
leak the next-day click distribution across split boundaries and
inflate offline AUC by roughly 1.5 points (see internal regression
note referenced in the monitoring contract). Rows are not
deduplicated across splits because the `(query_id, candidate_doc_id)`
pair is naturally time-scoped.

## Dataset Creation

### Curation Rationale

The re-ranker needs a training table that reflects what the online
candidate generator actually surfaces, not what a counterfactual
candidate universe might surface. We therefore log the features
inline with the serving path (Kafka topic
`serving.search.candidates.v3`) and materialize into Parquet
nightly. The alternative — reconstructing features offline from
raw logs — produces training-serving skew that is known to cost ~2
points of offline recall@10 on this stack.

### Source Data

#### Initial Data Collection and Normalization

Rows originate from the live production search path. The feature
build pipeline (`pipelines.search_feature_build`, nightly Airflow
DAG) joins candidate-list log events against the feature-store
snapshot pinned at request time. Categorical values outside the
enumerated set are collapsed to `other`. Numeric features are
clipped to the pinned ranges documented in the monitoring contract
and winsorized at 0.1 / 99.9 percentiles before storage.

#### Who are the source language producers?

Query text is produced by end users of the search product. **No
raw query text is present in this dataset.** Only the derived
`cat_query_class` (navigational / transactional / informational /
other) is retained, and it is inferred by a classifier whose version
is pinned in the monitoring contract.

### Annotations

#### Annotation Process

The label `clicked` is **implicit feedback**, not human annotation.
It is derived automatically from click-stream events and does not
reflect ground-truth relevance. The `cat_*` columns are
machine-generated features, not annotations.

No role annotated this dataset. There is no inter-annotator
agreement to report.

#### Who are the annotators?

Not applicable — no human annotation step.

### Personal and Sensitive Information

- No raw query text is present.
- No user identifiers are present. `query_id` is an opaque per-
  request ULID; it does not join to a user profile from this
  table alone.
- IP addresses and precise geolocation are **not** present; only
  the market locale derived from IP is kept, and only as a
  bounded `cat_locale` enumeration.
- Timestamps are retained at microsecond granularity and are a
  known quasi-identifier in combination with other side data. The
  retention policy (90-day rolling window) is intended to bound
  this risk.
- No row-level sensitive attributes (race, religion, health,
  sexual orientation, political affiliation) are collected or
  inferred.

The PII handling policy is described in prose here; the
**Google Data Cards** render below expresses it in structured
form, and the Croissant render does not express it (Croissant's
current vocabulary does not have a first-class PII field — see
`cross_format_diff.md`).

## Considerations for Using the Data

### Social Impact of Dataset

Models trained on this table drive what search results users see.
Mis-ranked results change which documents get traffic, which
affects publisher revenue and user information access. The dataset
is therefore an upstream input to a system with distributional
effects on the information ecosystem.

### Discussion of Biases

- **Position bias.** `feat_session_pos` records the position at
  which the candidate was shown. Clicks are strongly correlated
  with position, and models trained naively on `clicked` will
  overweight whatever the current online ranker already surfaces.
  Offline training is expected to apply position-bias correction
  (inverse-propensity weighting on `feat_session_pos`); this card
  does not enforce it.
- **Locale imbalance.** `en-US` is ~62% of rows; `ja-JP` is ~4%.
  Models trained without stratified sampling will underperform on
  smaller locales. The monitoring contract tracks subpopulation
  performance along `cat_locale` for this reason.
- **Vertical imbalance.** `web` is ~71% of rows; `video` is ~3%.
  Same mitigation.
- **Survivorship on candidate_doc_id.** Only candidates the online
  generator surfaced appear in the table. Candidates never
  surfaced (because of upstream filters, legal takedowns, or low
  quality scores) are not representable from this data.

### Other Known Limitations

- `feat_click_ctr_30d` and `feat_dwell_pred` have elevated null
  rates for cold-start candidates (candidates younger than 30
  days). This is documented and allowed by the monitoring
  contract; trainers should not impute silently.
- Rows are not deduplicated across `(query_id, candidate_doc_id)`
  within a single serving event.
- The `clicked` signal is 30-minute-windowed; longer-term
  engagement is not captured here.

## Additional Information

### Dataset Curators

Owned by the search-ranking team (role contact:
`search-ranking@example.internal`, Slack `#search-ranking-oncall`).
Contributions are managed via internal code review on the feature
pipeline repository. No individual role names are published in this
card by design; role-based contacts are the only supported path.

### Licensing Information

Internal use only; not redistributable. For the purposes of the Hub
card format, this is encoded as `license: other` with a pointer to
the internal `LICENSE` file. A public release of a similarly-shaped
dataset would substitute an OSI-approved license id.

### Citation Information

<!-- research-note: if this dataset is later released externally
     with a paper, add a BibTeX entry here. -->

```bibtex
@misc{search_reranker_v3_2026,
  title        = {Search Re-ranker v3 Features},
  author       = {{Search Ranking Team}},
  year         = {2026},
  howpublished = {Internal dataset, version 2026.11.24.1},
  note         = {Monitoring contract: dataset\_monitoring\_contract.yaml}
}
```

### Contributions

Contributions follow internal change-management on the feature
pipeline repository. External contributions are not accepted for
this internal release.

### Related Artifacts

- Monitoring contract: `dataset_monitoring_contract.yaml` (mod-111
  ex-01).
- Google Data Cards long-form: `data_card.md` (this exercise).
- Machine-readable Croissant record: `croissant.json` (this
  exercise).
```

Notes on the Hub card.

- The YAML frontmatter is the discoverability layer. Missing
  `size_categories` or `task_categories` silently removes the
  dataset from the Hub's faceted search; missing `license` is the
  single most common Hub-card failure.
- `dataset_info.features` is the Hub's typed schema. It is
  **lossless at the column-dtype level**, which is why the
  cross-format diff in §2.6 marks column dtypes as preserved in
  both HF and Croissant and lost in Google Data Cards.
- The narrative sections are conventional — Hub tooling does not
  validate them — but matching the HF template order makes the
  card legible to readers who have read any other Hub card.

### 2.4 Artifact B — Google Data Cards long-form `data_card.md`

The Google Data Cards format (Pushkarna, Zaldivar, Kjartansson 2022)
is prose-first and reader-first. The canonical rendering uses a
three-column pattern per prompt: **question → answer → links to
artifacts / code / repros**. In Markdown we render the three columns
as nested bullet lists: the prompt is a heading, the answer is a
paragraph underneath, and the links are a nested "Supporting
artifacts" bullet. The structured block feel matters because the
Data Cards Playbook uses the three-column layout as a visible
signal that answers are evidenced, not asserted.

```markdown
# Data Card — Search Re-ranker v3 Features

**Publisher:** Search Ranking team (role contact:
`search-ranking@example.internal`)
**Dataset version:** `2026.11.24.1`
**Card version:** 1.0.0 (2026-11-24)
**Scope of this card:** training-time feature-store table for the
inference-time retrieval re-ranker.

---

## 1. Dataset Description

- **Prompt:** What is in this dataset, at what grain, and how large
  is it?
- **Answer:** `features.search_reranker_v3` is a tabular feature-
  store table. Each row is one `(query_id, candidate_doc_id)` pair
  for a candidate that appeared in a served search candidate list.
  The table contains 22 feature columns (18 numeric, 4
  categorical), a binary label (`clicked`), two opaque keys, and
  two timestamp / partition columns. Volume is ~2.4 billion rows
  over a 90-day rolling window, roughly 482 GB on disk as Parquet.
  The grain is one row per candidate, not one row per query.
- **Supporting artifacts:**
  - Schema: `README.md` YAML frontmatter `dataset_info.features`.
  - Machine-readable: `croissant.json` `recordSet[0].field[*]`.
  - Monitoring contract: `dataset_monitoring_contract.yaml` §1.

## 2. Example of a Data Point

- **Prompt:** Show one representative row.
- **Answer:** See `README.md` §"Data Instances" for a worked
  example. Narrative highlights: `clicked = true`,
  `cat_locale = "en-US"`, `cat_vertical = "commerce"`,
  `cat_query_class = "transactional"`, and `feat_bm25 = 12.47`
  (a typical mid-range BM25 for this stack).
- **Supporting artifacts:**
  - `README.md` §"Data Instances".
  - `croissant.json` — same row shape derivable from the typed
    `recordSet`.

## 3. Motivations and Intended Uses

- **Prompt:** Why was this dataset built, and what is it meant to
  be used for?
- **Answer:** The dataset exists to train the inference-time
  retrieval re-ranker. The alternative — reconstructing features
  offline from raw logs — produces training-serving skew that has
  historically cost ~2 points of offline recall@10 on this stack,
  so logging features inline with serving and materializing
  nightly is the operational decision. Intended uses: offline
  training of the re-ranker, offline evaluation of candidate
  re-ranker variants, offline fairness and subpopulation audits.
- **Not-intended uses:**
  - Not a general web-ranking dataset; the candidate universe is
    filtered by upstream generators.
  - Not a user-behavior dataset; no user identifier is attached.
  - Not a query-understanding dataset; raw query text is **not**
    present.
  - Not a relevance dataset; `clicked` is implicit feedback and
    does not equal ground-truth relevance.
- **Supporting artifacts:**
  - Monitoring contract §1 (owners, downstream consumers).
  - `README.md` §"Supported Tasks and Leaderboards".

## 4. Access, Retention, and Wipeout

- **Prompt:** Who can read this dataset, for how long, and what
  happens when it is decommissioned?
- **Answer:**
  - **Access.** Read-only access is gated by the
    `search-ranking-training` IAM role in the feature-store
    account. Access requests are reviewed by the role contact
    above and are logged. No external access is granted.
  - **Retention.** Rows older than 90 days are deleted from the
    materialized Parquet by a nightly retention job. Raw Kafka
    events are retained for 7 days and then deleted by the broker
    TTL.
  - **Wipeout.** Dataset-level wipeout is supported: the
    retention job can be forced to drop all partitions under a
    named `event_date` range. Individual-row wipeout is not
    supported because no user identifier is retained; the
    dataset is already de-identified at materialization time.
- **Supporting artifacts:**
  - IAM policy: `infra/iam/search-ranking-training.rego`.
  - Retention job: `jobs/retention_feature_store.py`.

## 5. Provenance

- **Prompt:** Where did this dataset come from, and how is its
  chain of custody evidenced?
- **Answer:** The dataset is produced by a single pipeline —
  `pipelines.search_feature_build` — pinned to a specific git
  commit (`5f3a9c71d2e8`) and container image digest
  (see monitoring contract §2). That pipeline consumes
  `serving.search.candidates.v3` Kafka events and the pinned
  feature-store snapshot. The data version (`2026.11.24.1`) is a
  CalVer tag emitted by the pipeline and asserted by the
  monitoring contract on every downstream read. Every Parquet
  file in the distribution has a SHA-256 hash recorded in the
  manifest and in the Croissant `distribution` block.
- **Supporting artifacts:**
  - Pipeline repository (commit `5f3a9c71d2e8`).
  - Monitoring contract §2 (lineage pins).
  - `croissant.json` — `distribution[*].sha256`.

## 6. Human and Automated Labeling

- **Prompt:** How are the labels and categorical features
  produced? Which role did what?
- **Answer:**
  - `clicked` is **implicit feedback** derived automatically from
    client click-stream events joined to the candidate list
    within a 30-minute session window. No role annotated it.
  - `cat_locale` is derived from the market-locale of the request
    (itself derived from IP geolocation upstream of this
    pipeline). No role annotated it.
  - `cat_device` is derived from the request user-agent header.
    No role annotated it.
  - `cat_vertical` and `cat_query_class` are produced by two
    pinned classifiers upstream (`vertical-classifier@3.4.1`,
    `query-class-classifier@2.1.7`). The classifier identities and
    versions are recorded in the monitoring contract.
  - There is no human labeling step. There is therefore no inter-
    annotator agreement to report.
- **Supporting artifacts:**
  - Monitoring contract §2 (upstream classifier pins).
  - `README.md` §"Annotations".

## 7. Validation Types

- **Prompt:** What is validated about this dataset, by whom, and
  how?
- **Answer:**
  - **Schema validation** — every pipeline run validates against
    the pinned Great Expectations suite and emits a pass/fail
    status. Any schema breach fails the run.
  - **Freshness** — the run has to land its daily partition by
    06:00 UTC or the monitoring contract pages the owner.
  - **Distributional drift** — PSI and KS-statistic drift SLIs
    are computed per feature against the pinned reference window
    (`ref-2026Q3`). Breaches route to the owner Slack.
  - **Subpopulation drift** — per-`cat_locale` and per-
    `cat_vertical` drift is computed and routed on the same
    channel.
  - **Hash parity** — the distribution SHA-256 hashes in this
    Data Card, the Hub card's `dataset_info`, and
    `croissant.json` are required to match. Mismatch is a hard
    failure in the publish job.
- **Supporting artifacts:**
  - Monitoring contract §3-§5.
  - Exercise 02 of mod-111 (SLI compute harness).

## 8. Sampling Methods

- **Prompt:** Is the dataset sampled, and if so how?
- **Answer:** No downsampling is applied at materialization time;
  all served candidates are logged. However, the candidate
  **generator upstream** of this pipeline is not uniform: it is
  tuned to recall in the head of the query distribution and will
  under-surface tail-query candidates relative to a uniform
  baseline. This is a **survivorship** property of the dataset,
  not a sampling scheme, and it is described explicitly in §9.
- **Supporting artifacts:**
  - Candidate generator spec: internal doc (link omitted).

## 9. Known Biases and Risks

- **Prompt:** What biases are known or suspected to be present in
  this dataset?
- **Answer:**
  - **Position bias.** `clicked` is strongly correlated with
    `feat_session_pos`. Trainers must apply inverse-propensity
    weighting; this card does not enforce it, only documents it.
  - **Locale imbalance.** `en-US` is ~62% of rows; `ja-JP` is
    ~4%. Models without stratified sampling will underperform on
    smaller locales.
  - **Vertical imbalance.** `web` is ~71% of rows; `video` is
    ~3%. Same mitigation.
  - **Candidate survivorship.** Only candidates the online
    generator surfaced appear. Never-surfaced candidates are not
    representable.
  - **Implicit-feedback bias.** `clicked` is an action signal,
    not a relevance signal. Trained models optimize for
    click-through, not for user satisfaction.
- **Supporting artifacts:**
  - Monitoring contract §4 (subpopulation SLIs).
  - Internal fairness audit (quarterly).

## 10. Known Correlations

- **Prompt:** Which features are known to co-vary, and which
  spurious correlations should trainers beware of?
- **Answer:**
  - `feat_bm25` and `feat_dense_cosine` are positively correlated
    (ρ ≈ 0.55 on the reference window). Trainers using regularized
    linear models should not interpret coefficients independently.
  - `feat_session_pos` is correlated with `clicked` by
    construction (position bias, above).
  - `feat_price_log` is strongly correlated with `cat_vertical =
    commerce`; the feature is legitimately null outside commerce.
  - `cat_device = mobile` is correlated with shorter
    `feat_session_pos` distributions (mobile shows fewer results
    per screen).
- **Supporting artifacts:**
  - Correlation matrix snapshot in the reference-window artifact.

## 11. Access, PII, and Sensitive Attributes

- **Prompt:** What personal or sensitive information, if any, is
  present?
- **Answer:**
  - **Direct identifiers (name, email, phone, address, user id)** —
    not present.
  - **Raw query text** — not present. Only derived
    `cat_query_class` is retained.
  - **IP address, precise geolocation** — not present. Only market
    `cat_locale` is retained.
  - **Row-level sensitive attributes** (race, religion, health,
    sexual orientation, political affiliation) — not collected
    and not inferred.
  - **Quasi-identifiers** — microsecond-precision timestamps are a
    quasi-identifier in combination with external data. Mitigated
    by the 90-day retention window.
- **Supporting artifacts:**
  - Monitoring contract §1 (owner, access).
  - `README.md` §"Personal and Sensitive Information".

## 12. Maintenance and Versioning

- **Prompt:** How is this dataset maintained, and how will users
  of this card know when it has changed?
- **Answer:**
  - **Cadence.** Nightly materialization; CalVer version bump
    daily.
  - **Breaking changes.** Column additions, drops, or dtype
    changes require bumping the dataset major version
    (currently `v3`) and publishing a new card. Non-breaking
    changes (added enumerated categorical values, widened null-
    rate caps) bump the CalVer patch number only.
  - **Deprecation.** `v2` is deprecated as of 2026-11-01 and will
    be deleted on 2027-02-01. `v3` is the current major.
  - **Card review.** This card is reviewed by the role contact at
    every major-version bump. Minor-version data changes do not
    trigger a card review but do trigger a change note appended
    to the end of this file.
- **Supporting artifacts:**
  - Monitoring contract §6 (rollout schedule).
  - `CHANGELOG.md` (dataset-level).

---

### Change notes

- `2026.11.24.1` — `cat_locale` added `ja-JP` as a first-class
  value (was `other` in prior CalVer tags).
- `2026.10.05.1` — `feat_prev_click_indicator` added. Marked
  `research-note` null-rate cap pending one full week of
  observation.
```

Notes on the Data Cards card.

- The three-column pattern (prompt → answer → links to artifacts)
  is what makes Data Cards specifically Data Cards rather than
  "any prose dataset documentation." The links column is the
  antidote to unevidenced prose: every claim in the answer is
  supposed to point to an artifact, a monitoring SLI, or a repro.
- **PII handling is first-class** in this format (§11), unlike
  the Hub card (prose only) and Croissant (not a first-class
  field). The Data Cards format is the format you give to legal
  and privacy roles for exactly this reason.
- Per-column dtypes **do not appear** in this card. They are
  linked out to the Hub YAML frontmatter. This is intentional:
  Data Cards is a reader-first format and a typed schema is a
  machine-first artifact.

### 2.5 Artifact C — MLCommons Croissant `croissant.json`

Croissant is a JSON-LD vocabulary layered on top of schema.org's
`sc:Dataset`. The MLCommons vocabulary (`cr:`) adds the pieces
schema.org's native `Dataset` lacks: typed `cr:Field`s, `cr:FileSet`
for sharded distributions, and `cr:Source` pointers that let a
reader rebuild a column's provenance from the record-set definition.
The learner should treat Croissant as **the most lossless of the
three formats for column-level facts** — which is why we treat the
Croissant render as canonical and degrade the others.

```json
{
  "@context": {
    "@vocab": "https://schema.org/",
    "sc": "https://schema.org/",
    "cr": "http://mlcommons.org/croissant/",
    "data": "http://mlcommons.org/croissant/data/",
    "dct": "http://purl.org/dc/terms/",
    "rai": "http://mlcommons.org/croissant/RAI/",
    "wd": "https://www.wikidata.org/wiki/",
    "citeAs": "cr:citeAs",
    "column": "cr:column",
    "conformsTo": "dct:conformsTo",
    "dataType": {
      "@id": "cr:dataType",
      "@type": "@vocab"
    },
    "extract": "cr:extract",
    "field": "cr:field",
    "fileProperty": "cr:fileProperty",
    "fileObject": "cr:fileObject",
    "fileSet": "cr:fileSet",
    "format": "cr:format",
    "includes": "cr:includes",
    "isLiveDataset": "cr:isLiveDataset",
    "jsonPath": "cr:jsonPath",
    "key": "cr:key",
    "md5": "cr:md5",
    "parentField": "cr:parentField",
    "path": "cr:path",
    "recordSet": "cr:recordSet",
    "references": "cr:references",
    "regex": "cr:regex",
    "repeated": "cr:repeated",
    "replace": "cr:replace",
    "separator": "cr:separator",
    "source": "cr:source",
    "subField": "cr:subField",
    "transform": "cr:transform"
  },
  "@type": "sc:Dataset",
  "@id": "https://datasets.example.internal/search-reranker-v3",
  "conformsTo": "http://mlcommons.org/croissant/1.0",
  "name": "search-reranker-v3",
  "alternateName": "Search Re-ranker v3 Features",
  "description": "Tabular feature-store table consumed by the inference-time retrieval re-ranker in our product search stack. One row per (query_id, candidate_doc_id) pair over a 90-day rolling window; 18 numeric features, 4 categorical features, and a binary click label.",
  "license": "https://datasets.example.internal/search-reranker-v3/LICENSE",
  "url": "https://datasets.example.internal/search-reranker-v3",
  "version": "2026.11.24.1",
  "datePublished": "2026-11-24",
  "dateModified": "2026-11-24",
  "citeAs": "Search Ranking Team. Search Re-ranker v3 Features. Internal dataset, version 2026.11.24.1. 2026.",
  "creator": {
    "@type": "sc:Organization",
    "name": "Search Ranking Team",
    "email": "search-ranking@example.internal",
    "url": "https://datasets.example.internal/owners/search-ranking"
  },
  "publisher": {
    "@type": "sc:Organization",
    "name": "Search Ranking Team"
  },
  "keywords": [
    "search",
    "ranking",
    "tabular",
    "feature-store",
    "internal"
  ],
  "inLanguage": ["en", "de", "fr", "es", "ja"],
  "isLiveDataset": false,
  "distribution": [
    {
      "@type": "cr:FileObject",
      "@id": "manifest.json",
      "name": "manifest.json",
      "description": "Manifest listing per-partition Parquet files, row counts, and per-file sha256 hashes for this version.",
      "contentUrl": "https://datasets.example.internal/search-reranker-v3/2026.11.24.1/manifest.json",
      "encodingFormat": "application/json",
      "sha256": "d7f9c0e8b4a3928170605040302010f0e0d0c0b0a0918171615141312111009"
    },
    {
      "@type": "cr:FileSet",
      "@id": "parquet-files",
      "name": "parquet-files",
      "description": "Partitioned Parquet files for this dataset version. Partition key: event_date. Files are nominally one per event_date per locale shard.",
      "containedIn": { "@id": "manifest.json" },
      "encodingFormat": "application/vnd.apache.parquet",
      "includes": "data/event_date=*/**.parquet"
    }
  ],
  "recordSet": [
    {
      "@type": "cr:RecordSet",
      "@id": "rows",
      "name": "rows",
      "description": "One record per (query_id, candidate_doc_id) served candidate.",
      "key": { "@id": "rows/query_id" },
      "field": [
        {
          "@type": "cr:Field",
          "@id": "rows/query_id",
          "name": "query_id",
          "description": "Opaque per-request ULID. Not joinable to user identity.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "query_id" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/candidate_doc_id",
          "name": "candidate_doc_id",
          "description": "Opaque document ULID.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "candidate_doc_id" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/event_date",
          "name": "event_date",
          "description": "Partition date, in UTC.",
          "dataType": "sc:Date",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "event_date" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/event_ts_utc",
          "name": "event_ts_utc",
          "description": "Microsecond-precision event timestamp in UTC.",
          "dataType": "sc:DateTime",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "event_ts_utc" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/clicked",
          "name": "clicked",
          "description": "Implicit-feedback click label within a 30-minute session window.",
          "dataType": "sc:Boolean",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "clicked" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_bm25",
          "name": "feat_bm25",
          "description": "BM25 score. float32.",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_bm25" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_dense_cosine",
          "name": "feat_dense_cosine",
          "description": "Dense-embedding cosine similarity. float32.",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_dense_cosine" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_title_len",
          "name": "feat_title_len",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_title_len" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_body_len",
          "name": "feat_body_len",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_body_len" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_click_ctr_30d",
          "name": "feat_click_ctr_30d",
          "description": "Candidate 30-day click-through rate. float32. Null for cold-start candidates.",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_click_ctr_30d" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_query_log_freq",
          "name": "feat_query_log_freq",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_query_log_freq" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_candidate_age_days",
          "name": "feat_candidate_age_days",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_candidate_age_days" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_candidate_quality",
          "name": "feat_candidate_quality",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_candidate_quality" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_user_locale_rank",
          "name": "feat_user_locale_rank",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_user_locale_rank" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_session_pos",
          "name": "feat_session_pos",
          "description": "Position at which the candidate was shown. Correlated with clicked by construction (position bias).",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_session_pos" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_dwell_pred",
          "name": "feat_dwell_pred",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_dwell_pred" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_price_log",
          "name": "feat_price_log",
          "description": "Log-scaled candidate price. Null outside commerce vertical.",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_price_log" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_stock_flag",
          "name": "feat_stock_flag",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_stock_flag" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_domain_authority",
          "name": "feat_domain_authority",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_domain_authority" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_image_present",
          "name": "feat_image_present",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_image_present" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_video_present",
          "name": "feat_video_present",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_video_present" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_lang_match",
          "name": "feat_lang_match",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_lang_match" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/feat_prev_click_indicator",
          "name": "feat_prev_click_indicator",
          "dataType": "sc:Float",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "feat_prev_click_indicator" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/cat_locale",
          "name": "cat_locale",
          "description": "Market-locale enumeration: en-US, en-GB, de-DE, fr-FR, es-ES, ja-JP, other.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "cat_locale" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/cat_device",
          "name": "cat_device",
          "description": "Device-class enumeration: desktop, mobile, tablet, other.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "cat_device" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/cat_vertical",
          "name": "cat_vertical",
          "description": "Vertical enumeration: web, commerce, news, video, other.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "cat_vertical" }
          }
        },
        {
          "@type": "cr:Field",
          "@id": "rows/cat_query_class",
          "name": "cat_query_class",
          "description": "Query-class enumeration: navigational, transactional, informational, other.",
          "dataType": "sc:Text",
          "source": {
            "fileSet": { "@id": "parquet-files" },
            "extract": { "column": "cat_query_class" }
          }
        }
      ]
    }
  ]
}
```

Notes on the Croissant record.

- The `conformsTo` key pins the Croissant spec version this record
  targets (`1.0`). Validators use this to pick the right schema.
- Every `cr:Field` sources from the `cr:FileSet` by `@id`. This is
  what makes the record **round-trippable to a data frame** without
  external context: a Croissant-aware reader can mount the Parquet
  files and project columns from the field definitions.
- The `manifest.json` `cr:FileObject` is included as a separate
  distribution entry because the per-partition SHA-256 hashes live
  inside it; we commit the manifest's own hash here at the top level.
  A reviewer only has to trust one hash (the manifest's) to then
  trust every partition.
- We intentionally do not express PII policy, intended uses, or
  known biases in Croissant. The Croissant 1.0 spec extensions
  include an RAI vocabulary (`rai:`) for some of this, but the
  fields are not universally supported and we declare them out of
  scope for this record. The Data Cards and Hub cards carry that
  content.

### 2.6 Cross-format diff — what each format preserves and loses

The three formats are not interchangeable. The table below tracks the
fact classes this dataset has and marks each cell as **lossless**
(the format carries the fact in a first-class typed slot),
**lossy** (the format carries it only as free prose), or **absent**
(the format does not carry it at all). This is the single most
important deliverable of the exercise; the three artifacts above
are the evidence.

| Fact class                          | Hugging Face Hub card      | Google Data Cards           | MLCommons Croissant       |
| ----------------------------------- | -------------------------- | --------------------------- | ------------------------- |
| Dataset name and version            | lossless (frontmatter)     | lossless (header block)     | lossless (`name`, `version`) |
| Per-column dtype                    | lossless (`dataset_info`)  | **lossy** (prose + link out)| lossless (`cr:Field.dataType`) |
| Per-column enum / class labels      | lossless (`class_label`)   | lossy (prose)               | lossy (prose; `cr:Field` does not enumerate) |
| Split definitions                   | lossless (`splits`)        | lossy (prose)               | lossy (modeled per-file)  |
| Row count / size                    | lossless (`num_examples`)  | lossy (prose)               | lossy (prose)             |
| File manifest + sha256 hashes       | **absent** (not in frontmatter) | lossy (link out)       | lossless (`distribution[*].sha256`) |
| License (OSI id)                    | lossless (`license`)       | lossy (prose)               | lossless (`license` URL)  |
| Non-OSI / internal license text     | lossy (`license: other`)   | lossy (prose)               | lossy (URL link only)     |
| Owner / curator role                | lossless (prose section)   | lossless (header block)     | lossless (`creator`)      |
| Intended uses                       | lossy (prose)              | **lossless** (first-class)  | absent                    |
| Not-intended uses                   | absent                     | **lossless** (first-class)  | absent                    |
| PII handling                        | lossy (prose)              | **lossless** (first-class)  | absent (rai: extension only) |
| Access / retention / wipeout policy | absent                     | **lossless** (first-class)  | absent                    |
| Known biases                        | lossy (prose)              | **lossless** (first-class)  | absent                    |
| Known correlations between features | lossy (prose)              | **lossless** (first-class)  | absent                    |
| Sampling methods                    | lossy (prose)              | **lossless** (first-class)  | absent                    |
| Upstream provenance pins (commit, image) | lossy (prose link)    | **lossless** (first-class)  | lossy (link out)          |
| Monitoring-contract link            | lossy (prose link)         | **lossless** (first-class)  | lossy (link out)          |
| Discoverability metadata (task, size, language) | lossless (frontmatter) | absent                 | lossless (`keywords`, `inLanguage`) |

The headline reading of this table:

- **Croissant** is the lossless carrier for **machine-consumable
  schema and provenance** (dtypes, hashes, file layout).
- **Google Data Cards** is the lossless carrier for **human-
  consumable risk narrative** (intended uses, PII, biases, access).
- **Hugging Face Hub** is the lossless carrier for
  **discoverability and quick-start** (task tag, size bucket,
  split definitions).

None of the three is the superset of the other two. That is
precisely why the exercise asks for all three.

### 2.7 Design rationale — defending the hardest choices

**Why Croissant over DCAT or schema.org alone.** DCAT 3 and
schema.org's native `sc:Dataset` are generic dataset-catalog
vocabularies; they can name a dataset, point at a distribution, and
tag a license, but they do not model per-column typed fields or
file-set-level provenance. Croissant layers exactly those additions
on top of schema.org — so a Croissant record **is** a schema.org
record a generic catalog reader can still consume, and a
Croissant-aware reader gets the ML-specific vocabulary on top. That
layering is why we chose Croissant as the machine-readable artifact:
we get the catalog interop for free and the ML-specific precision
on top. Writing only schema.org would lose the per-`cr:Field`
typing; writing only DCAT would lose the ML-framework
round-trippability.

**Why Data Cards long-form over compact.** The compact form of
Data Cards (one-page summary) is appropriate for an executive read
and for dataset-selection shopping carts; the long-form is
appropriate for the role that approves use of the dataset for a
specific training purpose. For this dataset, the audience that
matters is the approver — because the dataset carries implicit-
feedback labels, position bias, locale imbalance, and a
retention policy that a reviewer has to actively sign off on. The
long form forces each of those into a named section with a
prompt-answer-artifact triple, and that structure is what makes the
review tractable. The compact form would elide the parts that
matter most.

**How PII handling is expressed in each format.** PII is the single
clearest example of fact-class loss across the three formats. In the
Hub card, PII is prose in "Personal and Sensitive Information" — a
reviewer has to read it to find it, and a machine cannot index it.
In Data Cards, PII is a first-class prompt (§11) with a required
answer and linked artifacts — a reviewer finds it in the table of
contents, and the structure forces the author to list each
identifier class explicitly. In Croissant, PII is **absent** from
the core 1.0 vocabulary; the RAI extension includes some
PII-adjacent fields but is not universally supported. We therefore
treat Croissant as **not the source of truth for PII**, and we say
so explicitly in the Croissant record's prose fields and in the
Data Cards §11 block. A reviewer who starts in Croissant and tries
to answer "is there PII in this dataset?" will correctly end up in
the Data Cards file.

**Why we hash the manifest, not every partition, at the top level
of Croissant.** The dataset has thousands of per-partition Parquet
files. Hashing every one of them at the top level of the Croissant
record would make the record enormous and would couple the card's
version to the partition count, which changes every day as rolling-
window retention kicks in. Instead we hash the manifest file, and
the manifest in turn hashes each partition. A reviewer who wants to
verify a specific partition gets exactly one hop of indirection
(manifest), and the Croissant record's own hash footprint is
stable across the rolling window. This is also what the OWASP ML02
anchor asks for: a single trustworthy hash anchor, not a wall of
hashes that nobody can diff.

## 3. Validation steps

A review of the three artifacts should not depend on reading them.
The validation below is scripted where possible and manual-with-
checklist where not.

1. **Hub YAML frontmatter lints under PyYAML.** `python -c "import
   yaml, pathlib; yaml.safe_load(pathlib.Path('README.md').read_text()
   .split('---')[1])"` must return a dict without raising. This
   catches tab-vs-space errors and unterminated lists, which are the
   most common hand-edit failures.
2. **Hub frontmatter carries the required discoverability keys.**
   The lint script asserts presence of `license`, `task_categories`,
   `language`, `size_categories`, `pretty_name`, `tags`, and
   `dataset_info`. Missing any one of these silently drops the
   dataset from Hub faceted search.
3. **Croissant JSON-LD is syntactically valid JSON.** `python -c
   "import json; json.load(open('croissant.json'))"` must succeed.
4. **Croissant validates under `mlcroissant`.** `mlcroissant validate
   --jsonld croissant.json` must exit 0. This is the only
   machine-checked artifact in the exercise; failing this check is a
   hard failure.
5. **Every column named in Hub `dataset_info.features` also appears
   in Croissant `recordSet[0].field` under the same `name`.** This is
   the cross-format consistency check that catches "the learner
   edited the Hub frontmatter but forgot to update Croissant." The
   provided validation script
   (`validation/validate_croissant.sh`) extracts both column lists
   and diffs them.
6. **Google Data Cards sections 1 through 12 are all present.** The
   twelve named prompts are the minimum-viable Data Cards shape;
   missing any one of them means the card is not a Data Card, it is
   a notes file.
7. **Cross-format license consistency.** The `license` key in the
   Hub frontmatter, the "License" line in the Data Card header, and
   the `license` URL in the Croissant record must all refer to the
   same license. For this dataset they all point at the internal
   `LICENSE` file.
8. **Cross-format version consistency.** `dataset_info` download
   size, Data Cards `Dataset version`, and Croissant `version` must
   all be the same CalVer tag (`2026.11.24.1`).
9. **Cross-format record-count consistency.** The sum of
   `splits[*].num_examples` in the Hub card must equal the row count
   quoted in Data Cards §1. (Croissant 1.0 does not require a row
   count at the record-set level, so it is not part of this check.)
10. **Manifest SHA-256 matches the actual file.** The `sha256` in the
    Croissant `distribution` block for `manifest.json` must equal
    `sha256sum manifest.json`. Mismatch means someone edited the
    manifest and did not re-hash.
11. **No individual role names.** `grep -E '\b[A-Z][a-z]+ [A-Z][a-z]+'`
    over all three files must return zero matches that are not a
    documented dataset/pipeline name. (This is the "roles not
    humans" rule.)
12. **No invented URLs.** Every URL in all three files resolves to a
    reachable host or is an `example.internal` placeholder. External
    documentation URLs must be present only in §6 of this solution,
    not in the three artifacts.

A reviewer who passes all twelve of these on a fresh checkout has
enough evidence to sign off.

## 4. Rubric or review checklist

Scores: `0` = missing or wrong, `1` = present but incomplete, `2` =
present and defensible. Critical rows (marked **C**) must score `2`
to pass. Pass threshold: **35 / 40** with no critical row below `2`.

| # | Criterion                                                                 | 0 / 1 / 2 | Critical |
| - | ------------------------------------------------------------------------- | --------- | -------- |
| 1 | One concrete named dataset is chosen and used consistently across all three artifacts. |           | **C** |
| 2 | Hub YAML frontmatter contains `license`, `task_categories`, `language`, `size_categories`, `tags`, `pretty_name`. |           | **C** |
| 3 | Hub `dataset_info.features` types every column named in the narrative (dtype present). |           | **C** |
| 4 | Hub `dataset_info.splits` names at least two splits with `num_examples` and `num_bytes`. |           |       |
| 5 | Hub narrative includes Data Fields, Data Instances, Data Splits, Dataset Creation, Considerations, Additional Information. |           |       |
| 6 | Data Cards card has all twelve named prompts, each with an answer and a "Supporting artifacts" nested list. |           | **C** |
| 7 | Data Cards §11 (PII / sensitive attributes) explicitly enumerates each identifier class (present / absent). |           | **C** |
| 8 | Data Cards §4 (Access / Retention / Wipeout) names concrete mechanisms, not aspirations. |           |       |
| 9 | Data Cards §9 (Known Biases) names at least three named biases with specific mitigations. |           |       |
| 10 | Croissant JSON is syntactically valid JSON.                              |           | **C** |
| 11 | Croissant validates under `mlcroissant validate`.                        |           | **C** |
| 12 | Croissant `@context` includes `@vocab`, `sc`, `cr`, `dct`.               |           |       |
| 13 | Croissant `distribution` includes at least one `cr:FileObject` with `contentUrl`, `encodingFormat`, `sha256`. |           | **C** |
| 14 | Every column in Hub `dataset_info.features` also appears in Croissant `recordSet[0].field` under the same `name`. |           | **C** |
| 15 | Cross-format diff table is present and classifies at least 15 fact classes as lossless / lossy / absent. |           | **C** |
| 16 | Design rationale defends at least three decisions (format choice, long-form vs. compact, PII handling placement). |           |       |
| 17 | NIST AI RMF (MAP, GOVERN) cited; OWASP ML02 and ML06 cited; MITRE ATLAS framing stated. |           |       |
| 18 | At least one `<!-- research-note: ... -->` marker is present where a spec detail is uncertain. |           |       |
| 19 | All three artifacts name the same dataset version and the same license. |           | **C** |
| 20 | No individual role names anywhere in the three artifacts; only role contacts. |           | **C** |

A card that ships without a critical row at `2` is not a card, it is
a draft.

## 5. Common mistakes

1. **Hub frontmatter missing `size_categories` or `task_categories`.**
   Why this bites: the Hub's faceted search uses these as filter
   keys; a card without them is effectively invisible to anyone who
   searches by task or by size. The dataset will be found only by
   direct URL, which defeats the point of publishing a card.
2. **Hub `dataset_info.features` uses the wrong dtype vocabulary.**
   Why this bites: the Hub card parser expects a specific vocabulary
   (for example `dtype: float32` is valid, `dtype: float` is not;
   `dtype: timestamp[us, tz=UTC]` is valid, `dtype: datetime` is
   not). Wrong dtypes silently fall back to `string` in some Hub
   tooling, which breaks typed loaders downstream.
3. **Croissant missing `conformsTo`.** Why this bites: the
   `mlcroissant` validator cannot pick the right schema version
   without it, and in some tooling will skip validation entirely and
   return a false "valid" result. Pin the spec version explicitly.
4. **Croissant fields sourced from the wrong file object.** Why this
   bites: a Croissant record where every column is declared under
   `cr:Field` but none of them is sourced from the actual file set
   is a legal JSON-LD document but not a usable Croissant record — a
   consuming framework cannot mount it. Always point `source.fileSet`
   (or `source.fileObject`) at a real `@id`.
5. **Treating the Data Cards format as "just prose."** Why this
   bites: Data Cards is defined by the three-column prompt / answer
   / artifacts structure, not by the prose alone. A file that is
   twelve headings with prose under each is a notes doc, not a Data
   Card. The "Supporting artifacts" list under each prompt is what
   lets a reviewer walk the evidence.
6. **Trying to put PII policy only in Croissant.** Why this bites:
   Croissant 1.0's core vocabulary does not model PII. The RAI
   extension is not universally supported. A reviewer looking for
   PII in Croissant will either find nothing or find an unsupported
   extension field; either way they cannot complete the review. PII
   belongs in the Data Cards §11 block, and the Hub card's "Personal
   and Sensitive Information" section should mirror it.
7. **Inconsistent versions across the three artifacts.** Why this
   bites: the three files are published together. If the Hub card
   says `2026.11.24.1`, the Data Card says `v3.1`, and the
   Croissant record says `2026-11-24`, a downstream consumer cannot
   tell which one is canonical, and a monitoring job that pins to
   version will accept the wrong artifact.
8. **Inventing `sha256` hashes.** Why this bites: a hand-typed hash
   that does not match the actual file is a lie by omission. The
   validation step must recompute the hash from the file and fail
   the card if they do not match. OWASP ML02 motivates taking this
   seriously.
9. **Naming individual people as curators.** Why this bites: people
   change teams; the dataset outlives any one of them. "Roles not
   humans" means the card still has a working contact when an
   individual leaves. It is also a quiet privacy improvement.
10. **Skipping `<!-- research-note: ... -->` markers where spec
    details are uncertain.** Why this bites: a card that pretends to
    be sure about spec details it is not sure about trains the next
    learner to fake confidence. The marker is the right posture:
    name the uncertainty, point at the source of truth, let the
    next reviewer close it.
11. **Over-indexing on the Hub card's "Citation" section for an
    internal dataset.** Why this bites: internal datasets do not
    have BibTeX entries. Leaving a fake BibTeX stub in makes the card
    look copied. Either omit the section or mark it research-note.
12. **Treating the cross-format diff as optional.** Why this bites:
    the diff is the whole point of the exercise. Three artifacts
    without a diff is a documentation output; three artifacts with a
    diff is a design argument about who the documentation serves.

## 6. References

Official framework and format references:

- **NIST AI Risk Management Framework** — https://www.nist.gov/itl/ai-risk-management-framework
- **OWASP Machine Learning Security Top 10** — https://owasp.org/www-project-machine-learning-security-top-10/
- **MITRE ATLAS** — https://atlas.mitre.org/
- **Hugging Face dataset cards documentation** — https://huggingface.co/docs/hub/datasets-cards
- **MLCommons Croissant** — https://docs.mlcommons.org/croissant/
- **Google Data Cards Playbook** — https://sites.research.google/datacardsplaybook/

Academic citations:

- Pushkarna, M., Zaldivar, A., and Kjartansson, O. (2022).
  *Data Cards: Purposeful and Transparent Dataset Documentation
  for Responsible AI.* In Proceedings of the 2022 ACM Conference on
  Fairness, Accountability, and Transparency (FAccT '22).
- Gebru, T., Morgenstern, J., Vecchione, B., Vaughan, J. W.,
  Wallach, H., Daumé III, H., and Crawford, K. (2021).
  *Datasheets for Datasets.* Communications of the ACM, 64(12).
- Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L.,
  Hutchinson, B., Spitzer, E., Raji, I. D., and Gebru, T. (2019).
  *Model Cards for Model Reporting.* In Proceedings of the
  Conference on Fairness, Accountability, and Transparency
  (FAT* '19).
- Bender, E. M., and Friedman, B. (2018). *Data Statements for
  Natural Language Processing: Toward Mitigating System Bias and
  Enabling Better Science.* Transactions of the Association for
  Computational Linguistics, 6.
- Hutchinson, B., Smart, A., Hanna, A., Denton, E., Greer, C.,
  Kjartansson, O., Barnes, P., and Mitchell, M. (2021). *Towards
  Accountability for Machine Learning Datasets: Practices from
  Software Engineering and Infrastructure.* In Proceedings of the
  2021 ACM Conference on Fairness, Accountability, and
  Transparency (FAccT '21).

VeriSwarm is referenced only as a practitioner example of
dataset-card tooling, not as an authority.
