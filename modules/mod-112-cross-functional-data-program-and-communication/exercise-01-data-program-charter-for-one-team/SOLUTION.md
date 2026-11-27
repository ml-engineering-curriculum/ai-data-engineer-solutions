# Exercise 01 — Author a Data Program Charter For One Team (Solution)

## 1. Solution overview

This is a **design** exercise. The learner authors a single document —
`CHARTER.md` — that a new data-engineering lead writes in their first 60
days and that the VP of Engineering countersigns. The charter is not a
runbook, not a roadmap, and not an org chart. It is the artifact that
says, in writing, four things that no other document in the company
says together: what the team is on the hook for, what it refuses, how
its partners can expect it to behave, and the small set of numbers the
team agrees to be judged on. Everything downstream — on-call rotations,
labelling-vendor contracts, EU AI Act Annex XI training-data summaries,
model-card data slices — hangs off this document. If the charter is
vague, the downstream artifacts are vague.

Deliverables (per the exercise contract):

- `CHARTER.md` — the charter itself. One concrete team, one page of
  scope, one page of decision rights, one page of SLAs, one page of
  KPIs, half a page of cadences and non-goals, a sign-off block.
- A "design rationale" subsection inside the charter that defends the
  3-4 hardest decisions (why these KPIs and not others, why these
  decision rights sit where they do). This is where the charter earns
  its right to be signed, not just published.
- Cross-references to the sibling mod-112 exercises so a reader can go
  from charter to dataset card (ex-02), to EU AI Act Annex XI summary
  (ex-03), to model-card slice (ex-04), to certification plan (ex-05).

Governance framing. A charter is first and foremost a **governance**
artifact, not an operational one. Three framework anchors apply:

- **NIST AI Risk Management Framework — GOVERN function.** The GOVERN
  pillar of the NIST AI RMF is explicitly about "cultures, policies,
  processes, and practices" that make AI risk visible and owned. A
  one-team charter is the smallest useful GOVERN artifact: it names
  roles, decision rights, escalation paths, and the KPIs the team is
  judged on. The **MAP** function is also in scope — the scope and
  non-goals section is a MAP exercise, because MAP is where an
  organisation declares the context and purpose of the AI system (or,
  here, the team that supports it). We cite the top-level NIST function
  names rather than invent sub-category numbers; the framework URL is
  in §6.
- **OWASP Machine Learning Security Top 10.** Two anchors matter for a
  data-engineering team charter. **ML02:2023 Data Poisoning Attack**
  anchors the "we own the data-quality tripwires" line of the charter
  (the SLA the team owes its ML-training partner is, at root, a
  poisoning-defence SLA). **ML06:2023 AI Supply Chain Attacks** anchors
  the vendor-labelling SLA (we treat upstream labelling vendors as a
  supply-chain risk, and the charter commits the team to specific
  vendor-audit cadences — see mod-107 ex-02 for the RFP, which this
  charter obligates the team to re-run annually).
- **MITRE ATLAS.** ATLAS is primarily a threat-knowledge base, but its
  mitigation side maps directly to the "who defends what" line in a
  charter. In particular, a data-engineering charter owns the defender
  half of poisoning and supply-chain techniques; the charter is where
  the team publicly commits to being the owner.

The charter is also where the team draws the line against three
anti-patterns that recur in young data orgs: scope creep ("sure, we can
own that too"), decision-rights confusion ("I thought platform decided
that"), and KPI inflation ("let's just track everything"). The design
rationale subsection defends the specific choices made against each.

Why this exercise deserves care. In practice, a vague charter is the
root cause of three organisational failure modes a VP will eventually
pay to debug: an on-call team that fights alerts it does not own, a
partner team that cannot plan because its upstream commitments are
verbal, and a quarterly review where leadership and the team disagree
on what "good" looks like. The charter closes all three — but only if
it is specific. A one-page charter that says "we own data quality" is
worse than no charter at all, because it looks like coverage and is
not.

## 2. Implementation — worked answer (a full `CHARTER.md`)

### 2.1 Chosen team

We stay on one concrete team for the entire charter:

- **Team name:** `search-ranking-data-engineering` (SRDE).
- **Parent org:** Search Platform, reporting to the VP of Engineering.
- **Headcount at charter sign:** 6 engineers, 1 engineering manager,
  0.4 FTE of a shared data scientist from the analytics guild.
- **Primary product surface:** the retrieval re-ranker that serves
  consumer search on the flagship product. Offline training consumes
  `features.search_reranker_v3`; online inference consumes the Redis
  mirror.
- **Why this team:** it has an ML-training partner (the re-ranker
  model team), a platform partner (feature-store SRE), a privacy
  partner (query logs touch user PII), and an analytics partner
  (search KPIs are company-level). That is the minimum cross-functional
  surface area where a charter earns its keep. We reuse the
  `features.search_reranker_v3` world from mod-111 ex-01 so reviewers
  can cross-reference the monitoring contract this charter obligates
  the team to maintain.

### 2.2 Project layout

```
exercise-01/
├── CHARTER.md                     # the charter itself
├── signoff/
│   ├── ml-training.md             # countersigned by ML-training lead
│   ├── platform-infra.md          # countersigned by platform lead
│   ├── legal-privacy.md           # countersigned by legal/privacy lead
│   └── product-analytics.md       # countersigned by analytics lead
└── changelog.md                   # amendments, dated, with charter_sha256
```

### 2.3 The charter

```markdown
# Charter — search-ranking-data-engineering (SRDE)

charter_version: 1.0.0
charter_effective: 2026-12-01
charter_review_due: 2027-12-01
charter_sha256: null   # filled by the sign-off script, see §9
countersigned_by:
  - role: VP of Engineering
    signoff_file: signoff/vp-engineering.md
  - role: ML Training Lead
    signoff_file: signoff/ml-training.md
  - role: Platform/Infra Lead
    signoff_file: signoff/platform-infra.md
  - role: Legal/Privacy Lead
    signoff_file: signoff/legal-privacy.md
  - role: Product Analytics Lead
    signoff_file: signoff/product-analytics.md

> Roles, not humans. This document never names an individual. Rotations,
> promotions, and departures must not require a charter amendment.

---

## 1. Mission

SRDE owns the data pipelines, feature tables, and data-quality
tripwires that make the consumer search re-ranker trainable, auditable,
and legally defensible. We measure success as: the model team can
retrain on a trustworthy snapshot on any business day, and we can
produce an EU AI Act Annex XI training-data summary for the current
production model within one business day of a regulator or auditor
request.

## 2. Scope

### 2.1 In scope

- The `features.search_reranker_v3` offline feature table and all its
  partitions on and after `event_date = 2025-01-01`.
- The producer pipeline `pipelines.search_feature_build` end to end:
  source extraction from query logs and click-stream, feature
  computation, schema enforcement, partition emission.
- The online mirror `syncs.feature_store_online` up to and including
  the Redis write; read-side latency is out of scope (owned by
  Platform/Infra — see §4.2).
- The dataset monitoring contract `dataset_monitoring_contract.yaml`
  for `features.search_reranker_v3` (authored in mod-111 ex-01).
- The dataset card for `features.search_reranker_v3` in all three
  formats required by mod-112 ex-02 (human-readable, Croissant JSON-LD,
  Hugging Face dataset card).
- The training-data portion of the Annex XI summary for any production
  re-ranker that was trained against our feature table (mod-112 ex-03).
- The data-slice inputs for the model card the ML-training team
  publishes (mod-112 ex-04): per-locale, per-device, per-vertical row
  counts, class-balance tables, null-rate tables.
- Vendor-labelling contracts for any human-labelled slice we commission
  (currently: the `cat_query_class` relabelling audit, done quarterly
  via the vendor contracted under mod-107 ex-02).

### 2.2 Out of scope

- Model architecture, loss function, hyperparameters. SRDE does not
  decide what the re-ranker learns, only what data it learns from.
- Online serving latency, autoscaling, and Redis cluster health. Owned
  by Platform/Infra.
- Experimentation and A/B test design. Owned by Product Analytics.
- Search product strategy, ranking policy, and editorial guidelines.
  Owned by the Search Product org.
- Any dataset that is not `features.search_reranker_v3` and not an
  upstream of it. In particular, we do not own: the query-understanding
  features consumed by the retrieval stage, the ads pipeline, the
  spell-correction corpus.

### 2.3 Edge cases (explicit, because these bite)

- **Query-log PII.** The upstream query-log table is owned by the
  Logging Platform team, but the derived feature `feat_query_log_freq`
  is ours. If a user issues a GDPR right-to-erasure request, Logging
  Platform deletes the row and SRDE re-emits the affected feature
  partitions. The deletion workflow is documented in mod-110 ex-03;
  SRDE's charter commitment is that the re-emission completes within
  the SLA in §4.3.
- **Shared categorical vocabularies.** `cat_vertical` is a vocabulary
  maintained jointly with Product Analytics. Additions to the
  vocabulary are an Analytics decision (they own taxonomy); the
  downstream re-coding of historical partitions is an SRDE decision
  and an SRDE cost.
- **Synthetic augmentation data.** If the ML-training team wants to
  mix synthetic data into training (mod-108 territory), SRDE owns
  the provenance tag on the synthetic rows and refuses to publish them
  into `features.search_reranker_v3` without a provenance column that
  separates them from observed rows. Mixing policy is the model team's
  decision; the provenance column is not negotiable.
- **Benchmark contamination.** If a public search-relevance benchmark
  ever enters our training mix, the de-contamination check from
  mod-106 is SRDE's responsibility. The benchmark choice is the model
  team's.

## 3. Decision rights (RACI)

> RACI legend: **R** = responsible (does the work), **A** = accountable
> (one role, final say), **C** = consulted (must be asked before
> decision is final), **I** = informed (told after decision is final).
> There is exactly one **A** per row. RACI is a BABOK concept; we are
> not inventing it here.

| Decision                                                                  | SRDE EM | SRDE Tech Lead | ML Training Lead | Platform Lead | Legal/Privacy Lead | Analytics Lead | VP Eng |
|---------------------------------------------------------------------------|---------|----------------|------------------|---------------|--------------------|----------------|--------|
| Add a new feature column to `features.search_reranker_v3`                 | C       | A              | C                | I             | C (if PII-derived) | I              | I      |
| Remove / deprecate an existing feature column                             | C       | R              | A                | I             | I                  | C              | I      |
| Change a feature's computation (same name, new semantics)                 | C       | A              | C                | I             | C (if PII-derived) | C              | I      |
| Change the monitoring contract SLOs (mod-111 ex-01)                       | C       | A              | C                | I             | I                  | I              | I      |
| Deprecate `features.search_reranker_v3` wholesale                         | C       | C              | C                | C             | C                  | C              | A      |
| Approve a labelling-vendor contract (mod-107 ex-02)                       | A       | C              | C                | I             | C                  | I              | I      |
| Change data retention for raw query logs underlying our features         | I       | C              | I                | C             | A                  | I              | I      |
| Change data retention for `features.search_reranker_v3` partitions        | C       | A              | C                | C             | C                  | I              | I      |
| Approve a production re-ranker model release (data-side gate only)       | C       | A              | R                | I             | C (if slice-fail)  | I              | I      |
| Approve an emergency rollback of a feature partition                      | I       | A              | C                | I             | I                  | I              | I      |
| Grant a new team read access to `features.search_reranker_v3`             | A       | R              | I                | I             | C                  | I              | I      |
| Add a new partner-team SLA to this charter                                | R       | C              | C                | C             | C                  | C              | A      |
| Amend this charter                                                        | R       | C              | C                | C             | C                  | C              | A      |

Notes on the hard cells:

- "Remove / deprecate an existing feature column" is **A = ML Training
  Lead** because the model is what breaks when a feature disappears,
  not the pipeline. SRDE proposes and does the work; the model team
  owns the consequences.
- "Approve a production re-ranker model release (data-side gate only)"
  is **A = SRDE Tech Lead** for the data-quality half of the release
  gate, not the model's half. The ML Training Lead is R because they
  run the release; SRDE Tech Lead is A for whether the data meets the
  contract. Legal/Privacy is C only when a protected slice has failed
  the slice checks in mod-112 ex-04.
- "Change data retention for raw query logs" is **A = Legal/Privacy
  Lead**, not SRDE, because raw logs are PII. SRDE is only I on that
  row — we do not have the authority to extend retention even if we
  want longer windows for feature backfills.

## 4. Partner-team SLAs

Each SLA is bidirectional. SRDE owes something; the partner owes
something back. The counter-party-signoff file is listed per SLA so a
reviewer can verify the SLA is not self-granted.

### 4.1 ML Training (partner: `search-reranker-model` team)

Counter-party signoff: `signoff/ml-training.md`.

What SRDE owes:

- **Freshness.** The daily partition of `features.search_reranker_v3`
  for `event_date = D` is readable by `06:00 UTC` on day `D+1`, 95% of
  days over any rolling 30-day window. This is the SLO that the
  monitoring contract in mod-111 ex-01 enforces.
- **Schema stability.** No column removal or dtype change without 10
  business days' notice to this partner, with a dated deprecation
  entry in `changelog.md`. New columns are allowed with 2 business
  days' notice.
- **Reproducibility.** Any partition the model team trained on in the
  last 365 days is recoverable (either live or restorable within 1
  business day), with an immutable version pin (mod-109 ex-01 world).
- **Slice health.** The per-locale and per-vertical row-count tables
  required by mod-112 ex-04 are refreshed weekly and linked from the
  model card.

What ML Training owes back:

- Named consumers. The model team publishes the list of columns the
  current production model reads. SRDE's deprecation workflow uses
  this list as ground truth; a column not on the list is fair game for
  removal after the 10-day notice.
- Advance notice of data-side needs. New features the model team wants
  must be requested with at least 20 business days of lead time, in
  writing, with a measurable training-metric motivation.
- Attribution of regressions. When a model-quality regression lands,
  the model team runs the training-side ablation before pointing at
  data. SRDE runs the data-side ablation after. The order matters
  because data-side ablations are expensive (full partition rebuilds).

### 4.2 Platform / Infra (partner: `feature-store-platform` team)

Counter-party signoff: `signoff/platform-infra.md`.

What SRDE owes:

- **Idempotent writes.** Every partition we publish is content-hashed
  and rewriting the same logical partition with the same content
  produces no downstream signal. The Redis mirror can treat writes
  as safely retryable.
- **Backpressure signals.** When our producer pipeline is more than 30
  minutes behind SLA, we page our on-call (not platform's) and open a
  thread in `#feature-platform` within 15 minutes of the detection.
- **Storage budget.** We stay within <!-- research-note: confirm
  current TB allocation with the storage platform team before signing
  --> TB in the partition root; overages require a written request
  through the platform's capacity process.

What Platform owes back:

- **Online-serving latency.** Redis read p99 <= 8 ms, 99.9% of the
  time, over any rolling 30-day window. If Redis is slow and our
  offline SLO is being met, the escalation is Platform's, not SRDE's.
- **Durable backup.** Partitions are retained in cold storage for the
  retention window agreed in §4.3 (365 days), independent of SRDE's
  warm-storage decisions. SRDE never deletes a partition without a
  Platform-acknowledged tombstone.
- **One pager.** A single on-call rotation at Platform covers
  feature-store incidents, reachable at `#feature-platform-oncall`.

### 4.3 Legal / Privacy (partner: `privacy-engineering` team)

Counter-party signoff: `signoff/legal-privacy.md`.

What SRDE owes:

- **GDPR Article 17 (right to erasure) re-emission.** When Privacy
  Engineering delivers a verified erasure request, SRDE re-emits the
  affected feature partitions within 10 business days of receipt.
  The 10-day window is a ceiling; most erasures complete inside the
  next scheduled build. <!-- research-note: confirm legal's internal
  SLA promises the regulator; we do not cite the GDPR deadline to
  avoid stating a number we have not verified -->.
- **PII derivation boundary.** No raw PII leaves SRDE pipelines in
  `features.search_reranker_v3`. Derived aggregates (counts,
  frequencies, log-odds) are allowed; raw query strings, user ids in
  the clear, and IP addresses are not. The dataset card (mod-112 ex-02)
  lists every PII-derived column.
- **Annex XI readiness.** We maintain the training-data summary from
  mod-112 ex-03 as a live document, re-generated on every production
  model release. "Live" means: the auditable version is never more
  than one production release behind.

What Legal/Privacy owes back:

- **A single intake channel** for erasure and access requests, with a
  machine-readable payload (user_id, affected_time_range). Verbal
  "please drop this user's data" requests are not actionable.
- **Advance review** of new PII-derived features at least 5 business
  days before SRDE ships them.
- **Retention policy authority.** Legal defines the retention window
  for raw query logs; SRDE implements. We do not debate the number in
  a review meeting; if it changes, Legal publishes an updated policy
  and SRDE schedules the backfill.

### 4.4 Product Analytics (partner: `search-analytics` team)

Counter-party signoff: `signoff/product-analytics.md`.

What SRDE owes:

- **Vocabulary stability.** Changes to `cat_vertical` / `cat_device` /
  `cat_locale` / `cat_query_class` vocabularies are landed only after
  Analytics has reviewed the historical re-coding plan.
- **Backfill parity.** When we rebuild historical partitions, Analytics
  gets a diff report: row counts, class balance, distribution digests
  per slice. This is the input to mod-112 ex-04's slice table.
- **Row-count honesty.** Reported row counts in the dataset card and
  the model card match the table's `COUNT(*)`, not an estimate. Any
  sampling is labelled.

What Analytics owes back:

- **Taxonomy ownership.** When a new vertical opens (e.g., a new
  commerce category), Analytics publishes the canonical vocabulary
  entry; SRDE codes it.
- **Experimentation metadata.** When a live A/B test changes
  click-stream distributions, Analytics flags the experiment id and
  the time window so SRDE can slice the training table accordingly
  and the drift SLIs in mod-111 don't false-fire.
- **KPI definitions.** The company-level search KPIs (CTR, abandon
  rate, satisfaction) are defined in Analytics' metric catalogue.
  SRDE's KPIs in §5 reference, not redefine, those metrics.

## 5. KPIs — the small set we are judged on

We pick six, not twenty. Each KPI has: a numerical target, a cadence,
and the dashboard or query that produces it. Where we do not yet have
a committed number, we say so.

| # | KPI                                                            | Target                            | Cadence   | Source dashboard / query                                              |
|---|----------------------------------------------------------------|-----------------------------------|-----------|-----------------------------------------------------------------------|
| 1 | Daily partition freshness (feature-table SLA hit rate)         | >= 95% days, 30d rolling          | Weekly    | `dash.srde.freshness` (built on mod-111 ex-01 monitoring contract)    |
| 2 | Monitoring-contract breach false-positive rate                 | <= 10% of breaches post-mortem    | Monthly   | `dash.srde.breach-triage` (post-mortem outcomes from mod-111 ex-03)   |
| 3 | Time-to-re-emit on GDPR erasure (p95)                          | <= 7 business days                | Monthly   | `query.privacy.erasure-latency` <!-- research-note: confirm Legal's current ask; placeholder --> |
| 4 | Annex XI summary staleness                                     | <= 1 production model release     | Per-release | `dash.srde.annex-xi-freshness` (built on mod-112 ex-03 output)       |
| 5 | Model-card slice-table completeness (slices with row-counts)   | 100% of slices named in ex-04     | Per-release | `dash.srde.slice-table-coverage`                                    |
| 6 | Vendor-labelling audit cadence met                             | 1 audit per quarter, no misses    | Quarterly | `query.srde.vendor-audit-register` (register from mod-107 ex-02)      |

Why six and not twenty — see §10.1 (design rationale).

What we deliberately do **not** make a KPI for SRDE:

- Search click-through rate. That is an Analytics KPI; making it an
  SRDE KPI invites the wrong optimisation (features that look good on
  CTR in the short run but corrupt the training distribution).
- Model offline AUC or NDCG. That is an ML Training KPI. SRDE owning
  it would blur the data/model line we drew in §3.
- Redis read p99. Platform's, per §4.2.

## 6. Escalation paths

Escalation is role-based and time-tiered. Humans are never named.

| Tier | Trigger                                                                 | Role on point (ack within) | Decision authority         |
|------|-------------------------------------------------------------------------|----------------------------|----------------------------|
| T0   | Monitoring contract breach, page fired                                  | SRDE on-call (15 min)      | SRDE on-call               |
| T1   | T0 not resolved within 2 hours, or breach is schema-level               | SRDE Tech Lead (30 min)    | SRDE Tech Lead             |
| T2   | Partner-team SLA miss, or cross-team dispute                            | SRDE Engineering Manager (1 business hour) | SRDE EM + peer EM |
| T3   | Charter-level conflict (decision-right ambiguity, non-goal challenged)  | Director of Data (same day) | Director of Data          |
| T4   | Policy-level conflict (retention, PII boundary, Annex XI exposure)      | VP of Engineering (next business day) | VP of Engineering |

Rules:

- A tier only escalates once the current tier has acknowledged, not
  resolved. "I am on it" counts; silence does not.
- Legal/Privacy escalations use a parallel path: any PII-touching
  incident goes to the Legal/Privacy Lead in parallel to the SRDE
  path, at the same tier.
- An escalation never skips a tier downward. "The VP will decide" is
  not a substitute for an SRDE Tech Lead decision.

## 7. Operating cadences

- **On-call.** One SRDE engineer on call per week, following-the-sun
  not attempted at current team size. Pager eligibility requires
  completion of the mod-111 ex-03 rollback runbook walkthrough.
- **Standup.** 15 minutes, 4 days a week (no standup on Fridays to
  protect deep-work time). Focus: blockers and partner-team pings
  only; status updates go to the written channel.
- **Weekly partner sync.** 30 minutes with ML Training. 30 minutes
  biweekly with Platform. Analytics and Legal/Privacy are on-demand
  but with a standing invite.
- **Monthly review.** 60 minutes. Walk the six KPIs. Breach triage.
  Charter-amendment candidates.
- **Quarterly roadmap.** One day off-site (or equivalent). Vendor
  audit (mod-107 ex-02). Annex XI dry-run (mod-112 ex-03). Charter
  non-goals review.
- **Annual charter revision.** Mandatory. The charter's
  `charter_review_due` is one year after `charter_effective`. If the
  review slips, the charter is still in force but the review slip is
  itself a §6 T3 escalation.

## 8. Non-goals (what we refuse)

The charter is as much about refusal as about commitment. SRDE will
not:

- Own features for retrieval stages other than the re-ranker. If the
  retrieval team needs features, they own the pipeline, even if it
  touches query logs. We will advise; we will not operate.
- Own dashboards for non-SRDE KPIs. The six KPIs in §5 are ours; CTR
  dashboards, model-quality dashboards, and infra latency dashboards
  belong to their owning teams.
- Accept verbal SLA requests. Every SLA is in §4 or is not an SLA.
- Accept "please just add a column" requests outside the §4.1
  lead-time commitment. Lead time is a feature, not a bug.
- Carry a pager for the online read path. If Redis is slow, Platform
  pages. See §4.2.
- Train or evaluate models. We provide data; the model team provides
  models. An SRDE engineer may run a sanity notebook; that is not the
  same as owning a model.
- Take on secondary PII sources without a Legal/Privacy sign-off
  (§3 table, retention row).

## 9. Change control

The charter is a versioned document. The amendment process:

1. Any role listed in `countersigned_by` may propose an amendment via
   a dated entry in `changelog.md` and a PR against `CHARTER.md`.
2. The PR must list the affected sections and, if a decision right or
   an SLA is changing, the counter-party signoff file to update.
3. Approval requires the role marked **A** in the §3 RACI row
   "Amend this charter" — currently the VP of Engineering — plus
   written sign-off from every affected partner's lead.
4. On approval, a release script:
   - Bumps `charter_version` (semantic: major for scope / decision
     rights changes, minor for SLA numbers, patch for editorial).
   - Computes `charter_sha256` over the body of `CHARTER.md` with the
     `charter_sha256` field zeroed, and writes it back into the field.
   - Appends a dated entry to `changelog.md` with the new SHA and the
     amended sections.
5. A charter whose `charter_sha256` does not match a recomputation is
   treated as invalid; CI fails the next `make validate-charter`.

## 10. Design rationale (defends the hard decisions)

### 10.1 Why six KPIs, not more

A KPI is a thing the team optimises. Each additional KPI is an
additional axis the team is scored on, and axes compete. Six was
chosen because it covers the four stakeholders in §4 (ML Training,
Platform, Legal/Privacy, Analytics) plus the two foundational
operational outputs (freshness and breach quality). Adding a seventh
KPI — e.g., "feature coverage rate" — was rejected because it would
reward adding features regardless of whether the model team asked for
them; §4.1 already governs that via lead time, which is a workflow
rule, not a KPI. Adding "cost per partition" was rejected at charter
v1 because we do not yet have a stable cost-allocation dashboard
<!-- research-note: revisit at v1.1 once FinOps allocation is ready -->.

### 10.2 Why ML Training is A (accountable) for feature removal, not SRDE

The team that bears the consequences of a decision should own the
decision. If SRDE unilaterally removes a feature and the model's
offline AUC drops 2 points, SRDE learns about it from a complaint, not
from its own KPIs. Giving ML Training the **A** on removal means the
consequence-bearer has the final say, and SRDE's job is to propose,
cost, and execute. This is the hardest cell in the RACI because at
first glance it looks like SRDE is giving up power over its own
table — it is not; SRDE still owns schema stability (§4.1) and can
still decline to add a feature in the first place. What SRDE gives up
is the option to delete surprise.

### 10.3 Why the charter never names humans

Charters that name people decay the day a person changes role. Worse,
a named-human charter creates an implicit hierarchy that the team
cannot change without an emotionally charged amendment. Role-based
charters survive promotions, rotations, parental leave, and
re-orgs. The sign-off file in `signoff/` records the human who signed
*at this version*, which is the right place for that information —
historical, auditable, not operational. The distinction is lifted from
the NIST AI RMF GOVERN guidance that organisational risk ownership
lives with roles, not individuals.

### 10.4 Why vendor labelling is in scope at all

A data-engineering team that does not own its labelling supply chain
is exposed to OWASP ML06 — AI Supply Chain Attacks — through its
vendors. Our `cat_query_class` relabelling is a quarterly audit
specifically because an attacker who corrupts the vendor's labelling
workflow has, in effect, inserted ML02-flavoured poison into our
training distribution via the label column. The vendor-audit KPI in
§5 is the charter's commitment to maintain the OWASP ML06 defence
posture. (MITRE ATLAS's mitigation side covers this under the general
supply-chain mitigation family; we cite ATLAS at the framework level
rather than guess specific mitigation IDs.)

## 11. Sign-off

> By countersigning this charter, each role affirms that the
> commitments made to and from their team in §4, and the decision
> rights in §3, reflect their team's understanding as of
> `charter_effective`. Sign-off files live in `signoff/` and contain
> the role name, the version signed, the date, and a free-text field
> for reservations.

```
charter_sha256: <filled by release script>
charter_version: 1.0.0
charter_effective: 2026-12-01
```

countersigned_by:
- VP of Engineering                       signoff/vp-engineering.md
- ML Training Lead                        signoff/ml-training.md
- Platform / Infra Lead                   signoff/platform-infra.md
- Legal / Privacy Lead                    signoff/legal-privacy.md
- Product Analytics Lead                  signoff/product-analytics.md
```

### 2.4 How this charter chains into the rest of mod-112

- **mod-112 ex-02 (dataset card, three formats).** The charter obligates
  SRDE to publish the dataset card for `features.search_reranker_v3`
  (scope §2.1). The dataset card is the public artifact; the charter
  is the private contract that says the dataset card will exist and
  who signs off on it.
- **mod-112 ex-03 (Annex XI training-data summary).** KPI #4 and SLA
  §4.3 obligate SRDE to keep the Annex XI summary within one
  production-release of current. The Annex XI document inherits its
  "what dataset, what provenance" claims from this charter's §2.1.
- **mod-112 ex-04 (model-card data slice authoring).** Scope §2.1 and
  KPI #5 obligate SRDE to produce the slice tables the model card
  consumes. Slices are per-locale, per-device, per-vertical as named
  in the dataset schema (mod-111 ex-01).
- **mod-112 ex-05 (certifications portfolio planner).** The charter
  is the "GOVERN" artifact a certifications audit will reference
  first, because it declares roles and responsibilities. The
  certifications plan will want to cite this charter by SHA.

### 2.5 How this charter chains into sibling modules

- **mod-111 ex-01 (dataset monitoring contract).** The charter's §4.1
  freshness SLA is operationalised by that monitoring contract. The
  monitoring contract's SLOs are the technical definition of this
  charter's "95% days, 30d rolling".
- **mod-109 ex-01 (versioning backbone).** The charter's §4.1
  reproducibility commitment ("any partition the model team trained
  on in the last 365 days is recoverable") is implemented against
  whichever versioning system (DVC / lakeFS / Pachyderm) that
  exercise selected.
- **mod-107 ex-02 (vendor RFP and audit).** The charter's §5 KPI #6
  and §10.4 rationale obligate SRDE to re-run that vendor audit
  annually at minimum, quarterly for the current scope.
- **mod-110 (PII and privacy risk).** The charter's §4.3 SLAs are the
  team-level commitments that implement the policy-level PII handling
  decisions made in mod-110.

## 3. Validation steps

A reviewer can verify this charter without talking to the team. The
checks are mechanical.

1. **Required-section presence.** Grep `CHARTER.md` for the ATX
   headings `^## 1\. Mission`, `^## 2\. Scope`, `^## 3\. Decision
   rights`, `^## 4\. Partner-team SLAs`, `^## 5\. KPIs`, `^## 6\.
   Escalation paths`, `^## 7\. Operating cadences`, `^## 8\.
   Non-goals`, `^## 9\. Change control`, `^## 10\. Design rationale`,
   `^## 11\. Sign-off`. Missing section = fail.
2. **Markdown lint.** Run `markdownlint-cli2 CHARTER.md`. No trailing
   whitespace, no tabs, bullets use `-`, headings ATX, fenced code
   blocks have language hints (`markdown`, `text`, or none for plain
   fences). Any lint error with severity `error` fails.
3. **No human names.** `grep -E '\b[A-Z][a-z]+\s+[A-Z][a-z]+\b'
   CHARTER.md | grep -vE '(ML Training|Legal/Privacy|Product
   Analytics|Platform/Infra|Annex XI|OWASP ML|MITRE ATLAS|NIST AI|VP
   of Engineering|Hugging Face)' | grep -v '^$'`. Any proper-name
   pair that is not on the allowlist is a fail. The allowlist is
   itself in the repo so the check is auditable.
4. **Every SLA has a counter-party signoff file.** For each SLA in
   §4, grep for `Counter-party signoff:` followed by a path that
   matches `signoff/*.md`. Missing path = fail. Check that the file
   exists in the working tree. Missing file = fail.
5. **Every KPI has a source dashboard pointer.** Parse the §5 table;
   every row's "Source dashboard / query" cell must be non-empty and
   must match either `dash.srde.*` or `query.*` or begin with
   `<!-- research-note:` (which is a soft fail, not a hard fail — it
   blocks auto-merge but is acceptable in v1).
6. **Every KPI has a numerical target.** Parse the §5 table; every
   row's "Target" cell must contain either a number, a percentage, or
   an explicit `<!-- research-note: -->` marker. "Good" or "high" is
   a fail.
7. **RACI table has exactly one A per row.** Parse the §3 table; for
   each row, count the number of cells containing `A` exactly (word
   boundary). Must be exactly 1.
8. **RACI columns match a sign-off role.** For each column header in
   §3, verify that either (a) the role is on the SRDE side (EM, Tech
   Lead) or (b) the role appears in `countersigned_by`. A column that
   neither signs nor sits inside SRDE is dead weight.
9. **Partner-team count.** §4 must list at least 4 partner teams.
   Count the `### 4\.\d+` headings; fewer than 4 is a fail.
10. **Non-goals count.** §8 must list at least 5 refusals. Count
    bullet lines after the `## 8\. Non-goals` heading; fewer than 5
    is a fail.
11. **Change control is self-consistent.** §9 must specify a role
    marked **A** on the "Amend this charter" RACI row (§3) that
    matches the role the sign-off script writes. If the RACI says
    **A = VP of Engineering** but §9 says the EM amends, fail.
12. **`charter_sha256` self-consistency.** If `charter_sha256` is
    non-null, recompute SHA-256 over `CHARTER.md` with the
    `charter_sha256` field value replaced by `null`. The recomputed
    SHA must equal the field. Mismatch = fail. A `null` SHA is
    acceptable only during pre-signoff.
13. **`research-note` markers block auto-merge.** Count
    `<!-- research-note:` occurrences. If > 0, CI should mark the
    PR as "requires human merge" rather than auto-merging on green.

A sample `Makefile` target for #1-#13 lives in the sibling exercise
`mod-111 ex-05` conventions; adapt for charters.

## 4. Rubric / review checklist

Each row is scored **0 / 1 / 2**: 0 = missing or wrong, 1 = present
but thin or ambiguous, 2 = specific and defensible. Rows marked
**[critical]** must score at least 1; a 0 on any critical row is an
automatic fail regardless of total. Pass threshold: **>= 30 / 40**
and no critical failure.

| #  | Criterion                                                                                              | Max | Critical |
|----|--------------------------------------------------------------------------------------------------------|-----|----------|
| 1  | Mission is one or two sentences, names the product surface, and names a measurable outcome            | 2   | yes      |
| 2  | Scope §2.1 lists the dataset / pipeline / adjacent artifacts in scope with concrete names             | 2   | yes      |
| 3  | Scope §2.2 lists out-of-scope with concrete names (not just "model stuff")                            | 2   | yes      |
| 4  | Scope §2.3 lists at least three edge cases with resolution                                            | 2   | no       |
| 5  | Decision rights §3 is a RACI table (not prose), with exactly one A per row                            | 2   | yes      |
| 6  | §3 covers schema change, dataset deprecation, vendor contracts, retention, release gating             | 2   | yes      |
| 7  | Partner-team SLAs §4 lists at least four partner teams                                                 | 2   | yes      |
| 8  | Every §4 SLA is bidirectional (SRDE owes, partner owes back)                                           | 2   | yes      |
| 9  | Every §4 SLA names a counter-party signoff file                                                       | 2   | no       |
| 10 | KPIs §5 lists four to six, each with a numerical target and a dashboard pointer                       | 2   | yes      |
| 11 | KPIs §5 explicitly names at least two metrics the team does NOT own                                   | 2   | no       |
| 12 | Escalation paths §6 are role-based, time-tiered, with ack windows                                      | 2   | yes      |
| 13 | Cadences §7 cover on-call, standup, monthly review, quarterly, annual                                 | 2   | no       |
| 14 | Non-goals §8 lists at least five refusals                                                              | 2   | yes      |
| 15 | Change control §9 specifies SHA recomputation and version bump rules                                   | 2   | no       |
| 16 | Design rationale §10 defends at least three of the hardest decisions                                   | 2   | yes      |
| 17 | Sign-off §11 lists at least four countersigning roles (not humans)                                     | 2   | yes      |
| 18 | No human names anywhere in the charter body                                                            | 2   | yes      |
| 19 | Governance anchors (OWASP ML Top 10, MITRE ATLAS, NIST AI RMF) are named by framework / function     | 2   | no       |
| 20 | `research-note` markers are explicit and bounded (no stealth placeholders like "TBD" or "?")        | 2   | no       |

Maximum score: **40**. Pass threshold: **30** and zero critical
failures.

## 5. Common mistakes

Twelve failure modes to avoid; most are charters that technically have
all the headings but do not actually commit to anything.

1. **Mission is aspirational.** "SRDE aims to enable world-class
   search." *Why this bites:* un-measurable missions make the §5 KPIs
   feel disconnected. A reader cannot tell whether the KPIs are the
   right KPIs for the mission, because the mission does not name an
   outcome. Fix: tie the mission to a concrete artifact and a
   concrete partner team.
2. **In-scope section has no edges.** "We own the data pipelines."
   *Why this bites:* under pressure, every team's definition of "the
   data pipelines" expands to whatever is most urgent that quarter.
   Edges — explicit out-of-scope and explicit edge cases — are what
   make a scope statement enforceable six months later. Fix: write
   §2.2 and §2.3 before you write §2.1.
3. **RACI has multiple A's on a row.** "VP of Engineering and ML
   Training Lead are both A on model release." *Why this bites:*
   dual accountability is no accountability; when the decision gets
   stuck, neither side owns the escalation. Fix: force exactly one A
   per row. If you cannot pick, the decision is not well-formed.
4. **SLAs are unidirectional.** "ML Training expects a 06:00 UTC
   partition." *Why this bites:* unidirectional SLAs are a wish list,
   not a contract. The partner team can drop a surprise feature
   request on Thursday and expect delivery Friday, because the
   charter does not say they owe lead time. Fix: every §4 subsection
   has both halves.
5. **KPIs that the team cannot move.** "Daily search CTR >= 10%."
   *Why this bites:* data-engineering actions do not move CTR
   directly; model, product, and UX do. Making CTR a KPI invites
   incentive misalignment (rushing a feature that looks good on CTR
   but corrupts the training mix). Fix: pick KPIs the team's own
   decisions move.
6. **No numerical target on a KPI.** "Freshness: high." *Why this
   bites:* every quarterly review becomes a debate about what "high"
   means. Fix: a number with a cadence, or a `research-note` marker
   that acknowledges the number is pending.
7. **Escalation paths that name humans.** "Escalate to Alice." *Why
   this bites:* Alice changes roles; the charter is now stale.
   Charters with human names rot faster than charters without. Fix:
   role names everywhere, sign-off files store the human-of-record.
8. **No non-goals.** The charter lists what the team does but not
   what the team refuses. *Why this bites:* in the absence of
   explicit refusal, the team accumulates ownership of every
   unassigned problem adjacent to its surface. In 12 months the team
   is a "data landfill" that cannot deliver on its real mission.
   Fix: §8 is as important as §2.
9. **No change-control mechanism.** The charter exists and is never
   amended. *Why this bites:* reality drifts; the charter does not.
   Within a year the on-record charter does not describe what the
   team does, and nobody treats it as authoritative. Fix: §9 forces
   amendments to happen on a visible track, with a SHA and a
   changelog entry.
10. **Governance framing is name-dropping.** "We align with NIST AI
    RMF and OWASP ML Top 10." *Why this bites:* without naming the
    specific function (GOVERN, MAP) or anchor (ML02, ML06), the
    reference carries no obligation. An auditor cannot verify what
    "align" means. Fix: cite by top-level function name and
    explicitly map a charter section to it (§10.4 does this for
    ML06).
11. **Dates and numbers that do not match the rest of the org's
    reality.** "95% freshness" when the current state is 80%. *Why
    this bites:* an aspirational SLO treated as an SLA is a trap. The
    first quarterly review is a crisis. Fix: pick targets that are
    (a) meetable today or (b) explicitly phased with a ramp. If you
    do not know current state, mark it `research-note`.
12. **Treating the charter as a one-shot write.** Signed, published,
    forgotten. *Why this bites:* the charter is the GOVERN artifact
    the rest of your compliance posture hangs off (dataset card,
    Annex XI summary, certification plan). If the charter decays,
    those artifacts inherit stale assumptions and the next audit
    becomes expensive. Fix: §7's annual revision and §9's amendment
    track are mandatory, not decorative.

## 6. References

External authorities — URLs verified at the framework level, not
sub-item level:

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — <https://atlas.mitre.org/>
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>
- EU AI Act (official text portal maintained by the European Commission)
  — the Annex XI training-data summary requirement is the anchor for
  §4.3 and KPI #4; the specific legal text is cited by the sibling
  mod-112 ex-03 solution.
- GDPR Article 17 (right to erasure) — cited at the Article level only
  in §4.3; specific deadline numbers are left to Legal/Privacy policy,
  not this charter.

Charter / RACI / DACI / SLA background. These are practitioner
concepts rather than single-source standards; we list the names
without inventing URLs:

- RACI responsibility-assignment matrix — standard reference material
  in the Business Analysis Body of Knowledge (BABOK) and in most
  project-management texts.
- DACI (Driver / Approver / Contributor / Informed) — a related
  decision-making matrix popularised in product-management practice;
  mentioned for contrast but not used here.
- SLA / SLO / SLI terminology — the Google SRE book's chapters on
  service-level objectives are the canonical practitioner reference
  for the SLA vs. SLO vs. SLI distinction used in §4.
- Team charters as an organisational-design artifact — covered in
  general-management texts; we do not cite a single canonical source
  because the practice is older and more diffuse than any single
  book.

Local anchors — sibling exercises this charter chains into or depends
on:

- `../exercise-02-dataset-card-across-three-formats/` — the dataset
  card this charter obligates SRDE to publish.
- `../exercise-03-eu-ai-act-annex-xi-training-data-summary-draft/` —
  the Annex XI training-data summary this charter obligates SRDE to
  keep current.
- `../exercise-04-model-card-data-slice-authoring/` — the model-card
  slice tables this charter obligates SRDE to produce.
- `../exercise-05-certifications-portfolio-planner/` — the
  certifications plan that will cite this charter by SHA.
- `../../mod-111-data-quality-monitoring-and-regression/exercise-01-dataset-monitoring-contract-for-one-pipeline/`
  — the monitoring contract that operationalises §4.1's freshness
  SLA.
- `../../mod-111-data-quality-monitoring-and-regression/exercise-03-schema-drift-alert-and-rollback-workflow/`
  — the rollback workflow required for on-call eligibility in §7.
- `../../mod-109-dataset-versioning-lineage-reproducibility/exercise-01-dvc-vs-lakefs-vs-pachyderm-on-one-dataset/`
  — the versioning backbone §4.1's reproducibility clause runs on.
- `../../mod-107-labeling-operations-and-annotation-programs/exercise-02-vendor-rfp-and-vendor-audit-authoring/`
  — the vendor audit KPI #6 obligates SRDE to re-run.
- `../../mod-110-pii-and-privacy-risk-for-training-data/` — the
  policy layer §4.3's SLAs implement.
