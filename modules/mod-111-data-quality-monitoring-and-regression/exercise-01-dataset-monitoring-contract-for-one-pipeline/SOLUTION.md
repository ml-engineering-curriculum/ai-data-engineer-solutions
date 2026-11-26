# Exercise 01 — Author `dataset_monitoring_contract.yaml` for One Pipeline (Solution)

## 1. Solution overview

This is a **design** exercise. The learner authors a single pinned
contract — `dataset_monitoring_contract.yaml` — for one concrete
dataset, not a framework. The contract is the artifact the monitor
runs against every pipeline execution and the artifact SREs read at
3am when the alert fires. Sibling exercises compute the SLIs
(ex-02), react to breaches (ex-03), handle subpop drift (ex-04), and
regression-test the contract shape (ex-05); this one names the SLIs.

Deliverables (taken verbatim from the exercise contract):

- `dataset_monitoring_contract.yaml` — the pinned contract itself:
  dataset identity, upstream lineage pins, SLIs, SLOs, alert routing,
  rollback hooks, rollout schedule, ownership, and evidence block.
- `rationale.md` — a design-rationale note that defends each SLO
  number against a named historical breach, a published baseline, or
  an explicit "we do not have evidence; this is a placeholder" call.
- `sample_run_report.yaml` — one worked monitoring-run artifact the
  contract produces so reviewers see what gets emitted.
- `tests/` — pytest fixtures asserting the contract loads under the
  JSON-Schema used by exercise-05.

Governance framing. The contract is the first-line operational
control for silent data quality failure during training-data refresh.
The OWASP Machine Learning Security Top 10 anchors in scope are
**ML02:2023 Data Poisoning Attack** (a shifted feature distribution
or a dropped column is an unintentional poisoning event; the
contract's drift and schema SLIs are what notice it) and **ML05:2023
Model Theft** indirectly, because an unmonitored dataset is a soft
target for injection attacks that later surface as inference-time
leakage. The MITRE ATLAS technique in scope is
**AML.T0020 Poison Training Data** — the contract is the tripwire for
the "did someone slip a bad shard in last night" question. The NIST
AI Risk Management Framework anchors are the
**MEASURE** function (the contract is a measurement artifact for
data-quality risk across the lifecycle) and the **MANAGE** function
(the rollback hooks are the "mechanism to deactivate" the AI RMF
explicitly calls for). We cite the top-level NIST functions rather
than invent sub-item numbers; the exact framework URLs are in §6.

## 2. Implementation — worked answer (a complete contract)

### 2.1 Chosen dataset

We pick one concrete, named pipeline and stay on it throughout:

- **Dataset:** `features.search_reranker_v3` — a tabular feature-store
  table consumed by the inference-time retrieval re-ranker.
- **Granularity:** one row per `(query_id, candidate_doc_id)` pair
  for the last 30 days, partitioned by `event_date`.
- **Columns in scope:** 18 numeric features, 4 categorical features,
  1 label column (`clicked`), partition key, and timestamps.
- **Upstream producer:** `pipelines.search_feature_build` — a nightly
  Airflow DAG whose task-level git commit is pinned in the contract.
- **Downstream consumers:** the re-ranker's offline training job and
  the online feature-store sync that publishes to Redis.

Why this pipeline: it has a schema, a freshness SLA (training needs
today's partition by 06:00 UTC), a subpop axis that matters (query
locale), and a historically observable breach class (producer
silently drops a feature when upstream enrichments error, visible as
a null-rate spike).

### 2.2 Project layout

```
exercise-01/
├── README.md
├── dataset_monitoring_contract.yaml
├── rationale.md
├── sample_run_report.yaml
├── schemas/
│   └── dataset_monitoring_contract.schema.json   # ref'd by exercise-05
└── tests/
    ├── test_contract_loads.py
    ├── test_contract_schema.py
    └── test_sample_report_matches_contract.py
```

### 2.3 `dataset_monitoring_contract.yaml`

```yaml
# dataset_monitoring_contract.yaml
# Monitoring contract for features.search_reranker_v3.
# One contract, one dataset. Pinned. No CLI overrides.
contract_version: 1.2.0
contract_schema: schemas/dataset_monitoring_contract.schema.json
contract_sha256: null   # filled by sign-off step; see §2.8

# ---------------------------------------------------------------------
# 1. Dataset identity
# ---------------------------------------------------------------------
dataset:
  id: features.search_reranker_v3
  slice: event_date=rolling-30d
  storage:
    location: s3://ml-featurestore-prod/search_reranker_v3/
    format: parquet
    partition_key: event_date
  owner:
    team: search-ranking
    slack: "#search-ranking-oncall"
    email: search-ranking@example.internal
  downstream_consumers:
    - id: jobs.search_reranker_offline_training
      contact: search-ranking@example.internal
      sla_impact: blocks nightly training if freshness breaches
    - id: syncs.feature_store_online
      contact: feature-platform@example.internal
      sla_impact: stale online features if freshness breaches
  sla_tier: T1          # T1 (hourly page-worthy) | T2 (next-business-day) | T3 (best-effort)

# ---------------------------------------------------------------------
# 2. Upstream lineage pins
# ---------------------------------------------------------------------
# These are what the monitor verifies against the producer's manifest
# every run. A mismatch between what the producer emitted and what
# this contract pins is itself a breach — see §5 alerts.
lineage:
  producer_pipeline_id: pipelines.search_feature_build
  producer_pipeline_git_commit: 5f3a9c71d2e8                 # 12-hex short
  producer_pipeline_image_digest: >
    sha256:4a1b9c0e8d7f6c5b4a3928170605040302010f0e0d0c0b0a0918171615141312
  data_version:
    scheme: calver        # YYYY.MM.DD.N
    pinned: "2026.11.24.1"
  schema_registry:
    system: confluent     # or `dbt-contracts`, `great-expectations`
    subject: features.search_reranker_v3-value
    version: 17
  reference_window:
    # The window the drift SLIs score against. Pinned so a fresh run
    # cannot silently redefine "normal". Exercise-02 computes digests
    # over this slice and stores them next to the contract.
    label: ref-2026Q3
    start_date: "2026-07-01"
    end_date:   "2026-09-30"
    reference_window_sha256: >
      0f0e0d0c0b0a0918171615141312111009080706050403020100ffeeddccbbaa99

# ---------------------------------------------------------------------
# 3. SLIs — what the monitor computes each run
# ---------------------------------------------------------------------
# Every SLI is: (a) a name, (b) a compute recipe, (c) the SLO that
# makes it actionable. "Compute" is a pointer into exercise-02's
# whylogs/Evidently harness; this contract names the inputs, not the
# code.
slis:

  # ---- 3.1 Schema conformance ----
  - name: schema_conformance
    kind: schema
    compute:
      engine: great_expectations
      suite: suites/features_search_reranker_v3.json   # mod-104 ex-01
      # Columns the monitor pins. Any addition or drop is a breach.
      required_columns:
        - query_id
        - candidate_doc_id
        - event_date
        - event_ts_utc
        - clicked
        # 18 numeric features
        - feat_bm25
        - feat_dense_cosine
        - feat_title_len
        - feat_body_len
        - feat_click_ctr_30d
        - feat_query_log_freq
        - feat_candidate_age_days
        - feat_candidate_quality
        - feat_user_locale_rank
        - feat_session_pos
        - feat_dwell_pred
        - feat_price_log
        - feat_stock_flag
        - feat_domain_authority
        - feat_image_present
        - feat_video_present
        - feat_lang_match
        - feat_prev_click_indicator
        # 4 categorical features
        - cat_locale
        - cat_device
        - cat_vertical
        - cat_query_class
      dtypes:
        clicked: bool
        event_date: date
        event_ts_utc: timestamp[utc]
        cat_locale: dictionary[string]      # pandera / arrow category
        cat_device: dictionary[string]
        cat_vertical: dictionary[string]
        cat_query_class: dictionary[string]
        # All feat_* columns are float32; asserted per-column below
      nullability:
        required_non_null:
          - query_id
          - candidate_doc_id
          - event_date
          - cat_locale
        null_rate_max_per_column:
          # Column-specific allowances, decimal fraction.
          feat_bm25: 0.001
          feat_dense_cosine: 0.001
          feat_click_ctr_30d: 0.05      # cold-start candidates legitimately null
          feat_dwell_pred: 0.02
          feat_price_log: 0.30          # non-commerce verticals have no price
          feat_stock_flag: 0.30
          default: 0.005                # any feat_* not listed above
    slo:
      # Hard: zero tolerance for schema drift. If the producer emits a
      # column we did not pin, the run is BREACHED regardless of the
      # distributional SLIs. See exercise-03 for the workflow this
      # triggers.
      extra_columns_allowed: false
      missing_columns_allowed: 0
      dtype_mismatches_allowed: 0
      null_rate_breach_severity: critical

  # ---- 3.2 Freshness ----
  - name: freshness
    kind: freshness
    compute:
      metric: minutes_since_last_successful_partition
      source: object_store_listing
      timezone: UTC
    slo:
      # The nightly training job starts at 06:00 UTC. We give the
      # producer a 90-minute margin and page if it slips past 07:30.
      max_lag_minutes: 450          # 7h30m since 00:00 UTC
      max_lag_minutes_weekend: 720  # +4h on Sat/Sun; training skipped
      severity: critical

  # ---- 3.3 Volume ----
  - name: volume
    kind: volume
    compute:
      metric: row_count
      window: today_partition
    slo:
      # The 30d rolling P5/P95 is computed by exercise-02 nightly and
      # pinned into the reference-window digest. Absolute floor and
      # ceiling below are tripwires in case the P5/P95 itself is bad.
      floor_rows: 4_000_000
      ceiling_rows: 25_000_000
      # Rolling drift: today's row count against the P5–P95 of the
      # reference window. Breach if outside the band for 2 consecutive
      # runs (debounces a single bad producer run).
      drift:
        rolling_window_days: 30
        band_percentile_low: 5
        band_percentile_high: 95
        consecutive_breaches_to_alert: 2
      severity: high

  # ---- 3.4 Null / empty rates per column ----
  - name: null_rate
    kind: null_rate
    compute:
      metric: column_null_rate
      scope: all_non_partition_columns
    slo:
      # Per-column thresholds are declared under schema.nullability
      # (§3.1). This SLI's role is to flag any column whose null rate
      # breaks its own declared ceiling — we don't repeat the numbers
      # here, we point at the schema block.
      source_of_truth: slis[schema_conformance].compute.nullability.null_rate_max_per_column
      severity: high

  # ---- 3.5 Distributional drift (continuous features) ----
  - name: drift_continuous
    kind: drift
    compute:
      test: ks            # Kolmogorov-Smirnov; exercise-02 implements
      fallback_test: wasserstein_1d
      against: reference_window
      bootstrap_iterations: 500
      multiple_testing_correction: benjamini_hochberg
    slo:
      # One SLO per feature family. A feature's KS statistic above
      # threshold OR its adjusted p-value below alpha is a breach.
      alpha_after_bh: 0.01
      ks_stat_max_per_column:
        feat_bm25: 0.10
        feat_dense_cosine: 0.08
        feat_click_ctr_30d: 0.15     # noisier; cold-start churn
        feat_dwell_pred: 0.15
        feat_price_log: 0.20
        feat_stock_flag: 0.20
        default: 0.10
      consecutive_breaches_to_alert: 1
      severity: high

  # ---- 3.6 Distributional drift (categorical features) ----
  - name: drift_categorical
    kind: drift
    compute:
      test: psi           # Population Stability Index
      fallback_test: chi_square
      against: reference_window
      psi_bins: fixed_from_reference
    slo:
      # PSI bands follow the widely-cited credit-risk convention:
      # < 0.10 stable, 0.10-0.25 moderate, > 0.25 unstable. We page
      # on > 0.25 and warn on > 0.10 for two consecutive runs.
      psi_warn_at: 0.10
      psi_page_at: 0.25
      columns:
        - cat_locale
        - cat_device
        - cat_vertical
        - cat_query_class
      consecutive_breaches_to_alert: 1
      severity: high

  # ---- 3.7 Subpopulation balance drift ----
  - name: subpop_balance_drift
    kind: subpop_drift
    compute:
      # Exercise-04 implements the detection logic; this contract
      # names the subpopulation axis and the metric.
      engine: exercise_04_subpop_detector
      subpop_key: cat_locale
      metric: share_of_rows
    slo:
      # Per-locale share change against reference, absolute percentage
      # points. A 5pp shift in `en-US` matters; a 5pp shift in a
      # locale that is 0.3% of the dataset is noise — hence absolute
      # percentage points gated on a minimum presence.
      min_locale_share_to_score: 0.01
      share_change_pp_warn: 3.0
      share_change_pp_page: 5.0
      severity: medium

  # ---- 3.8 Deduplication-rate drift ----
  - name: dedup_rate_drift
    kind: dedup
    compute:
      # The producer deduplicates on (query_id, candidate_doc_id,
      # event_date). The monitor recomputes the dedup rate and
      # compares to the reference window. Points at mod-105 for the
      # dedup recipe.
      engine: mod_105_dedup_counter
      key: [query_id, candidate_doc_id, event_date]
    slo:
      dedup_rate_change_pp_warn: 1.0
      dedup_rate_change_pp_page: 3.0
      severity: medium

  # ---- 3.9 PII-finding-rate drift ----
  - name: pii_finding_rate_drift
    kind: pii_rate
    compute:
      # Pointer into mod-110 ex-01's pipeline manifest. We do not
      # rescan here; we read `findings.parquet` and aggregate.
      source: mod_110_ex01_findings_parquet
      path_template: >
        s3://ml-privacy-prod/presidio/{producer_data_version}/findings.parquet
      per_entity:
        - EMAIL_ADDRESS
        - PHONE_NUMBER
        - US_SSN
        - CREDIT_CARD
        - PERSON
    slo:
      # Features aren't free text, so finding rate is expected to be
      # near zero. Any entity exceeding 50 findings per million rows
      # pages — likely a producer regression that leaked free-text
      # context into a feature.
      findings_per_million_rows_page: 50
      severity: critical

# ---------------------------------------------------------------------
# 4. Alert routing
# ---------------------------------------------------------------------
# Severity names are pinned: {critical, high, medium, low}. Channels
# are the operational truth, not aspirational. On-call rotation is
# owned by mod-112 and referenced here by name; the contract does
# NOT hard-code individual humans.
alerting:
  channels:
    pagerduty:
      service: PD-SEARCH-RANKING
    slack_critical: "#search-ranking-oncall"
    slack_high:     "#search-ranking-oncall"
    slack_medium:   "#search-ranking-dq"
    slack_low:      "#search-ranking-dq"
    email_digest:   search-ranking@example.internal
  severity_to_channel:
    critical:
      - pagerduty
      - slack_critical
    high:
      - slack_high
      - email_digest
    medium:
      - slack_medium
      - email_digest
    low:
      - email_digest
  oncall_rotation:
    # Owned by mod-112; we only reference the handle.
    source: mod_112_oncall
    rotation_id: search-ranking-primary
    escalation_policy_id: PD-EP-SEARCH-RANKING-STD
  silence_rules:
    # Explicit silence windows during producer-known maintenance.
    - window: "scheduled_producer_backfill"
      pagerduty_only: true
      requires_approver_role: data_platform_oncall

# ---------------------------------------------------------------------
# 5. Rollback hooks
# ---------------------------------------------------------------------
# What the monitor can do when a breach is confirmed. The ACTIONS are
# pinned; the WHO is a role, not a name.
rollback:
  snapshot:
    # mod-109 owns dataset versioning; the snapshot id below is what
    # we revert to. The previous successful data_version is recorded
    # in every run report and promoted here on green.
    source: mod_109_dataset_snapshots
    last_known_good_data_version: "2026.11.23.1"
  revert_command: >
    aicg dataset revert
      --dataset features.search_reranker_v3
      --to-data-version {last_known_good_data_version}
      --approver {approver_user}
      --reason-ticket {jira_ticket}
  requires_approval:
    - role: search_ranking_oncall
    - role: data_platform_oncall           # dual control
  automatic_rollback_allowed: false        # humans only; see §5 of rationale.md
  post_rollback_checks:
    - name: training_job_dry_run
      command: aicg train --dataset features.search_reranker_v3 --dry-run
    - name: feature_store_online_parity
      command: aicg feature-store compare-offline-online

# ---------------------------------------------------------------------
# 6. Rollout schedule
# ---------------------------------------------------------------------
# Promoting this contract from candidate -> enforced is itself a
# staged rollout. Exercise-05 regression-tests the contract shape
# before any promotion.
rollout:
  burn_in:
    # Non-paging shadow period. The monitor emits a run report every
    # execution but severity is capped at `low` regardless of SLI
    # verdict.
    duration_days: 14
    cap_severity_at: low
  ramp:
    # Progressive promotion of severity caps.
    stages:
      - from_day: 15
        to_day:   21
        cap_severity_at: medium
      - from_day: 22
        to_day:   28
        cap_severity_at: high
      - from_day: 29
        cap_severity_at: critical           # full enforcement
  exit_criteria:
    - zero_unexplained_criticals_during_burn_in: true
    - every_sli_has_at_least_one_green_run: true
    - rationale_md_signed_off_by_owner_team: true

# ---------------------------------------------------------------------
# 7. Ownership + escalation chain
# ---------------------------------------------------------------------
# Humans-as-roles only; names live in mod-112.
ownership:
  primary:
    role: search_ranking_oncall
    rotation_id: search-ranking-primary
  secondary:
    role: data_platform_oncall
    rotation_id: data-platform-primary
  escalation:
    - after_minutes: 15
      role: search_ranking_oncall_secondary
    - after_minutes: 45
      role: search_ranking_manager
    - after_minutes: 120
      role: director_search

# ---------------------------------------------------------------------
# 8. Evidence block
# ---------------------------------------------------------------------
# Every monitor run emits one manifest. Reviewers at post-mortem read
# this, not log lines. The schema of the manifest is the output half
# of the exercise-05 JSON-Schema.
evidence:
  manifest_path_template: >
    s3://ml-dq-prod/runs/{dataset_id}/{run_id}/run_manifest.yaml
  required_fields:
    - run_id
    - started_at_utc
    - finished_at_utc
    - dataset_id
    - data_version_scanned
    - reference_window_sha256
    - contract_sha256
    - producer_pipeline_git_commit
    - sli_values                 # one entry per SLI name; numeric
    - verdicts                   # one entry per SLI name; enum
    - overall_verdict            # pass | warn | breach
    - alert_fanout               # which channels were paged
    - rollback_invoked           # bool
    - rollback_data_version      # null unless rollback_invoked
  retention_days: 365
  signing:
    # The manifest is signed with the DQ-platform service key so a
    # downstream consumer can detect tampering.
    sign_with_kms_key_id: alias/aicg-dq-manifest-signer
    algorithm: ECDSA_SHA_256
```

### 2.4 Design rationale (excerpt from `rationale.md`)

The full note lives in `rationale.md`; the review-worthy bits:

- **Why KS for continuous and PSI for categorical.** KS is
  distribution-free and does not require binning; it also
  underweights tail changes, which for click-through features is
  acceptable (tails are already noisy). PSI is the industry baseline
  for categorical shift, well-understood, and its 0.10 / 0.25
  thresholds are quotable at review. Exercise-02 implements both.
- **Why Benjamini–Hochberg on `drift_continuous`.** 18 numeric
  features times nightly runs is a false-discovery-rate problem at
  Bonferroni; BH gives the desired FDR of 1% without flattening
  sensitivity to a true single-feature drift.
- **Why per-column null allowances.** Flat null thresholds either
  (a) page on cold-start candidates every night or (b) hide a
  genuine upstream failure. The per-column numbers encode the
  domain fact that `feat_price_log` and `feat_stock_flag` are null
  for non-commerce verticals.
- **Why automatic rollback is off.** Rolling back the feature-store
  table during nightly training is cheap; rolling it back during
  online feature-sync propagation is not — stale online features
  degrade live search quality. Dual-approver gate is deliberate.
- **Why burn-in is 14 days.** One weekly cycle plus one buffer. Any
  shorter and the weekend-locale-share variance looks like drift.

The 50-per-million PII-finding-rate SLO is a defensible placeholder;
the authoritative incident corpus that would ground it numerically
lives in `mod-110` exercise-01's findings archive, and reviewers are
expected to tighten the number once that archive is linked here.

### 2.5 Sample run report (`sample_run_report.yaml`)

The monitor produces one of these per run, written to the path pinned
in §8.

```yaml
# sample_run_report.yaml
run_id: dq-2026-11-24-0612
started_at_utc:  "2026-11-24T06:12:03Z"
finished_at_utc: "2026-11-24T06:17:41Z"
dataset_id: features.search_reranker_v3
data_version_scanned: "2026.11.24.1"
reference_window_sha256: >
  0f0e0d0c0b0a0918171615141312111009080706050403020100ffeeddccbbaa99
contract_sha256: >
  9e8d7c6b5a4938271605f4e3d2c1b0a09e8d7c6b5a4938271605f4e3d2c1b0a0
producer_pipeline_git_commit: 5f3a9c71d2e8

sli_values:
  schema_conformance:
    required_columns_present: 26
    required_columns_missing: 0
    extra_columns_present: 0
    dtype_mismatches: 0
    null_rates:
      feat_bm25: 0.0002
      feat_dense_cosine: 0.0003
      feat_click_ctr_30d: 0.041
      feat_price_log: 0.287
      feat_stock_flag: 0.287
  freshness:
    minutes_since_last_successful_partition: 372
  volume:
    row_count: 11_842_613
    reference_p5: 9_900_000
    reference_p95: 14_200_000
  drift_continuous:
    ks_statistics:
      feat_bm25: 0.062
      feat_dense_cosine: 0.051
      feat_click_ctr_30d: 0.118
      feat_price_log: 0.093
    columns_over_threshold: []
  drift_categorical:
    psi:
      cat_locale: 0.07
      cat_device: 0.04
      cat_vertical: 0.11        # warn band
      cat_query_class: 0.03
  subpop_balance_drift:
    share_change_pp:
      "en-US": 0.3
      "en-GB": -0.1
      "de-DE": 0.2
      "ja-JP": -0.4
  dedup_rate_drift:
    dedup_rate: 0.013
    reference_dedup_rate: 0.012
    change_pp: 0.1
  pii_finding_rate_drift:
    findings_per_million_rows:
      EMAIL_ADDRESS: 0
      PHONE_NUMBER: 1
      US_SSN: 0
      CREDIT_CARD: 0
      PERSON: 3

verdicts:
  schema_conformance: pass
  freshness: pass
  volume: pass
  null_rate: pass
  drift_continuous: pass
  drift_categorical: warn       # cat_vertical in warn band
  subpop_balance_drift: pass
  dedup_rate_drift: pass
  pii_finding_rate_drift: pass

overall_verdict: warn
alert_fanout:
  - slack_medium
  - email_digest
rollback_invoked: false
rollback_data_version: null

signature:
  kms_key_id: alias/aicg-dq-manifest-signer
  algorithm: ECDSA_SHA_256
  signature_b64: MEUCIQD...truncated...==
```

### 2.6 What the regression harness (exercise-05) will assert

Exercise-05 reads the contract and checks its *shape*, not its values.
The JSON-Schema at `schemas/dataset_monitoring_contract.schema.json`
pins the invariants: every `slo` block has a `severity`; every SLI
name is unique; `rollback.requires_approval` has at least one role;
`evidence.required_fields` is a superset of the hardcoded minimum
list. The contract-shape test is what prevents a future refactor from
silently dropping the `rollback_invoked` field.

### 2.7 Sibling-exercise handoffs (pinned in the contract)

- **exercise-02** owns `compute.engine: whylogs` / `evidently` for the
  three drift SLIs. It reads `reference_window_sha256` and asserts
  it against its own profile digest before scoring.
- **exercise-03** owns the alert-to-rollback workflow. It reacts to
  `verdicts[schema_conformance] == breach` and invokes the pinned
  `rollback.revert_command`.
- **exercise-04** owns subpop detection logic. The `subpop_balance_drift`
  SLI's `compute.engine` is the exercise-04 detector by name.
- **exercise-05** owns the regression fixtures that assert the YAML
  parses, the JSON-Schema accepts it, and
  `sample_run_report.yaml` matches the `evidence` block.
- **mod-104 exercise-01** owns the Great Expectations suite behind
  `schema_conformance` and the per-column null thresholds.
- **mod-109** owns dataset versioning — `lineage.data_version` and
  `rollback.snapshot.last_known_good_data_version` both point into it.
- **mod-112** owns the on-call rotation and the escalation chain;
  the contract references `rotation_id`s, not humans.

### 2.8 Contract sign-off

The `contract_sha256` field is intentionally `null` on authorship
and filled by a sign-off step. The flow:

1. Author lands the contract with `contract_sha256: null`.
2. `aicg dq contract sign --file dataset_monitoring_contract.yaml`
   computes SHA-256 of the canonical YAML (keys sorted, no trailing
   whitespace, NFC-normalised) and writes it back.
3. The monitor refuses to run on a contract whose on-disk SHA-256
   does not match the self-declared value — a drift check analogous
   to the slice-hash check in mod-110 ex-01.

## 3. Validation steps

1. **YAML lints clean.** `yamllint -s dataset_monitoring_contract.yaml`
   exits 0. The repo's `.yamllint.yml` pins 2-space indent, line
   length 120, and prohibits tabs.
2. **JSON-Schema accepts the contract.** `python -m jsonschema -i
   dataset_monitoring_contract.yaml schemas/dataset_monitoring_contract.schema.json`
   exits 0. This is the gate exercise-05 reuses.
3. **Pytest fixtures pass.** `pytest tests/` runs three cases:
   `test_contract_loads.py` (`yaml.safe_load` returns a dict),
   `test_contract_schema.py` (JSON-Schema accepts the on-disk file),
   and `test_sample_report_matches_contract.py` (every SLI named in
   the contract has a verdict in the sample report).
4. **Contract SHA-256 is self-consistent.** After running `aicg dq
   contract sign`, re-hashing the canonical YAML reproduces the
   `contract_sha256` value exactly. A manual edit without re-signing
   breaks the monitor on next run.
5. **Reference-window hash is pinned.** `grep reference_window_sha256
   dataset_monitoring_contract.yaml` matches `sha256sum
   artifacts/reference_window_ref-2026Q3.parquet`. Exercise-02 writes
   that artifact; the contract references it.
6. **Severity labels are from the pinned enum.** `python -c "import yaml;
   c = yaml.safe_load(open('dataset_monitoring_contract.yaml'));
   bad = [s['slo'].get('severity') for s in c['slis'] if s['slo'].get('severity')
   not in {'critical','high','medium','low'}]; assert not bad, bad"`
   exits 0. The alerting block's `severity_to_channel` keys are the
   same enum.
7. **Every SLI has a channel.** For each `severity` that appears in any
   `slos[*].severity`, `alerting.severity_to_channel` has a non-empty
   list. Script: `scripts/check_sli_routing.py`.
8. **Rollback is dual-approver.** `len(rollback.requires_approval) >= 2`
   and `rollback.automatic_rollback_allowed` is `false` or has an
   accompanying signed waiver referenced by ticket id.
9. **Burn-in semantics are enforced.** During the window
   `now < rollout.burn_in.duration_days` from contract creation, the
   monitor caps `overall_verdict`'s routed severity at
   `rollout.burn_in.cap_severity_at`. A smoke test posts a synthetic
   `critical` breach during burn-in and asserts PagerDuty is NOT
   invoked.
10. **Evidence block fields are present in the sample report.** Every
    field listed in `evidence.required_fields` appears in
    `sample_run_report.yaml`. Script:
    `scripts/check_evidence_fields.py`.
11. **No humans by name.** `grep -E
    '([a-z]+\.)?[a-z]+@(?!(example\.internal))' dataset_monitoring_contract.yaml`
    returns no matches. The contract routes to teams and roles; names
    live in mod-112.
12. **Lineage pins are non-placeholder.** `producer_pipeline_git_commit`
    matches `^[0-9a-f]{7,40}$`; `data_version` matches the pinned
    CalVer regex; `producer_pipeline_image_digest` starts with
    `sha256:` and has 64 hex chars after.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: **>= 22/30 with zero
zeros on critical rows**.

| #  | Dimension                                                                                                                 | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Dataset identity block pins `dataset.id`, slice, owner team, downstream consumers, and SLA tier                            | Yes       |
| 2  | Upstream lineage pins producer pipeline id, git commit, image digest, data version, and reference-window hash              | Yes       |
| 3  | Schema SLI enumerates every required column by name and pins dtypes for non-float columns                                  | Yes       |
| 4  | Per-column null-rate thresholds are declared with a documented default; cold-start columns justified in `rationale.md`     | Yes       |
| 5  | Freshness SLI pins both a weekday and a weekend max-lag                                                                    | No        |
| 6  | Volume SLI has an absolute floor/ceiling AND a rolling-window P5/P95 band with a consecutive-breach debounce               | Yes       |
| 7  | Continuous-drift SLI names a test (KS or Wasserstein), a reference window, an alpha, and a multiple-testing correction    | Yes       |
| 8  | Categorical-drift SLI uses PSI or chi-square with pinned warn/page thresholds                                              | Yes       |
| 9  | Subpop-balance-drift SLI names a subpop key and a minimum-presence gate; points at exercise-04 by name                     | Yes       |
| 10 | Dedup-rate-drift SLI references the mod-105 recipe and compares to the reference window                                    | No        |
| 11 | PII-finding-rate-drift SLI references the mod-110 ex-01 findings artifact and per-entity thresholds                        | Yes       |
| 12 | Every SLI has a `severity` from the pinned enum and a routed channel in `severity_to_channel`                              | Yes       |
| 13 | Rollback hook pins a last-known-good data version, a revert command, >= 2 approver roles, and post-rollback checks         | Yes       |
| 14 | Automatic-rollback flag is explicit (`false` by default); any `true` setting has a signed waiver                           | Yes       |
| 15 | Rollout block has burn-in, ramp stages with severity caps, and exit criteria                                               | Yes       |
| 16 | Ownership uses roles and rotation ids; no human names appear in the contract                                               | Yes       |
| 17 | Evidence block lists required manifest fields including `contract_sha256` and `reference_window_sha256`                    | Yes       |
| 18 | Contract is signed (`contract_sha256` non-null) and the monitor refuses mismatched contracts                               | Yes       |
| 19 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0020, NIST AI RMF MEASURE/MANAGE) cited in overview or rationale | No        |
| 20 | Unverifiable claims are called out inline as placeholders with a named owner for the follow-up                             | Yes       |
| 21 | `rationale.md` defends every SLO number against a breach, baseline, or explicit placeholder                                | Yes       |
| 22 | `sample_run_report.yaml` exists and matches `evidence.required_fields`                                                     | Yes       |
| 23 | Sibling-exercise handoffs (ex-02/03/04/05, mod-104/109/112) are named in the contract, not implied                         | No        |

## 5. Common mistakes

- **Pinning SLOs with adjectives instead of numbers.** "Page if drift
  is significant" means the monitor is a wish. Why this bites: at 3am
  the on-call has to decide what "significant" means, and the
  decision becomes about who shouted loudest. Pin numbers. If you
  cannot justify a number, call it out inline as a placeholder with a
  named owner for the follow-up so reviewers see the debt.
- **One global null threshold for every column.** Why this bites:
  columns that are legitimately sparse (`feat_price_log` in a
  non-commerce vertical) either get flat-out banned or hide a real
  producer failure. Per-column thresholds with a documented default
  are the only sustainable shape.
- **No reference-window hash.** A reference window that is "the last
  90 days" silently redefines itself every run; a drift SLI against
  a moving reference will never fire on slow drift. Why this bites:
  the SLI looks green while quality drops 2% / week.
- **Hard-coding on-call humans.** Why this bites: Alice leaves the
  team, the contract is now a landmine. Roles and `rotation_id`s only;
  mod-112 owns the mapping from role to human.
- **Automatic rollback enabled by default.** Why this bites: a
  single spurious breach invalidates today's training run or, worse,
  rolls back the online feature-store sync and visibly degrades
  live product quality. Automatic rollback must be opt-in with a
  signed waiver.
- **Severity-to-channel map that routes `low` to PagerDuty.** Why
  this bites: alert fatigue destroys the signal within a quarter.
  The pinned enum plus the explicit channel map prevents an
  accidental `low -> pagerduty` entry from the alerting system's
  default behaviour.
- **Shadow/burn-in skipped because "the contract is obvious".** Why
  this bites: the first real monitor run almost always finds a
  pre-existing baseline problem (a feature that has been silently
  drifting for weeks) which the burn-in would have surfaced before
  the contract became load-bearing.
- **Schema SLI that allows extra columns.** `extra_columns_allowed:
  true` is comfortable and wrong. Why this bites: a producer silently
  adds `feat_experimental_v2` and nothing notices; three weeks later
  the re-ranker starts reading it as `NaN` and metrics degrade.
- **No consecutive-breach debounce on volume.** Why this bites: a
  single partition late in the write path has a transient row count
  dip, and the monitor pages on a self-healing problem. Two
  consecutive runs is the right tradeoff for daily partitions.
- **Reusing the producer's own schema registry version as the
  source of truth.** Why this bites: the contract then re-ratifies
  whatever the producer did last night. The schema SLI must pin
  columns in the contract itself; the registry version is additional
  metadata, not ground truth.
- **Treating `contract_sha256` as cosmetic.** Why this bites: a
  reviewer silently edits a threshold between landings and no run
  report flags the change. The contract hash must be a load-bearing
  input the monitor refuses to run against on mismatch.
- **One mega-SLI that computes "overall data quality".** Why this
  bites: when it fires, nobody knows which dimension broke, and the
  post-mortem spends an hour reverse-engineering the alert. Keep the
  SLIs named, narrow, and individually actionable.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The schema and drift SLIs are the tripwire for unintentional
  poisoning (bad shard, dropped column, class-balance shift).
- **MITRE ATLAS — AML.T0020 Poison Training Data:**
  <https://atlas.mitre.org/>. The attacker-facing technique the
  contract's SLIs are designed to detect; the subpop-balance and
  dedup-rate SLIs are the softer-signal detectors.
- **NIST AI Risk Management Framework (MEASURE, MANAGE, GOVERN):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  anchors the contract as a measurement artifact across the data
  lifecycle; MANAGE anchors the rollback hooks as the "mechanism to
  deactivate" the framework requires.
- **Great Expectations:** <https://greatexpectations.io/>. The
  suite behind the schema-conformance and per-column null SLIs; see
  mod-104 exercise-01 for the pinned suite.
- **dbt data contracts:**
  <https://docs.getdbt.com/docs/collaborate/govern/model-contracts>.
  The upstream-side contract this monitoring contract pairs with;
  the dbt model contract is what the producer declares, this file is
  what the consumer verifies.
- **Open Data Contract Standard (ODCS):**
  <https://bitol-io.github.io/open-data-contract-standard/latest/>.
  The industry-aligned contract vocabulary for identity, lineage, and
  SLA fields; align section names with ODCS where possible.
- **OpenTelemetry:** <https://opentelemetry.io/>. The emission
  channel for SLI values as metrics (one metric per SLI name); the
  evidence manifest and the OTLP metric stream are two views of the
  same run.
- **whylogs:** <https://whylabs.ai/whylogs>. The profile format the
  reference-window digest is computed against in exercise-02.
- **Evidently AI:** <https://docs.evidentlyai.com/>. The alternative
  drift-test engine exercise-02 compares against whylogs.
- **PSI threshold convention (0.10 / 0.25):** widely used in
  credit-risk model monitoring; the exact origin paper is often cited
  as Siddiqi (2006), *Credit Risk Scorecards*. Treat the 0.10 / 0.25
  bands as convention rather than universal law; some texts cite
  0.10 / 0.20, and a primary-source citation is a defensible follow-up
  for the reviewer who ratifies this contract.
- **Local anchors:** exercise-02 (whylogs/Evidently SLI compute —
  consumes `reference_window_sha256` from this contract); exercise-03
  (alert-to-rollback workflow — consumes `verdicts` and
  `rollback.revert_command`); exercise-04 (subpop-balance detector —
  implements the `subpop_balance_drift` SLI's `compute.engine`);
  exercise-05 (regression fixtures — asserts this contract parses and
  that the sample report matches the evidence block); `mod-104`
  exercise-01 (Great Expectations suite behind schema/null SLIs);
  `mod-105` (dedup recipe for `dedup_rate_drift`); `mod-109` (dataset
  versioning — provides `data_version` and
  `last_known_good_data_version`); `mod-110` exercise-01 (Presidio
  findings parquet — source for `pii_finding_rate_drift`); `mod-112`
  (on-call rotation and escalation chain — provides `rotation_id`s).
