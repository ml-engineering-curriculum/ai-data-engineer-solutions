# Exercise 03 — EU AI Act Annex XI Training Data Summary Draft (Solution)

## 1. Solution overview

This is a **design + documentation** exercise. The learner drafts the
training-data portion of a public disclosure that a provider of a
general-purpose AI (GPAI) model would publish under the EU AI Act.
The deliverable is a single markdown artifact —
`training-data-summary.md` — plus a short design-rationale section
defending the hardest disclosure calls. The point is to produce a
format that a real provider could adapt, not to make a real
disclosure for a real model.

**Legal anchor.** The EU AI Act is Regulation (EU) 2024/1689. The
public summary of content used to train a GPAI model is required
under **Article 53(1)(d)** (the "sufficiently detailed summary"
obligation). The AI Office published a **Commission template** for
that public summary in 2025, which providers are expected to use as
the common structure. GPAI-model obligations under the Act began
applying on **2 August 2025**. This exercise's deliverable targets
that Article 53(1)(d) summary.

**Annex XI is a different document.** Annex XI of the Act is
titled "technical documentation for providers of general-purpose
AI models" — it lists the regulator-facing technical information a
provider keeps and makes available to competent authorities.
Annex XI is **not** the public training-data summary template;
the training-data content in the two artifacts overlaps but is
not identical (Annex XI goes deeper on model architecture,
evaluation, and compute). We are writing the **public** training-
data summary, and we treat Annex XI as the regulator-facing
container that cites this public summary as one of its sections.
A good solution calls that distinction out in the opening
paragraph of the drafted artifact so reviewers see the learner
understands which document they are writing.
<!-- research-note: exact Annex XI section numbering and the AI Office template section headings. Do not invent them in a reviewed artifact. -->

**Governance framing.** The EU AI Act is the primary legal anchor.
Three secondary frameworks shape the drafting discipline:

- **NIST AI Risk Management Framework 1.0** — the MAP function
  covers data provenance and intended use (exactly what a
  training-data summary documents); GOVERN covers the
  organisational discipline of keeping the summary current. We
  cite top-level NIST functions, not sub-item numbers.
- **OWASP Machine Learning Security Top 10 (2023)** — the public
  summary is the artifact that forces honesty about **ML02 Data
  Poisoning** (who had write access to training shards) and
  **ML06 AI Supply Chain** (which vendors, licensed sets, crawl
  operators). A summary that papers over these is a drafting
  failure.
- **MITRE ATLAS** — publicly disclosing training-data categories
  is itself an attacker aid (prompt-injection shaping, poisoning,
  membership inference). We name the tension in §10 of the drafted
  artifact rather than hide it.

**Scope boundary.** This is a DRAFT for curriculum purposes. A real
provider's Article 53(1)(d) summary would be reviewed by qualified
legal counsel, by trust-and-safety, by privacy, and by executive
sign-off before publication. Nothing in this solution is legal
advice, and the learner should not copy it into production without
that review loop. We say so in the artifact itself (see §2.1 of the
drafted document) because reviewers of real summaries expect that
disclaimer.

**What success looks like.** A reviewer should be able to read the
drafted `training-data-summary.md` and answer without follow-up:
(a) which model and when was it released, (b) which categories of
data and in roughly what proportions, (c) what legal basis per
category, (d) how rightsholder opt-outs (TDM reservations under
Article 4 of Directive 2019/790) were respected, (e) PII posture,
(f) contamination controls, (g) what was explicitly withheld and
why, and (h) who to contact as a rightsholder. If any of those
eight questions is unanswered, the draft is not ready for counsel.

## 2. Worked answer or implementation

### 2.1 Chosen model

We pick one concrete, named model and stay on it throughout:

- **Model name:** `acme-text-base-7b`
- **Provider:** Acme Research (fictional; used for illustration
  only)
- **Architecture family:** decoder-only transformer, 7B parameters
- **Modality:** text in, text out
- **Release channels:** open weights on a model hub; also exposed
  via a hosted inference endpoint by the provider
- **Training-data mixture (headline):** filtered public web crawl +
  code + academic papers + licensed books + a small synthetic
  mathematics set

**Illustrative only.** Numbers are not real benchmarks; the
provider does not exist. Treat them as plausible placeholders that
a real provider would replace with audited figures. Inline notes
mark fields that would need real-world sourcing.

### 2.2 Project layout

```
exercise-03/
├── README.md
├── training-data-summary.md          # the main deliverable
├── design-rationale.md               # defends the hardest calls
├── contacts.yaml                     # machine-readable contact block
├── evidence/                         # pointers only, not the data itself
│   ├── crawl-respect-metrics.md
│   ├── pii-redaction-pipeline.md
│   └── dataset-lineage-pins.yaml
└── tests/
    ├── test_markdown_lint.sh
    ├── test_required_sections.py
    └── test_no_human_names.py
```

This solution presents `training-data-summary.md` and
`design-rationale.md` inline as fenced blocks; other files are
machine artifacts or regulator-facing evidence pointers that
follow the same shape as the summary.

### 2.3 `training-data-summary.md` (full worked artifact)

```markdown
<!--
training-data-summary.md
Public summary of content used to train acme-text-base-7b.
Prepared under EU AI Act Regulation (EU) 2024/1689 Article 53(1)(d).
Illustrative DRAFT for curriculum purposes. Not legal advice.
-->

# Public summary of training data for `acme-text-base-7b`

**Document status:** DRAFT v0.3 — for internal review prior to
public release. This is an illustrative artifact produced for a
curriculum exercise; the model, provider, and numbers are fictional.
A real provider should have this reviewed by qualified legal
counsel, privacy, and trust-and-safety roles before publication.

**Legal basis for this document.** We publish this summary to meet
the obligation on providers of general-purpose AI models to make
publicly available a sufficiently detailed summary of the content
used to train the model, as set out in Regulation (EU) 2024/1689
(the EU AI Act) Article 53(1)(d). The structure of this document
follows the Commission template published by the AI Office for
providers to use.
<!-- research-note: cite the AI Office template release by exact title and date before publication. -->

**Scope.** This document is the public summary. It is a companion
to the technical documentation we maintain for competent
authorities under Annex XI of the same Regulation. The Annex XI
package is more detailed and is not public; where the two documents
overlap on training data, this summary is the canonical public
statement and the Annex XI version cross-references it.

**Version:** 0.3
**Date of this version:** 2026-11-26
**Planned update cadence:** at every material model release and at
least annually; a materially changed training-data mixture triggers
a new version with a changelog entry.
**Point of contact for rightsholders:** see §2.11.

---

## 1. Model identity

| Field | Value |
| --- | --- |
| Model name | `acme-text-base-7b` |
| Model version covered by this summary | `1.0.0` |
| Modality | Text in, text out |
| Architecture family | Decoder-only transformer, 7B parameters |
| Release date | 2026-10-14 |
| Release channels | Open weights on a model hub; hosted inference endpoint operated by the provider |
| Intended uses | General-purpose text generation and language understanding for downstream fine-tuning, research, and application-layer integration |
| Out-of-scope uses | Any high-risk use under Annex III of the EU AI Act without additional deployer-side controls; safety-critical autonomous decisions; use in jurisdictions where the deployer has not satisfied local legal review |
| Provider legal entity | Acme Research B.V. (illustrative) |
| GPAI model lifecycle stage | Placed on the EU market on 2026-10-14 |

The model is covered by the provider's internal data-governance
program; this summary names only what a public reader needs to
evaluate the training-data posture, not internal controls.

## 2. Narrative overview of the training-data lifecycle

The training corpus for `acme-text-base-7b` was assembled in four
phases over roughly nine months, with freezes at each phase gate so
that lineage and legal review could proceed against a stable
snapshot. Phase 1 was source acquisition — web crawl, licensed book
corpora, code hosting platforms, and open-access academic
repositories. Phase 2 was filtering and de-duplication, which
removed low-quality boilerplate, repeated passages, and documents
flagged by safety classifiers. Phase 3 was legal and rights review,
during which rightsholder opt-out signals were applied and licensed
content was reconciled against vendor manifests. Phase 4 was final
mixture assembly, pinning, and handoff to the training job.

Across the four phases, the corpus went through a factor-of-ten
reduction from raw acquired bytes to tokens that reached the model.
Most of that reduction was de-duplication and language filtering,
not safety redaction; safety redaction accounts for a small
single-digit percent of removed tokens. We publish this ratio
because readers of public summaries have historically been
misled about the share of "aggressive filtering" in modern web-scale
corpora, and we want to be honest that most of what is removed is
noise, not content.

Each phase produced a signed manifest. The manifest pins the source
URIs, the content-hash per document where available, the filter
versions applied, and the operator identity for each batch. The
manifests themselves are regulator-facing (Annex XI) and are not
published here, but the summary numbers below are derived from
them directly rather than from a point-in-time count.

Role-based reviews occurred at three points: rights-and-licensing
review before each licensed-data ingestion, a quarterly safety
spot-check on sampled crawl output, and a final-mix review before
training started. No individuals are identified.

The training-data corpus is **immutable** from the moment training
begins. If a rightsholder exercises an opt-out after training, the
remedy is prospective — we exclude their content from the next
training run and evaluate whether targeted unlearning or model
retirement is appropriate. The complaint intake is in §11.

## 3. Categories of data sources and approximate volumes

All volumes are **approximate** and expressed as token ranges
rather than exact counts. We disclose ranges rather than exact
counts because exact counts expose internal mixture weights that
have commercial and adversarial sensitivity; the ranges are tight
enough for a reader to understand the mixture while not disclosing
the mixture coefficients. The design rationale defending this
choice is in the Annex XI technical documentation.

| Category | Approx. share of training tokens | Legal basis claimed | Section |
| --- | --- | --- | --- |
| Publicly available online content via web crawl | 55-70% | Text-and-data-mining exception under Article 4 of Directive (EU) 2019/790, subject to rightsholder reservation | §3.1 |
| Licensed content (book corpora, news archives, specialist databases) | 10-20% | Written commercial licence from the rightsholder or aggregator | §3.2 |
| Publicly available open-access academic content | 5-10% | Open-access licence terms (predominantly Creative Commons variants); provenance retained per document | §3.3 |
| Publicly available code hosted on public repositories | 5-15% | Repository's declared licence per file; aggregate-level compliance review; permissive-licence-only subset for a specific slice | §3.4 |
| Synthetic data generated in-house | <5% | Generated by the provider; derivative-rights handling documented | §3.5 |
| Government and open-data sources | <2% | Public-sector reuse terms applicable to each source | §3.6 |
| User-contributed data from the hosted inference endpoint | 0% for this model version | n/a — no user-conversation data from the hosted endpoint was used to train `acme-text-base-7b v1.0.0` | §3.7 |

The seven categories below use a shared internal schema so a reader
can compare them. For each category we state: the source profile,
the approximate volume, the filtering pipeline applied, the legal
basis claimed, the rightsholder-reservation / opt-out handling, and
the content-moderation and PII controls.

### 3.1 Publicly available online content via web crawl

- **Source profile.** Pages reached by the provider's crawler and
  by a commercial web-archive partner; language coverage is
  predominantly English with secondary coverage of several European
  and East Asian languages.
- **Approximate volume.** On the order of several trillion tokens
  pre-filter; after de-duplication, language filtering, and safety
  filtering, the fraction that reaches the training mix is in the
  single-digit trillion range. We disclose the post-filter range
  rather than the exact token count.
- **Filtering pipeline.** Language identification, boilerplate and
  template removal, exact and near-duplicate deduplication,
  heuristic quality classifiers, a safety-classifier cascade, and
  a PII-pattern pass. Each filter is versioned and its version is
  recorded in the per-document manifest.
- **Legal basis.** We rely on the text-and-data-mining exception in
  Article 4 of Directive (EU) 2019/790 as transposed in the member
  states where our crawl operates, subject to rightsholders'
  express reservation of rights.
  <!-- research-note: the exact Article 4 reservation-expression requirements are an active legal question; confirm the specific national-law transposition the crawl is operated under before publication. -->
- **Rightsholder opt-out / TDM reservation handling.** We respect
  `robots.txt` directives for our crawler's declared user-agent;
  we respect site-level opt-out signals as expressed in the
  emerging machine-readable opt-out conventions (including
  `ai.txt`-style files and HTTP headers that assert TDM
  reservation); we additionally maintain a provider-level
  allowlist/blocklist that absorbs ad-hoc rightsholder requests.
  The specific recognised mechanism for expressing a reservation
  under the EU TDM framework is set by Article 4 of Directive
  2019/790; our compliance is reviewable.
  <!-- research-note: the exact set of machine-readable mechanisms recognised as sufficient to express a reservation is still being shaped; defer to counsel. -->
- **Respect metrics.** We publish aggregate metrics on how often
  our crawler encountered an opt-out and honoured it versus failed
  to honour it (for example, due to retrieval errors or latent
  propagation of a newly-added opt-out signal). For version 1.0.0
  of the model, those metrics are available at
  `evidence/crawl-respect-metrics.md`.
- **Content moderation.** The safety-classifier cascade removes
  documents flagged for child sexual abuse material (CSAM, hashed
  against known datasets by a specialised partner under contract),
  terrorism and violent-extremism content, and documents that
  exceed a threshold on a general-harm classifier. Removal is
  logged per-document in the manifest.
- **PII handling.** See §2.6.

### 3.2 Licensed content

- **Source profile.** Book corpora, news archives, and specialist
  databases obtained under written commercial licence. We do not
  disclose the specific vendors in this public summary — see §2.10
  for the design rationale defending that choice.
- **Approximate volume.** Low single-digit hundreds of billions of
  tokens pre-filter.
- **Filtering pipeline.** De-duplication against the web-crawl
  corpus to avoid double-counting; a PII-pattern pass; a
  licence-scope check that flags content whose licence does not
  extend to model-training use.
- **Legal basis.** Written licence from the rightsholder or
  aggregator covering training-data use, including sub-licensed
  rights where applicable.
- **Opt-out handling.** Where a licence permits a mid-term carve-
  out by the rightsholder, we honour it on next-training-run basis
  (as described in §2.11). For licences that are fully granted for
  the training-data use case, no in-term opt-out applies.
- **Content moderation.** Same classifier cascade as §3.1,
  configured with per-source thresholds.

### 3.3 Open-access academic content

- **Source profile.** Open-access papers and preprints from
  recognised repositories.
- **Approximate volume.** High tens of billions of tokens
  post-filter.
- **Filtering pipeline.** Licence detection per document (we only
  retain documents whose declared open-access licence permits
  training-data use); PDF text extraction; mathematical-notation
  normalisation.
- **Legal basis.** Open-access licence terms attached to each
  document (predominantly Creative Commons variants; also
  publisher-specific open-access terms).
- **Opt-out handling.** Where a repository exposes a per-document
  takedown signal, we honour it on next-training-run basis.

### 3.4 Publicly available code

- **Source profile.** Code repositories hosted on public code
  platforms.
- **Approximate volume.** Hundreds of billions of tokens
  post-filter.
- **Filtering pipeline.** Per-file licence detection; retention
  only of files whose declared licence is on our permissive
  allowlist; secret-scanning and credential redaction; aggressive
  de-duplication across repository forks.
- **Legal basis.** Per-file declared licence. A separate
  licence-compliance review is performed at the aggregate level
  and documented in the Annex XI package.
- **Opt-out handling.** We honour repository- and account-level
  opt-out requests; a user whose code was in a public repository
  at crawl time may request exclusion for subsequent training runs
  via the intake in §2.11.

### 3.5 Synthetic data

- **Source profile.** Generated in-house via earlier internal
  models and procedural generators, targeting mathematics, long-
  form reasoning, and structured-output tasks.
- **Approximate volume.** Low single-digit percent of training
  tokens.
- **Filtering pipeline.** Automatic correctness scoring where
  ground truth exists (e.g., arithmetic); deduplication against
  the rest of the mix.
- **Legal basis.** Provider-generated; derivative-rights handling
  documented internally. We disclose use and task categories but
  not the generator or procedural recipes (competitive
  sensitivity).
- **Known limitations.** Without ground truth (e.g., long-form
  reasoning) we rely on downstream evaluation; this is a residual
  quality risk.

### 3.6 Government and open-data sources

- **Source profile.** Statistical publications, legal and
  regulatory texts, and public-sector datasets released under
  reuse-permitting terms.
- **Approximate volume.** Low tens of billions of tokens.
- **Legal basis.** Public-sector reuse terms applicable to each
  source.
- **Opt-out handling.** Not applicable in the common case; where a
  source later retracts its reuse grant we exclude it on
  next-training-run basis.

### 3.7 User-contributed data

- **Status for this model version.** We did **not** use user
  conversations from the hosted inference endpoint to train
  `acme-text-base-7b v1.0.0`. If a future version changes this,
  that change will be reflected in an update to this summary and
  will be preceded by a product-surface disclosure and an
  opt-in/opt-out control that defaults to not using
  user-conversation data for training.

## 4. Content moderation and removal pipeline

The content-moderation pipeline is invoked at crawl time, at
pre-mixture time, and at post-mixture spot-check time. It applies
three layers:

- **Hash-based removal** for CSAM and known terrorist content,
  operated against externally maintained hash lists by a
  specialised partner under contract. Hits are removed and
  reported to the appropriate authority where required.
- **Classifier-based removal** for broader categories: violent
  extremism, hate content, sexualised-minor adjacent content,
  coordinated harassment content, and documents that exceed a
  threshold on a general-harm classifier.
- **Rule-based redaction** for specific patterns: credentials,
  API keys, long numeric identifiers that match PII patterns, and
  known phishing templates.

Each layer is versioned and recorded per document. Classifier
precision and recall on held-out sets are tracked internally but
withheld from this summary (adversarially sensitive — publishing
recall numbers tells an attacker where the gaps are); they are
reported inside the Annex XI package.

**Post-training removal.** For CSAM or similar, we escalate per
our incident-response process; for general content complaints we
exclude from the next training run.

## 5. Copyright and TDM-reservation respect protocol

We claim the text-and-data-mining exception under Article 4 of
Directive (EU) 2019/790 as the basis for crawling and extracting
features from publicly available online content, subject to the
rightsholder's express reservation of rights. In practice the
protocol is:

- **Signal ingestion.** The crawler ingests `robots.txt`,
  `ai.txt`-style machine-readable opt-outs, HTTP-header-level
  reservation signals, and provider-level allowlist/blocklist
  entries.
  <!-- research-note: the exact machine-readable mechanisms recognised as a reservation under Article 4 of Directive 2019/790 are still being clarified; the implementation surface is reviewable and will be updated. -->
- **Honour.** Signals are honoured at crawl time. A page flagged
  for reservation is not stored, not tokenised, and not propagated
  into the training mix.
- **Latency.** We measure the latency between a new opt-out being
  published and our crawler honouring it. The metric for model
  version 1.0.0 is available in the evidence pointer.
- **Review.** The protocol is reviewable: our Annex XI package
  contains the code paths, the signal-ingestion configuration, and
  the honour/miss metrics.
- **Complaint remedy.** A rightsholder who discovers their content
  is in the mix despite having asserted a reservation should
  contact us via §2.11. The remedy is exclusion on next training
  run, review of any residual trained model, and, where
  appropriate, a public update to this summary.

**Known limits.** We cannot retroactively excise content from an
already-trained model without retraining. We represent that we
honour reservations known at crawl time and that a complaint-
driven process handles missed cases going forward.

## 6. PII posture

The training-data PII posture is governed by the pipeline referenced
in mod-110 (`data-engineering/mod-110-pii-and-privacy-risk-for-training-data`).
Concretely:

- **Findings-rate SLO.** The per-document PII-pattern pass flags a
  defined set of pattern classes (email, phone, long numeric
  identifiers, structured credential patterns). The internal SLO
  on findings-rate stability is tracked per corpus slice and is
  not published here in numeric form; a non-published SLO is still
  a real SLO and a breach triggers an internal review.
- **Redaction pipeline.** Flagged spans are redacted with
  category-preserving placeholder tokens before the document
  enters the training mix. Low-risk patterns (e.g., public email
  addresses in open-source commit metadata) are handled per a
  documented policy, not silently retained.
- **Residual risk.** Pattern-based detection has known false-
  negative modes (names, quasi-identifiers, contextual leakage).
  The model may reproduce personal information from the training
  corpus, especially where already widely public. We do not claim
  this model is PII-free; the deployer is responsible for
  evaluating residual risk for their use case.
- **Right to object / erasure.** Persons wishing to object to
  processing or exercise erasure rights contact us via §11; we
  handle such requests under our published privacy notice.

## 7. Benchmark contamination mitigation

The mitigation stack follows the pattern described in mod-106
(`data-engineering/mod-106-eval-set-management-and-governance`):

- **Contamination registry.** A pinned list of evaluation sets we
  want to keep out of training. Each entry is a stable identifier
  plus a content fingerprint.
- **Decontamination pass.** Before the training mix is frozen, we
  run a decontamination pass that removes documents matching
  contamination-registry fingerprints by exact match and by
  near-duplicate near-neighbour detection.
- **Residual contamination estimate.** We measure residual
  contamination against a held-out sample of each evaluation set
  and report the estimate internally. We do not publish exact
  per-benchmark contamination percentages in this summary because
  the detailed numbers belong in the Annex XI evaluation section;
  we do confirm that a decontamination pass was run and that its
  residual estimates were reviewed before release.

## 8. Dataset lineage and version pins

Every training-data component is pinned by a signed manifest
(see mod-109, `data-engineering/mod-109-dataset-versioning-and-reproducibility-on-a-budget`).
The manifest records:

- A stable identifier for the source batch
- A content-hash per document where available
- The filter version applied at each phase gate
- The signing role that approved the batch

For `acme-text-base-7b v1.0.0`, the top-level manifest pin is
tracked in `evidence/dataset-lineage-pins.yaml`; it carries the
mix identifier and the per-category sub-manifest pins. The
regulator-facing Annex XI package contains the full manifest
chain.

## 9. Evaluation of training data quality

We evaluate training-data quality along four axes, drawing on the
disciplines in mod-104 (`data-engineering/mod-104-data-quality-and-schema-contracts`)
and mod-111 (`data-engineering/mod-111-data-quality-monitoring-and-regression`):

- **Schema conformance.** Every document that reaches training
  conforms to the internal document schema (language, declared
  encoding, structural tokens). Non-conformant documents are
  dropped at phase 2.
- **Drift against reference window.** For the web-crawl category,
  we measure drift between the current acquisition window and a
  pinned reference window for key subpopulation axes (language
  distribution, document-type distribution). Breaches require
  explicit sign-off before the batch is included.
- **Known-bias audit.** We audit the final mixture against
  subpopulation axes we can measure (language, publication era
  for licensed text, domain origin for crawl). We **do not**
  audit demographic axes we lack reliable measurement for — see
  §10.
- **Spot-check sampling.** Each phase ships a random sample to a
  role-based review queue, used to tune filter thresholds, not to
  make per-document decisions.

## 10. What we would NOT disclose, and why

The AI Act requires a summary that is **sufficiently detailed**
to allow interested parties to exercise their rights, while
permitting the provider to protect trade secrets. We make the
following non-disclosures explicitly rather than silently, so
readers know where the gaps are and what rationale we apply:

- **Exact mixture weights.** We disclose category ranges rather
  than exact mixture coefficients. Exact coefficients are a
  material commercial input that competitors would use directly
  and that an adversary could use to narrow a membership-inference
  attack.
- **Specific licensed vendors.** We disclose categories (book
  corpora, news archives, specialist databases) rather than the
  names of the specific vendors or the terms of the licences.
  Vendor-level disclosure exposes commercial terms that are
  typically covered by the licence confidentiality clauses and
  that can distort the licensing market for all providers.
- **Specific classifier thresholds and recall numbers.** We
  disclose the existence and purpose of each classifier but not
  its per-class recall or threshold. Those numbers are
  adversarially sensitive: publishing them tells a bad-faith
  content producer exactly where the gap is.
- **Generator recipes for synthetic data.** We disclose that
  synthetic data was used and the categories of tasks it covered;
  we do not disclose the generator or the procedural recipes,
  because those reveal how the model learned specific skills.
- **Named individuals.** No named individuals involved in dataset
  review or sign-off are listed. Named disclosure is neither
  required nor helpful; role-based accountability is covered in
  the Annex XI technical documentation.

Where we withhold a field, we mark it as "redacted — trade secret"
or "redacted — adversarially sensitive" rather than silently
omitting it, so a reader can see that a decision was made.

## 11. Point of contact for rightsholders and complaints

- **Rightsholder opt-out, complaints, erasure requests:**
  `rightsholders@acme-research.example` (illustrative address)
- **Security disclosure:** see the provider's security.txt
- **Press and general enquiries:** out of scope for this summary

Requests are acknowledged within a published business-day SLA. The
remedy pathway is described in §2.5 for TDM reservations and §2.6
for personal-data requests.

## 12. Document metadata

- **Version:** 0.3
- **Date:** 2026-11-26
- **Next planned update:** 2027-04-01 (annual cadence; materially
  changed training mixture triggers an off-cycle update)
- **Changelog pointer:** maintained in the repository at
  `CHANGELOG.md`
- **Legal review status:** DRAFT — pending counsel review
```

### 2.4 `design-rationale.md` (abbreviated)

```markdown
<!--
design-rationale.md
Short defence of the hardest disclosure decisions in
training-data-summary.md. Internal document; not published.
-->

# Design rationale for the training-data summary

This note defends the four hardest disclosure decisions in the
public summary. It is internal — its audience is counsel, the
drafting roles, and future versions of us.

## D1. Token counts as ranges, not exact numbers

**Decision.** Per-category shares as ranges (e.g. "55-70%");
volumes as order-of-magnitude ranges.

**Why.** Exact mixture coefficients are a material commercial
input and are adversarially useful (they narrow membership
inference). Ranges give a reader enough to understand the mix for
rights-exercise purposes without disclosing the coefficients.
**Risk.** A regulator could call a 15-point range not
"sufficiently detailed"; we mitigate by keeping ranges tight
enough to rank-order categories and by publishing the methodology.

## D2. Licensed vendors by category, not by name

**Decision.** Disclose category (book corpora, news archives,
specialist databases) but not vendor identities.

**Why.** Licence agreements typically carry confidentiality
clauses, and vendor-level disclosure distorts the licensing
market. The rightsholder-remedy pathway in §2.11 does not depend
on vendor names.
**Risk.** A rightsholder cannot verify directly; we mitigate via
the complaint intake, which checks against the manifest without
public vendor disclosure.

## D3. Publishing opt-out respect metrics, not just the policy

**Decision.** Publish aggregate metrics on how often the crawler
honoured a TDM reservation versus missed it, alongside the policy.

**Why.** Policy-only disclosures are the dominant failure mode of
training-data summaries industry-wide; they are low-cost to make
and low-information to read. Metrics let rightsholders and
regulators assess whether the policy actually works.
**Risk.** A low number can be used adversarially; we mitigate by
publishing the methodology so readers can judge the denominator.

## D4. Demographic composition — what we measured and what we didn't

**Decision.** State the subpopulation axes we measured (language,
publication era, domain origin for crawl); do not report
demographic axes for which we lack reliable measurement.

**Why.** Reporting a demographic breakdown we did not measure
reliably would mislead the reader. The honest answer is that
measurement is a capability gap.
**Risk.** Readers can interpret omission as evasion; we mitigate
by naming the gap in §2.9 and listing the axes we did and did not
measure.
```

### 2.5 `contacts.yaml` (sketch)

```yaml
# contacts.yaml — machine-readable contact block referenced by §11.
# Roles only. No human names.
rightsholder_intake:
  email: rightsholders@acme-research.example
  response_sla_business_days: 5
  remedy_pathway: exclude-from-next-training-run
privacy_requests:
  email: privacy@acme-research.example
  response_sla_business_days: 30
  legal_basis_reference: provider-privacy-notice
security_disclosure:
  url: https://acme-research.example/.well-known/security.txt
document_owner_role: head-of-model-governance    # role, not person
```

### 2.6 How the artifact connects to adjacent modules

- **mod-104 — Data quality and schema contracts.** Schema-
  conformance claims in §9 of the drafted artifact are the
  training-data analogue of mod-104 contracts.
- **mod-106 — Eval set management and governance.** The
  contamination registry referenced in §7 is authored in mod-106.
- **mod-109 — Dataset versioning and reproducibility on a
  budget.** Per-document manifests and the top-level mix pin in
  §8 are the lineage artifacts from mod-109.
- **mod-110 — PII and privacy risk for training data.** The PII
  posture in §6 is a public distillation of the pipeline, SLOs,
  and residual-risk statements authored in mod-110.
- **mod-111 — Data quality monitoring and regression.** The
  quality-evaluation claims in §9 are grounded in the monitoring
  contracts authored in mod-111.

### 2.7 Governance framing wrapped into the artifact

- **NIST AI RMF (MAP / GOVERN).** The summary is the public
  artifact in which a provider documents context and provenance;
  GOVERN covers keeping this document current.
- **OWASP ML Top 10 — ML02 Data Poisoning.** Lineage pins and
  per-batch signing are the public-facing side of the controls
  that prevent attacker-written shards from entering training.
- **OWASP ML Top 10 — ML06 AI Supply Chain.** Category-level
  disclosure of licensed and crawled sources is the supply-chain
  statement; a provider that cannot make it is signalling the
  supply chain is not well-understood internally either.
- **MITRE ATLAS tension.** The summary is also a potential
  attacker aid (prompt-injection shaping, membership inference).
  We accept the tension and withhold adversarially sensitive
  numbers while disclosing policy and category structure.

## 3. Validation steps

A drafted `training-data-summary.md` is ready for counsel review
when the following checks all pass. These are **gating** checks;
failing any one of them means the draft goes back to the drafting
role, not to counsel.

1. **Markdown lint clean.** Run `markdownlint-cli2` (or an
   equivalent) against the drafted artifact; no errors.
2. **Required sections present.** A structural check asserts
   that every section named in §2.3 is present in the draft
   (model identity through document metadata).
3. **Every category has a volume estimate or an explicit
   redaction tag.** The seven categories in §3 each carry a
   token-range disclosure or an explicit "redacted — trade
   secret" / "redacted — adversarially sensitive" tag. Silent
   omission is a failure.
4. **Every claimed opt-out mechanism has an implementation
   pointer.** For each mechanism listed in §5 (robots.txt,
   machine-readable opt-out files, HTTP-header signals, provider-
   level blocklist), the Annex XI package contains a pointer to
   the implementing code or configuration.
5. **Every third-party-licensed category has a legal-basis
   field.** No category rests on a generic "lawful basis"
   formulation.
6. **PII posture pointer exists.** §6 points to the mod-110
   pipeline artifact and names the residual-risk classes; "we
   handle PII" without a residual-risk acknowledgement fails.
7. **Lineage pins are non-placeholder.**
   `evidence/dataset-lineage-pins.yaml` has real pin values (no
   `TODO`, `xxxx`, `null`); top-level mix pin is present.
8. **Contact field is present and routed.** §11 names a mailbox
   monitored by a role (not a person), with a published SLA and
   remedy pathway.
9. **No named individuals.** A grep over the draft returns zero
   matches against known-name patterns; the draft is role-based.
10. **Every `research-note` marker is deliberate.** At counsel
    review time each marker is either resolved or explicitly
    deferred with sign-off.
11. **Changelog entry added.** The CHANGELOG entry for this
    version describes what changed and why; a version bump
    without a changelog entry fails.
12. **"Not legal advice" banner present.** The drafted artifact
    carries the status line in its header; a real release
    replaces it with a counsel-approved legal-review-complete
    status line.

## 4. Rubric or review checklist

Each row is scored **0, 1, or 2**. **0** = absent or wrong,
**1** = present but materially incomplete, **2** = present and
defensible for counsel review. Rows marked **critical** must score
**2** for the draft to pass. Target total: at least **28/40** with
no critical row below **2**.

| # | Criterion | Critical? | 0 | 1 | 2 |
| --- | --- | --- | --- | --- | --- |
| 1 | Distinguishes the Annex XI regulator-facing package from the Article 53(1)(d) public training-data summary | yes | Treats them as the same document | Mentions the distinction once but blurs it in the body | Names the distinction in the opening and consistently writes to the Article 53(1)(d) scope |
| 2 | Model identity block is complete (name, version, release date, intended uses, release channels, provider legal entity) | yes | Missing one or more fields | All fields present but some are vague | All fields present and specific |
| 3 | Narrative overview describes the training-data lifecycle in 3-6 paragraphs with phase gates and freeze points |  | Missing or single-paragraph | Present but no phase structure | Present with phase structure and freeze points named |
| 4 | Each data-source category has a filled volume field (range OR explicit redaction tag) | yes | One or more categories silently omit volume | All have a field but some are vague | All have a specific range or an explicit redaction tag |
| 5 | Each data-source category has a filled legal-basis field | yes | One or more categories have no legal basis | Legal basis stated but generic | Legal basis stated and specific to the category |
| 6 | TDM opt-out respect protocol is described with signal types and honour mechanism |  | Policy-only language, no mechanism | Mechanism named but no latency / respect-rate claims | Mechanism named and respect metrics referenced |
| 7 | TDM opt-out respect is quantified or honestly marked as unknown | yes | Silent on the respect rate | Claims respect without data | Either quantified, or marked "unknown — in progress" |
| 8 | CSAM and extremist-content removal is described without exposing classifier thresholds |  | Absent | Present but names thresholds (adversarially risky) | Present and withholds thresholds deliberately |
| 9 | PII posture names residual risk, not just controls | yes | Claims PII-free | Controls listed, residual risk silent | Controls listed and residual risk named |
| 10 | Benchmark contamination mitigation is referenced and tied to a registry | yes | Absent | Mentioned but no registry | Registry pinned (mod-106 anchor) |
| 11 | Dataset lineage pins are present and non-placeholder | yes | Placeholder values | Pins present but incomplete | Pins present and complete (mod-109 anchor) |
| 12 | Trade-secret redactions are explicit, not silent | yes | Silent omissions | Some explicit, some silent | Every withheld field carries an explicit redaction tag |
| 13 | Synthetic data category is disclosed with its purpose |  | Absent | Mentioned without purpose | Disclosed with task categories named |
| 14 | User-contributed-data posture is stated (even if "none") |  | Absent | Mentioned without clarity | Clear statement that includes future-change handling |
| 15 | Design rationale defends the hardest 3-4 decisions |  | Absent | One or two decisions defended | Three or four decisions defended with named tradeoffs |
| 16 | Contact for rightsholder complaints is a monitored mailbox with a published SLA | yes | Missing | Present but no SLA | Present with SLA and remedy pathway |
| 17 | Version, date, and update cadence are stated |  | Missing | Partial | All three present |
| 18 | No named individuals; role-based accountability | yes | Named individuals present | Mostly roles, one or two names slipped in | Role-based throughout |
| 19 | Governance framing (NIST AI RMF, OWASP ML Top 10, MITRE ATLAS) is named without inventing sub-numbers |  | Absent | Framing named but with invented sub-numbers | Framing named at top-level function / category |
| 20 | "Not legal advice / DRAFT" status is present at the top of the artifact | yes | Missing | Present but buried | Present in the header block |

## 5. Common mistakes

1. **Conflating Annex XI with the Article 53(1)(d) public
   summary.** Different artifacts, different audiences
   (regulator vs. public). Why this bites: a reviewer who sees
   "Annex XI" in the public summary's legal-basis line will
   question whether the provider understands which obligation
   they are meeting.
2. **Silent redactions.** Omitting a field because it is
   commercially sensitive without saying so. Why this bites: a
   reader cannot distinguish "we withheld this" from "we did not
   consider this"; the first is defensible, the second is not.
3. **Publishing classifier recall numbers.** Per-class recall
   values for safety classifiers tell bad-faith content producers
   exactly where the gap is. Why this bites: the summary becomes
   an attacker-aid against the provider's own safety pipeline.
4. **Policy-only opt-out disclosure.** Claiming `robots.txt` and
   TDM signals are honoured without publishing respect metrics.
   Why this bites: this is the dominant failure mode of public
   training-data summaries and regulators discount it.
5. **Claiming PII-free training data.** Pattern-based detection
   has known false-negative modes (names, quasi-identifiers,
   contextual leakage). Why this bites: the first time the model
   reproduces a phone number, the earlier claim becomes a finding.
6. **Named individuals in the artifact.** Why this bites:
   role-based accountability survives personnel changes and does
   not expose individuals to harassment.
7. **Invented article sub-points, template sections, or recital
   numbers.** Why this bites: a lawyer reviewing the draft catches
   invented citations immediately and loses trust in the whole
   artifact.
8. **Over-precise token counts.** Exact per-category counts
   disclose the mixture coefficients. Why this bites: the provider
   loses competitive and defensive ground to zero rights-exercise
   benefit for readers.
9. **"We did a decontamination pass" without a registry.** Why
   this bites: a reviewer who tries to reproduce the
   decontamination finds nothing to reproduce; the claim is
   vacuous.
10. **Missing or unroutable complaints contact.** Why this bites:
    the complaint pathway is the main remedy for rightsholders
    who find themselves in the mix despite an opt-out; a broken
    pathway invalidates the whole opt-out posture.
11. **No "what we did not disclose" section.** Why this bites:
    regulators and journalists read for what is missing; naming
    the withholds explicitly is more defensible than hoping no-one
    notices.
12. **Treating the draft as legal advice.** Why this bites: a
    drafter who treats the draft as final skips the counsel review
    loop that exists to catch drafting errors in regulated
    disclosures.

## 6. References

Primary legal anchors (verify URLs and versions at review time):

- **EU AI Act — Regulation (EU) 2024/1689.** Consolidated text on
  EUR-Lex: https://eur-lex.europa.eu/eli/reg/2024/1689/oj —
  Article 53(1)(d) is the public summary obligation; Annex XI is
  the regulator-facing technical documentation annex.
- **EU TDM / Copyright Directive — Directive (EU) 2019/790.**
  EUR-Lex: https://eur-lex.europa.eu/eli/dir/2019/790/oj — Article
  4 governs text-and-data mining with rightsholder reservation.
- **AI Office Commission template for the public summary of
  training data** (2025). <!-- research-note: cite by exact title and URL on the AI Office site before publication. -->

Governance frameworks:

- **NIST AI Risk Management Framework 1.0** —
  https://www.nist.gov/itl/ai-risk-management-framework (MAP and
  GOVERN functions are the relevant anchors for a training-data
  summary).
- **OWASP Machine Learning Security Top 10 (2023)** —
  https://owasp.org/www-project-machine-learning-security-top-10/
  (ML02 Data Poisoning, ML06 AI Supply Chain).
- **MITRE ATLAS** — https://atlas.mitre.org/ (tactics and
  techniques that use public training-data disclosures as inputs).

Secondary material (practitioner-level):

- **VeriSwarm** — used in the curriculum as a practitioner example
  of a data-governance tooling surface; not a legal reference.
  <!-- research-note: cite to the VeriSwarm project page if used in the draft artifact. -->

Local anchors (within this curriculum repository):

- **mod-104** — `data-engineering/mod-104-data-quality-and-schema-contracts/`
  (schema-conformance discipline applied to training-data
  documents).
- **mod-106** — `data-engineering/mod-106-eval-set-management-and-governance/`
  (contamination registry and decontamination pass).
- **mod-109** — `data-engineering/mod-109-dataset-versioning-and-reproducibility-on-a-budget/`
  (per-document manifest and top-level mix pin).
- **mod-110** — `data-engineering/mod-110-pii-and-privacy-risk-for-training-data/`
  (PII pipeline, SLOs, residual-risk statements).
- **mod-111** — `data-engineering/mod-111-data-quality-monitoring-and-regression/`
  (monitoring contracts behind the training-data quality claims).

**Reminder.** This solution is a curriculum artifact; the drafted
`training-data-summary.md` is illustrative, not legal advice. A
real provider's Article 53(1)(d) summary should be reviewed by
qualified counsel before publication.
