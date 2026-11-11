# Exercise 04 — Datasheets for Datasets and Data Statement Authoring (Solution)

## 1. Solution overview

This is a **design** exercise. Two documentation artifacts are authored for the
same corpus, each one speaking to a different audience:

- A **Datasheet for Datasets** (Gebru et al., 2021 framework) answers the
  seven-section questionnaire that lets a downstream team decide whether the
  dataset is appropriate for their use.
- A **Data Statement** (Bender & Friedman, 2018, revised in a 2021 schema V2
  update) answers the questions a *modeller* needs to reason about system
  behaviour on populations the dataset does (or does not) represent — curator
  rationale, language varieties, speaker demographics, annotator demographics,
  recording quality, and so on.

Both documents are authored once at corpus freeze, pinned to the dataset
version, and re-authored (not patched) when the dataset materially changes.
"Materially" is defined in §2.5.

The deliverable is the two filled-in documents plus a short **joint narrative**
tying them to the Exercise 01 training objective and the Exercise 02 licence
policy. The joint narrative is the piece reviewers most often skip and the
piece that most often catches "this dataset isn't appropriate for that model".

## 2. Implementation — worked answer

### 2.1 Datasheet for Datasets — section skeleton with worked content

Section titles below reflect the Gebru et al. framework.

#### Motivation

- **For what purpose was the dataset created?** Supervised fine-tuning data for
  an internal Python-packaging assistant (see mod-102 Exercise 01). Not a
  benchmark; not a general-purpose corpus; not intended for use outside the
  objective.
- **Who created the dataset?** Named team, reporting line, funding source. The
  role is named; the person is not, so the artifact survives staff turnover.
- **Who funded the creation?** Named cost centre (internal engineering).

#### Composition

- **What do the instances represent?** Question-and-answer pairs on Python
  packaging topics.
- **How many instances are there?** `N` total, split `N_train / N_dev / N_held`
  (filled at freeze time).
- **Does the dataset contain all possible instances, or a sample?** Sample. The
  sampling frame is: internal support tickets 2024-01-01 to 2025-12-31;
  packaging.python.org snapshot at commit `<sha>`; Stack Overflow Data Explorer
  export `<yyyy-mm-dd>` restricted to the four packaging tags listed in
  Exercise 01.
- **What data does each instance consist of?** Prompt (string), answer
  (string), per-row provenance (source URL, SPDX licence identifier, retrieval
  timestamp), quality flags.
- **Is there a label or target associated with each instance?** The answer
  string is the target for the SFT objective.
- **Is any information missing from individual instances?** Author identity is
  deliberately omitted after PII scrubbing; the omission is intentional, not an
  accident. (Document which fields were stripped and why.)
- **Are relationships between individual instances made explicit?** Yes —
  per-row provenance preserves source URL grouping so the licence and opt-out
  decisions from Exercises 02–03 can be re-run per-row.
- **Are there recommended data splits?** The `held` split is a chronologically
  later window (2026-01 onward) to make temporal drift observable (see
  mod-106).
- **Are there any errors, sources of noise, or redundancies?** Stack Overflow
  answers can be duplicative across similar questions; dedup is applied per
  mod-105 (recorded in lineage, not re-documented here).
- **Is the dataset self-contained, or does it link to external resources?**
  Self-contained for training; per-row provenance references external URLs
  that are *not* re-fetched at training time.
- **Does the dataset contain data that might be considered confidential?**
  Tickets are filtered to remove internal-only content tagged confidential;
  cross-check documented in the data card.
- **Does the dataset contain data that, if viewed directly, might be offensive,
  insulting, threatening, or might otherwise cause anxiety?** Low risk by
  topic, but the corpus is still run through the content safety classifier
  before freeze; retained edge cases are listed with counts.
- **Does the dataset relate to people?** Yes (support-ticket authors and Stack
  Overflow contributors). PII handling: mod-110 pipeline applied before this
  dataset was assembled.
- **Does the dataset identify any subpopulations?** No explicit subpopulation
  labels. This is a limitation for representativeness analysis and is called
  out in the Data Statement.

#### Collection Process

- **How was the data acquired?** Internal tickets: SQL export from the support
  database. PyPA docs: `git clone` + Markdown extraction from the pinned commit.
  Stack Overflow: Data Explorer export under the SE terms of service.
- **What mechanisms or procedures were used to collect the data?** Documented
  in `pipeline/ingest/<source>.py`; manifest hashes pinned in lineage (mod-109).
- **If the dataset is a sample from a larger set, what was the sampling
  strategy?** Convenience sample constrained by date range and tag filter; not
  statistically representative of Python-packaging users at large (this is
  flagged in the Data Statement).
- **Who was involved in the data collection?** Named roles; compensation terms
  for any paid labellers recorded in mod-107 artifacts.
- **Over what timeframe was the data collected?** Collection window: snapshot
  taken at `YYYY-MM-DD HH:MM:SS Z`. Content window: as above.
- **Were any ethical review processes conducted?** Internal governance review
  per mod-101; outcome archived.
- **Did you collect the data from the individuals directly, or obtain it via
  third parties?** Third-party for SO; first-party for tickets and PyPA docs.
- **Were the individuals notified about the data collection?** Tickets:
  covered by the product's privacy notice (confirmed under Exercise 01). PyPA
  / SO: notified by virtue of public posting under their respective ToS.
- **Did the individuals consent to the collection and use of their data?**
  Ticket submitters: see the privacy-notice determination. Public contributors:
  consent is via the content licence and the honoured opt-out protocol
  (Exercise 03).
- **If consent was obtained, were consenting individuals provided with a
  mechanism to revoke their consent?** Opt-out re-sweep (Exercise 03) picks up
  revocations before the next retrain; deletion-on-opt-out process in mod-110.

#### Preprocessing / Cleaning / Labeling

- **Was any preprocessing, cleaning, or labeling of the data done?** PII scrub
  (mod-110), dedup (mod-105), quality filter (mod-104); each step produces a
  manifest captured in lineage.
- **Was the raw data saved in addition to the preprocessed data?** Yes; raw
  snapshot stored in cold storage with a 30-day retention beyond the final
  training run, per the retention schedule.
- **Is the software used to preprocess / clean / label the data available?**
  Yes — in-tree under `pipeline/`.

#### Uses

- **Has the dataset been used for any tasks already?** Internal SFT only; no
  external releases.
- **Is there a repository that links to any or all papers or systems that use
  the dataset?** `MODEL_CARD.md` on the deployed model.
- **What other tasks could the dataset be used for?** Potentially: a retrieval
  index over packaging docs. Not: open-domain English QA.
- **Is there anything about the composition of the dataset or how it was
  collected that might impact future uses?** Yes — convenience sample; English
  only; packaging topics only; see the Data Statement for population-level
  consequences.
- **Are there tasks for which the dataset should not be used?** Yes — do not
  use for general-purpose QA, code generation beyond the stated scope, or any
  customer-facing deployment without renewed governance review.

#### Distribution

- **Will the dataset be distributed to third parties?** No. Internal use only.
- **How will the dataset be distributed?** It will not be.
- **When will the dataset be distributed?** Not applicable.
- **Will the dataset be distributed under a copyright or other IP license?**
  Internal-use only; downstream artifacts (model weights) are subject to the
  propagation flags from the Exercise 02 policy (ShareAlike / NC flags where
  applicable).
- **Will the dataset be subject to any export controls or regulatory
  restrictions?** Internal review flagged none at this risk tier; re-check on
  re-authoring.

#### Maintenance

- **Who is supporting / hosting / maintaining the dataset?** Named owning team
  (role, not person).
- **How can the owner be contacted?** Internal distribution list.
- **Is there an erratum?** `ERRATA.md` adjacent to the dataset artifact.
- **Will the dataset be updated?** Not patched. Re-authored as `v1.1`, `v2.0`,
  etc., on the triggers in §2.5.
- **If the dataset relates to people, are there applicable limits on the
  retention of the data?** Yes — per mod-110 retention schedule.
- **Will older versions of the dataset continue to be supported / hosted /
  maintained?** Yes until the governance-mandated minimum retention elapses.
- **If others want to extend / augment / build on / contribute, is there a
  mechanism for them to do so?** Yes — open a change request against the
  ingest manifest; new sources go through Exercise 01 → Exercise 02 →
  Exercise 03 before admission.

### 2.2 Data Statement — section skeleton with worked content

Section titles below reflect the Bender & Friedman schema (V2).

- **A. Curation rationale.** Who selected which texts, from where, and why.
  Document the same reasoning the Exercise 01 matrix produced, framed
  statistically: the corpus is *not* a random sample of Python-packaging
  discourse; it is constrained to the four tag families plus the PyPA canonical
  docs. State this plainly.
- **B. Language variety / varieties.** English, primarily. Indicate the
  dialectal provenance insofar as it can be inferred: US / UK English
  dominant on SO and PyPA; internal tickets include non-native English.
- **C. Speaker demographic.** Not individually recorded post-PII-scrub.
  Aggregate inferences (internal engineers; SO contributors) are stated at the
  population level only, with a clear "not measured" where applicable.
- **D. Annotator demographic.** For any annotation layer added on top of the
  raw text (e.g. quality scoring): named roles, compensation model, training
  provided (details in mod-107).
- **E. Speech situation and text characteristics.** Asynchronous written text;
  Q&A genre; technical register; mean post length and standard deviation
  reported from the corpus.
- **F. Preprocessing and labeling.** Point to §Preprocessing in the Datasheet;
  do not duplicate the manifest here.
- **G. Recording quality.** Not applicable (no audio).
- **H. Other.** Known limitations (e.g. packaging ecosystem churn — some
  answers cite deprecated APIs; held-out split flags temporal drift).
- **I. Provenance appendix.** Per-row provenance lives in the dataset artifact,
  not in the data statement; the data statement points to it.

### 2.3 Joint narrative (the piece reviewers skip)

A short section that explicitly ties the two documents together:

> The datasheet documents *what the dataset is*; the data statement documents
> *who it represents*. The Exercise 01 training objective constrains how both
> documents should be read: the dataset is fit for a narrow, internal
> packaging-Q&A fine-tune, and not for any broader claim. If the deployment
> context expands (customer-facing, non-English, non-packaging), both documents
> must be re-authored *before* the expansion, not after.

### 2.4 Pinning and co-location

Both documents live beside the dataset artifact and are committed in the same
change as the artifact's manifest. File names:

- `DATASHEET.md` (Datasheet for Datasets)
- `DATA_STATEMENT.md` (Data Statement for NLP)
- `ERRATA.md` (post-freeze corrections only)

Each document's front matter pins the dataset version it describes. Changing
the dataset version without re-authoring these files is a rubric failure
(§4 row 10).

### 2.5 "Material change" triggers (when to re-author, not patch)

Re-author both documents (and bump the dataset version) when **any** of the
following changes:

- New source added, or existing source's licence / opt-out posture changes.
- Population coverage changes (new language, new country, new content
  category).
- Collection window extended or truncated.
- PII handling policy changed (mod-110).
- Deployment context changes (internal → customer-facing, non-commercial →
  commercial, risk tier change).

Non-material changes (typos, clarifications) go to `ERRATA.md`; the dataset
version stays pinned.

## 3. Validation steps

1. **Both documents exist and each is pinned to the same dataset version.** A
   datasheet without a matching data statement (or vice versa) fails this
   exercise.
2. **Every Gebru-et-al section prompt is addressed.** "Not applicable" is an
   acceptable answer **only** when preceded by a one-sentence reason; blank
   fields are rubric failures.
3. **Every Bender-&-Friedman section is addressed.** Same rule: "not measured"
   is acceptable with a reason; blank is not.
4. **Joint narrative is present.** It explicitly names the training objective
   and the deployment context from Exercise 01.
5. **The two documents do not contradict each other.** E.g. a datasheet that
   says "English only" and a data statement that does not mention language
   variety is an inconsistency that must be fixed.
6. **No individual-level demographic claims are made from aggregate signals.**
   The data statement calls out "not measured" rather than inferring.
7. **The datasheet's Distribution section is consistent with the Exercise 02
   licence policy flags.** E.g. if any `CC-BY-SA` material entered the corpus,
   the datasheet acknowledges the ShareAlike flag.
8. **Sanity-check cross-references.** The datasheet should reference mod-101
   (governance), mod-104 (quality), mod-105 (dedup), mod-107 (annotation),
   mod-109 (lineage), mod-110 (PII) — where each applies. Missing references
   are a tell that the document was authored in isolation.
9. **"Who to contact" is a role, not a person.** Personal names in the
   maintenance section cause the document to go stale on staff change.
10. **Re-authoring triggers are listed.** Documents without §2.5-style
    triggers tend to decay into "we updated the corpus but forgot to update
    the datasheet" within a quarter.

## 4. Rubric or review checklist

Score each row 0–2. Passing solution ≥ 22/28 and zero critical misses.

| # | Dimension                                                                                           | Critical? |
|---|-----------------------------------------------------------------------------------------------------|-----------|
| 1 | Datasheet present, covers all seven Gebru-et-al sections                                            | Yes       |
| 2 | Data statement present, covers all Bender-&-Friedman V2 sections                                    | Yes       |
| 3 | Both pinned to the same dataset version in front matter                                              | Yes       |
| 4 | Training objective and deployment context from Exercise 01 are explicitly cited                      | Yes       |
| 5 | Each "not applicable" / "not measured" field has a one-sentence reason                              | Yes       |
| 6 | PII handling, opt-out, and licence references point to mod-110, Exercise 03, Exercise 02            | Yes       |
| 7 | Downstream-use guidance states what the dataset is *not* for, not only what it is for               | Yes       |
| 8 | Owning *role* (not person) named in Maintenance                                                     | Yes       |
| 9 | Joint narrative tying the two documents together is present                                          | No        |
| 10 | Material-change triggers are listed; `ERRATA.md` handles non-material fixes                         | Yes       |
| 11 | No internal-only or confidential content leaks into the datasheet                                    | Yes       |
| 12 | Open follow-up notes used wherever a template version was not freshly verified                      | No        |
| 13 | Distribution section is consistent with the Exercise 02 licence flags (SA / NC propagation)          | No        |
| 14 | Documents do not contradict each other (language variety, population, time window)                   | Yes       |

## 5. Common mistakes

- **Treating "Not applicable" as the universal answer.** Rubric row 5 catches
  this; every N/A needs a reason.
- **Naming a person as the dataset owner.** Owner is a role; otherwise the
  document rots on staff change.
- **Omitting "what the dataset is *not* for".** Downstream teams default to
  "general-purpose" unless the datasheet refuses that framing.
- **Patching the datasheet in place when a new source is added.** A new
  source is a material change; bump the version and re-author.
- **Writing the data statement as a mirror of the datasheet.** They have
  different audiences (dataset consumer vs. modeller reasoning about
  population). Overlap is fine; identity is not.
- **Inferring demographics from signal.** "Most SO contributors are …" is a
  claim that requires a measurement. Without one, write "not measured".
- **Letting the datasheet drift from the licence policy.** If the policy
  admits `CC-BY-SA` with a propagation flag and the datasheet doesn't mention
  the flag, the downstream reader cannot do their job.
- **Not citing the Exercise 01 training objective.** The datasheet without a
  linked objective reads as a general-purpose corpus and is reused as one.

## 6. References

- Gebru, T. et al. "Datasheets for Datasets." (Original 2018 proposal;
  revised 2021.) (Verify the canonical publication URL before citing.)
- Bender, E. M., & Friedman, B. "Data Statements for NLP." (Original 2018
  TACL paper; V2 schema update 2021.) (Verify the canonical publication URL
  before citing.)
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (GOVERN 1.4 / GOVERN 6 — documentation; MAP 2 / MAP 3 — scope and
  categorisation; MEASURE 2.11 — documentation of data and its context.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (Documentation of data lineage and contents is a defence-in-depth control
  against several risks in the Top 10; cite the specific risk when the
  datasheet is used in a threat-model review.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (The datasheet is one of the artifacts an ATLAS-based data-supply-chain
  threat model uses; browse the current matrix for the appropriate technique
  references at review time.)
- Local anchors: Exercise 01 (training-objective scoping), Exercise 02
  (licence propagation flags that the datasheet must reflect), Exercise 03
  (opt-out reality the datasheet must not overclaim away), mod-107
  (annotation demographics), mod-109 (lineage pointer), mod-110 (PII
  handling).
