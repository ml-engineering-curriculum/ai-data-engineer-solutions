# Exercise 02 — ISO/IEC 42001 Clause 8 + Annex A.7 and ISO/IEC 5259 Walk (Solution)

## 1. Solution overview

This is a **design** exercise. ISO/IEC 42001 is the management-system standard
for AI (an AIMS, analogous to an ISMS under 27001). ISO/IEC 5259 is the
data-quality standard family *specifically* for analytics and machine learning.
The learner's job is to walk Clause 8 ("Operation") and the data-related
controls in Annex A, then show how ISO/IEC 5259 artifacts satisfy them at the
pipeline layer.

The deliverable is a crosswalk document with three parts:

1. A **Clause 8 ↔ pipeline-stage map** that pins each operational clause to
   the stage(s) of the training-data pipeline where the AI data engineer
   discharges it.
2. An **Annex A control ↔ artifact table** for the data-governance controls
   (Annex A.7 family), each row naming the artifact that satisfies it.
3. A **5259 overlay** that names the specific quality characteristic and
   measurement artifact the pipeline emits to close the control.

Because ISO standards are purchased documents, the solution does not reprint
their text. It names the clause and control numbers, summarises their intent
in the learner's own words, and points to artifacts in this repository that
satisfy them.

> **Standard version pinning.** ISO/IEC 42001 is the 2023 publication (first
> edition). ISO/IEC 5259 is a multi-part standard; parts 1 (concepts and
> terminology), 2 (measures), 3 (governance), and 4 (process) are the parts
> this exercise cites. Fix the exact publication year on your crosswalk and
> re-check the ISO catalogue entry for the 5259 series at review time to pick
> up any additional parts published after this solution was authored.

## 2. Implementation — worked answer

### 2.1 Clause 8 ↔ pipeline-stage map

ISO/IEC 42001 Clause 8 (Operation) covers operational planning and control of
the AI management system. The sub-clauses this exercise cares about:

| Clause (ISO/IEC 42001) | Sub-clause intent (summarised)                                                        | Pipeline stage it binds to                              | Artifact satisfying it                                            |
|------------------------|---------------------------------------------------------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------|
| 8.1 Operational planning and control | Plan, implement, and control AI processes needed to meet AIMS requirements. | All stages; the pipeline SOP itself.                   | Pipeline SOP + per-release runbook.                                |
| 8.2 AI risk assessment              | Perform AI risk assessments at planned intervals and when changes occur.       | Source selection; curation; release.                    | Per-training-objective risk assessment memo.                       |
| 8.3 AI risk treatment               | Implement controls identified in the risk-treatment plan.                      | Licence/consent gate; PII scrub; decontamination.       | Risk-treatment register + per-control evidence.                    |
| 8.4 AI system impact assessment     | Assess impact on individuals, groups, and society.                             | Source selection; labelling; release.                   | AI impact assessment (AIIA); dataset card impact section.          |

The learner's map should be at least this granular. It is acceptable to add
rows for stages where a clause does not bind — in fact, saying "Clause 8.4
does not bind at the acquisition stage because impact is assessed at the
training-objective layer" is stronger than leaving the cell empty.

### 2.2 Annex A.7 control ↔ artifact table

ISO/IEC 42001 Annex A.7 is the control family for **Data for AI systems**.
The named controls (summarised; cite control codes from the standard when
authoring):

| Control topic (Annex A.7 family)       | Intent (summarised)                                                           | Artifact in the data pipeline                                               | ISO/IEC 5259 overlay                                           |
|----------------------------------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Data management process                | A documented process for data acquisition through retirement.                 | Pipeline SOP; stage-level runbooks.                                          | 5259-3: data-quality governance process.                        |
| Data for AI systems — requirements     | Define data requirements for the AI system before acquisition.                | Training-objective statement (Exercise 01); data requirements memo.          | 5259-2: data-quality measures mapped to requirements.           |
| Acquisition of data                    | Acquire data in a way that respects legal, ethical, and quality obligations.  | Licence/consent policy; `noai`/`ai.txt` compliance log.                      | 5259-4: process view of acquisition.                            |
| Data quality for AI systems            | Assess and treat data quality against documented criteria.                    | Data-quality report per build (completeness, accuracy, consistency).         | 5259-1 + 5259-2: quality characteristics and measures.           |
| Data provenance                        | Record where data came from, when, and under what terms.                      | Lineage manifest (`manifest.json`) + per-source provenance record.           | 5259-3: provenance within DQ governance.                        |
| Data preparation                       | Document transformations applied between acquisition and training.             | Curation rulebook; filter manifest; dedup/decontam reports.                  | 5259-4: process view of preparation.                            |

The Annex A controls are **candidate** controls — organisations include /
exclude them based on their declared risk posture. Note explicitly in your
crosswalk whether each control is **included** or **excluded** and why; an
unchosen control is not a missing artifact, it is a declared non-applicability.

### 2.3 ISO/IEC 5259 quality characteristics (worked overlay)

ISO/IEC 5259 series defines data-quality concepts, measures, governance, and
process for ML/analytics data. The quality characteristics that most often
appear in a training-data pipeline map:

| Quality characteristic | Short definition (in the learner's words)                                 | Pipeline check that evidences it                                            |
|------------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Accuracy               | Data correctly reflects the real-world entity or event it is about.        | Labelled subset sampled and re-labelled; inter-annotator agreement report.  |
| Completeness           | All required attributes / records are present.                             | Schema-conformance check; non-null rates per column.                         |
| Consistency            | Data does not contradict itself within or across sources.                  | Cross-source consistency sweep; contradiction counts.                        |
| Currency / timeliness  | Data reflects the time period required by the training objective.          | Date-distribution histogram; stale-source exclusion log.                     |
| Credibility            | The data originates from a trustworthy source.                             | Per-source provenance record; publisher reputation note in sources memo.    |
| Relevance              | The data is appropriate to the AI task.                                    | Training-objective statement + representativeness audit.                     |
| Representativeness     | The data covers the population(s) the model will be used on.               | Coverage breakdown by segment; under-represented-segment follow-up list.    |
| Accessibility          | Data can be located and read by authorised consumers.                      | Dataset manifest with URIs; access-control bindings.                         |

A learner's crosswalk should pick the *subset* of 5259 characteristics the
training objective actually depends on. Listing all of them without picking
is as weak as listing none.

### 2.4 Worked crosswalk example (one row, end-to-end)

> **Artifact:** `datasheet.md` (dataset card) for `v1.2.0` of the
> packaging-assistant fine-tune corpus.
>
> **ISO/IEC 42001 clauses satisfied:**
> - Clause 8.1 (operational planning): the release process is documented in
>   the pipeline SOP and the dataset card is the release artifact.
> - Clause 8.4 (AI impact assessment): the dataset-card "Intended use and
>   out-of-scope use" section binds impact to the training-objective
>   statement.
>
> **Annex A.7 controls satisfied:**
> - *Data for AI systems — requirements*: the card restates the data
>   requirements from the training-objective memo.
> - *Data provenance*: the card links to the lineage manifest, which pins
>   every source URL, licence SPDX ID, and acquisition timestamp.
> - *Data quality for AI systems*: the card embeds the DQ report for this
>   release.
>
> **ISO/IEC 5259 characteristics measured:**
> - Completeness (schema-conformance 99.4%), Consistency (contradiction rate
>   0.3%), Representativeness (segment breakdown table), Currency (date
>   distribution 2019-01 through 2025-06).
>
> **Signer:** AI data engineer owns emission; training-team lead signs
> acceptance; data-governance lead counter-signs.

## 3. Validation steps

1. **Clause numbers are cited, not paraphrased away.** The crosswalk should
   name `8.1`, `8.2`, `8.3`, `8.4` explicitly; a reviewer will fail a
   crosswalk that describes "operational planning" without naming 8.1.
2. **Annex A controls are listed with their inclusion / exclusion status.**
   A control that is excluded with a documented reason is a satisfactory row;
   an unexplained missing control is a gap.
3. **5259 characteristics are chosen, not scattered.** The learner picks the
   subset the training objective depends on and defends the subset.
4. **Every artifact is a file in this repo (or named as missing).** Hand-waved
   artifacts ("we have a doc somewhere") fail the walk.
5. **Standards are version-pinned.** The crosswalk names the publication year
   of 42001 and the parts of 5259 it cites.
6. **The AIIA artifact is identified.** AI impact assessment (Clause 8.4) is
   the control a learner is most likely to miss — if the walk has no AIIA, it
   has not touched the standard's Operation clause in full.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 18/24 and loses zero points on the rows marked
**critical**.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | ISO/IEC 42001 sub-clauses 8.1, 8.2, 8.3, 8.4 are all mapped to pipeline stages                                   | Yes       |
| 2  | Annex A.7 (data for AI systems) controls are enumerated with inclusion/exclusion status                         | Yes       |
| 3  | Each control row names a concrete artifact (file path or document name)                                         | Yes       |
| 4  | ISO/IEC 5259 characteristics are chosen to match the training objective, not listed exhaustively                | Yes       |
| 5  | Dataset card is cross-referenced to Clause 8.4 (impact)                                                         | Yes       |
| 6  | AI risk assessment artifact is named and bound to Clause 8.2                                                    | Yes       |
| 7  | Risk-treatment register exists and is bound to Clause 8.3                                                       | Yes       |
| 8  | Standards are version-pinned (42001 edition year; 5259 parts cited)                                             | Yes       |
| 9  | Excluded Annex A controls have a one-sentence justification                                                     | No        |
| 10 | Crosswalk links to the NIST AI RMF map from Exercise 01 so overlaps are visible                                 | No        |
| 11 | Every row names the single accountable signer                                                                   | No        |
| 12 | Open follow-up notes (`<!-- follow-up: ... -->`) are used where a clause intent is unclear to the author        | No        |

## 5. Common mistakes

- **Treating 42001 as a technical standard.** It is a management-system
  standard; its Clause 8 artifacts are *process* artifacts (SOPs, registers,
  sign-offs), not model code. Learners who try to satisfy 8.1 with a Python
  script have misread the standard.
- **Confusing Annex A controls with mandatory controls.** Annex A is the
  candidate control set. Inclusion decisions are owner decisions; a crosswalk
  that pretends every control is mandatory shows the learner has not read the
  standard's structure.
- **Treating 5259 as a replacement for 42001.** 5259 is the data-quality
  standard family; it slots *inside* 42001's data-control clauses. They are
  complementary, not alternatives.
- **Listing all 5259 quality characteristics without a training objective.**
  Representativeness means nothing without a population; relevance means
  nothing without an intended use.
- **Omitting the AI impact assessment (Clause 8.4).** This is the control
  most often collapsed into "we have a risk register" — those are different
  artifacts.
- **Not pinning the standard edition.** ISO standards are revised; citing
  "ISO/IEC 42001" without the year is the ISO equivalent of citing a web
  page without an access date.
- **Paraphrasing Annex A control names so loosely that the standard's intent
  is lost.** Keep the short name; summarise the intent beside it.

## 6. References

- ISO/IEC 42001:2023 — *Information technology — Artificial intelligence —
  Management system* (ISO publication; purchased via ISO or a national
  standards body). Clause 8 (Operation) and Annex A (Reference controls) are
  the anchors for this exercise.
- ISO/IEC 5259 series — *Artificial intelligence — Data quality for analytics
  and machine learning (ML)* (multi-part; purchased via ISO).
  - Part 1: Overview, terminology, and examples
  - Part 2: Data-quality measures
  - Part 3: Data-quality management and governance
  - Part 4: Data-quality process framework
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (For cross-walking 42001 clauses to the NIST function the same artifact
  also satisfies — see Exercise 01.)
- NIST AI 600-1 — *Generative AI Profile* —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
  (For aligning the risk-register content under Clause 8.2 with contemporary
  genAI risks.)
- Local anchors: Exercise 01 (NIST AI RMF artifact map), Exercise 03 (EU AI
  Act Article 10 deliverables), Exercise 04 (lawful-basis admit function).
