# Exercise 03 — EU AI Act Article 10 and Annex XI Deliverables (Solution)

## 1. Solution overview

This is a **design** exercise. The learner reads EU Regulation 2024/1689 (the
AI Act), identifies the data-governance obligations that bind at training
time, and produces the concrete deliverables an AI data engineer must hand to
the compliance function.

The two anchors:

- **Article 10 — Data and data governance.** Imposes data-quality,
  representativeness, bias-examination, and documentation obligations on
  high-risk AI systems. Training, validation, and testing datasets must meet
  the quality criteria enumerated there.
- **Annex XI — Technical documentation for providers of general-purpose AI
  (GPAI) models.** Sets the documentation obligations that bind GPAI model
  providers at the data layer (training-data description, data-governance
  measures, copyright policy, and — for GPAI models placed on the EU market —
  the summary of content used for training).
- **Annex IV — Technical documentation for high-risk AI systems.** Sets the
  broader technical-documentation obligations into which the data section
  slots when the system is high-risk.

A learner's deliverable is a package of three documents:

1. An **Article 10 compliance memo** that walks each paragraph of Article 10
   and names, for each obligation, the pipeline artifact that satisfies it.
2. A **Technical-documentation data section** sized for either Annex IV
   (high-risk system) or Annex XI (GPAI model), matching the system's
   regulatory classification.
3. A **public training-data summary** (if the system is a GPAI model placed
   on the EU market) — the "sufficiently detailed summary" required by the
   Act for GPAI providers.

Not every system is high-risk and not every model is a GPAI model; the memo
must state the **classification choice** up front and defend it. A deliverable
that cannot name the Act category it is targeting cannot be assessed.

> **Regulation-version pinning.** EU Regulation 2024/1689 ("the AI Act") was
> adopted in 2024 with staged entry into application. Obligations on GPAI
> providers and on high-risk systems have distinct application dates. Pin the
> dates on your memo at authoring time and re-check against the Official
> Journal before submission; confirm which GPAI and high-risk obligations
> are in application at the date of the learner's submission against the
> staged application dates set by Article 113.

## 2. Implementation — worked answer

### 2.1 Classification statement (worked example, high-risk case)

> **System.** Internal CV-screening assistant for the HR team, 500-employee
> company, EU operations.
> **Classification.** High-risk under Annex III (employment, workers
> management, and access to self-employment) — specifically the recruitment
> category.
> **Consequence.** Article 10 applies. The full technical-documentation
> package under Annex IV applies, with the data section informed by
> Article 10's paragraphs. Annex XI does not apply because the system is not
> a GPAI model placed on the market; however, if a GPAI base model is used,
> Annex XI obligations transfer to the base-model provider and are inherited
> as upstream documentation.

An alternative classification statement for the GPAI case:

> **System.** General-purpose LLM at the 10B-parameter scale, released as
> downloadable weights to EU users.
> **Classification.** GPAI model. Annex XI applies (including the public
> training-data summary). Whether the model additionally meets "GPAI with
> systemic risk" criteria must be assessed against the Act's thresholds; if
> so, the additional obligations of that tier apply.

### 2.2 Article 10 paragraph ↔ artifact table

Article 10's substantive paragraphs (summarised in the learner's words; cite
the paragraph numbers — the article's structure as adopted is paragraph-based
and must be read against the Official Journal text):

| Paragraph (Art. 10)                                                                                       | Obligation (summarised)                                                                                       | Artifact in the data pipeline                                                                                 |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Data-governance practices apply to training, validation, and testing datasets.                            | A single data-governance process covers all three splits.                                                     | Pipeline SOP covers all three splits; dataset card lists train/val/test provenance separately.                |
| Design choices and collection processes must be documented.                                               | Why these sources, how they were collected, what was excluded.                                                | Source-selection matrix (mod-101 Ex-01 / mod-102 Ex-01); acquisition log.                                    |
| Data preparation (annotation, labelling, cleaning, updating, enrichment, aggregation) must be documented. | Each transformation between acquisition and training is recorded.                                             | Curation rulebook; labelling guidelines; dedup / decontam reports; versioned transform manifest.              |
| Formulation of assumptions, notably about information the data is supposed to measure and represent.      | The data's intended construct and the assumptions that justify using it.                                      | Training-objective statement; representativeness audit.                                                       |
| Assessment of availability, quantity, and suitability of datasets.                                        | Explicit coverage assessment vs. the deployment population.                                                   | Representativeness and coverage report.                                                                       |
| Examination in view of possible biases that could affect health, safety, fundamental rights or discrimination. | Bias examination with documented methodology and findings.                                                   | Bias-examination report; per-protected-attribute disaggregation where legally permissible.                    |
| Identification of relevant data gaps or shortcomings and how to address them.                             | Gaps are named, not hidden; remediation is tracked.                                                           | Gap list (from mod-101 Ex-01) with remediation owner and date.                                                |
| Relevant, representative, free of errors (to the best extent possible), and complete *for the intended purpose*. | Quality measured against the use, not in the abstract.                                                   | DQ report keyed to the training objective; see ISO/IEC 5259 overlay in Ex-02.                                 |
| Account for geographical, contextual, behavioural, or functional setting.                                 | Setting-specific coverage decisions are documented.                                                           | Representativeness report breaks coverage by relevant setting (e.g. EU-member-state distribution if relevant).|
| Processing of special categories of personal data for bias detection, with strict safeguards (if used).   | If special-category PII is used for bias detection, name the Article 9 derogation applied and the safeguards. | Special-category-data handling memo; privacy sign-off.                                                        |

The learner's memo should restate each obligation briefly and point to the
artifact. A row that says "we handle this" without a filename fails the walk.

### 2.3 Annex IV data section (worked example skeleton)

Annex IV requires technical documentation covering, among other things, the
data used to develop the system. The data section a learner should produce:

```markdown
# Technical documentation — Data section (Annex IV mapping)

## D.1  Datasets and their characteristics
- Training dataset: version, size, source list, provenance manifest reference.
- Validation and test datasets: same, plus decontamination report vs. training.
- Geographical / contextual / functional coverage statements.

## D.2  Data-collection and preparation procedures
- Acquisition methods (crawl, licence, vendor, synthetic — per source).
- Labelling procedure; annotator guidelines; inter-annotator agreement.
- Cleaning, filtering, deduplication, and decontamination steps.
- Versioned transform manifest reference.

## D.3  Data-governance measures
- Representativeness assessment and findings.
- Bias examination methodology and findings.
- Gaps identified and remediation plan.
- Personal-data handling, including any special-category data and safeguards.

## D.4  Assumptions about what the data represents
- Training-objective statement.
- Construct validity discussion: what the data is a proxy for.

## D.5  Accountability
- Named owner of the data section.
- Sign-off chain and dates.
```

### 2.4 Annex XI training-data summary (worked example skeleton, GPAI case)

For GPAI models placed on the EU market, the Act requires a *sufficiently
detailed* summary of the content used for training, made publicly available.
The summary should be a public-facing document, not a legal brief.

```markdown
# Public training-data summary — <Model name / version>

## 1. Overview
- Model identifier and version.
- Approximate total training-data volume and token count.
- Date range covered by training data.

## 2. Major data categories
- Web crawl (named crawls and date windows).
- Code (named repositories / aggregators).
- Books and academic text (licensing posture).
- Other substantial sources (synthetic data, partner corpora, etc.).
- For each: approximate share, licensing posture, and opt-out-signal handling.

## 3. Copyright policy
- How the model provider respects reservations of rights under EU copyright
  law (including the TDM opt-out under Directive (EU) 2019/790).
- How `robots.txt`, `ai.txt`, `noai`, and comparable signals are honoured.
- How copyright complaints and takedown requests are received and handled.

## 4. Data-governance measures
- Quality, bias, and safety filtering applied.
- Personal-data posture (PII scrubbing, special-category exclusions).

## 5. Known limitations
- Languages / regions under-represented.
- Domains explicitly excluded.
- Known failure modes attributable to data.

## 6. Contact
- Who to contact for copyright, data-subject, or correction issues.
```

### 2.5 Copyright compliance statement (Annex XI, GPAI)

GPAI providers must adopt a copyright policy. The minimum content:

- A statement that EU copyright law is respected, with specific reference to
  the text-and-data-mining exception and the rightholders' ability to
  reserve use.
- A description of how reservations are detected (robots.txt, ai.txt, `noai`
  metadata, X-Robots-Tag directives, publisher-API opt-outs, and any
  bilateral agreements).
- A description of exclusions applied at acquisition time, with evidence
  (acquisition logs).

## 3. Validation steps

1. **Classification is stated first.** Which category under the AI Act applies
   — high-risk (Annex III), GPAI, both, neither — is named before any
   deliverable is produced.
2. **Every Article 10 paragraph has a mapped artifact.** Paragraphs the
   learner believes are inapplicable are marked "N/A — <reason>", not simply
   omitted.
3. **Annex IV data section covers D.1–D.5** or an equivalent partition. A
   section missing D.4 (assumptions / construct validity) is a common failure.
4. **Annex XI public summary is actually public.** The content must be
   publishable; if it reads like an internal brief, it is not the artifact
   the Act requires.
5. **Copyright policy names the TDM opt-out machinery.** `robots.txt`,
   `ai.txt`, and `noai` are the operational hooks; the policy must name
   which signals are honoured and how.
6. **Special-category personal-data use is called out.** If no special-
   category data is used, say so; if it is used for bias detection, name the
   derogation and the safeguards.
7. **Bias examination has a methodology.** "We looked at bias" without a
   documented method fails the Article 10 requirement that bias examination
   be a described process.
8. **Dataset provenance is version-pinned.** Named crawls and date windows,
   not "various web sources".

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 20/28 and loses zero points on the rows marked
**critical**.

| #  | Dimension                                                                                                        | Critical? |
|----|------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Classification under the AI Act (high-risk / GPAI / neither) is stated and defended                              | Yes       |
| 2  | Every Article 10 paragraph has a mapped artifact or an explicit N/A with reason                                  | Yes       |
| 3  | Train / validation / test splits are each documented, not conflated                                              | Yes       |
| 4  | Annex IV data section covers datasets, collection, governance, assumptions, accountability                       | Yes       |
| 5  | Annex XI public training-data summary exists if (and only if) the system is a GPAI model on the EU market        | Yes       |
| 6  | Copyright policy names the TDM reservation mechanism and the operational signals honoured                        | Yes       |
| 7  | Bias-examination methodology is documented, not just asserted                                                    | Yes       |
| 8  | Representativeness is tied to the deployment population, not evaluated in the abstract                           | Yes       |
| 9  | Gap list from Exercise 01 is referenced and gaps have remediation owners                                         | No        |
| 10 | Special-category personal-data use is addressed (even if the answer is "none used")                              | Yes       |
| 11 | Regulation is version-pinned (Regulation (EU) 2024/1689 and the application-date tier that binds)                | Yes       |
| 12 | Open follow-up notes are used anywhere a dataset share or crawl window is approximate                            | No        |
| 13 | Cross-reference to the ISO/IEC 42001 walk (Ex-02) so overlaps are visible                                        | No        |
| 14 | Public summary is written at a reading level a non-specialist can act on                                         | No        |

## 5. Common mistakes

- **Treating every AI system as high-risk.** High-risk is a narrow list
  (Annex III and products covered by sectoral law listed in Annex I).
  Mis-classifying an internal productivity tool as high-risk inflates
  obligations and signals the learner has not read the Act's structure.
- **Treating no fine-tune as "not a provider".** Providers of GPAI models
  placed on the EU market are in scope; if your organisation puts a
  fine-tuned GPAI model on the EU market, the obligations may transfer.
- **Confusing Annex IV with Annex XI.** Annex IV binds providers of
  high-risk AI systems. Annex XI binds GPAI model providers. They overlap
  but are not the same document.
- **Omitting the public summary (Annex XI).** The GPAI public summary is a
  specific artifact — the "sufficiently detailed summary" required by the
  Act — and its publicness is the point.
- **Collapsing bias examination into a single bullet.** The Act expects a
  documented examination, not a claim.
- **Ignoring the TDM opt-out machinery.** The copyright policy must name
  how machine-readable reservations are detected and honoured.
- **Using special-category data for bias detection without documenting the
  legal basis.** The Act permits this *under safeguards*; the safeguards
  must be named.
- **Writing "data is representative" without saying of what.** The
  representativeness claim requires a named population.

## 6. References

- Regulation (EU) 2024/1689 (the AI Act) — primary text, published in the
  Official Journal of the European Union. Article 10 (Data and data
  governance), Article 11 and Annex IV (Technical documentation for
  high-risk systems), Article 53 and Annex XI (Obligations for providers of
  GPAI models), and Article 9 of GDPR as referenced for special-category
  data.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (For cross-walking Article 10 obligations to NIST MAP / MEASURE /
  MANAGE anchors.)
- NIST AI 600-1 — *Generative AI Profile* —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
  (For the GenAI-specific risks that the Annex XI summary must address.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (Secondary lens on the data-governance measures Annex XI describes.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Adversarial-technique lens on acquisition-time controls.)
- Directive (EU) 2019/790 (DSM Copyright Directive) — Articles 3 and 4 on
  text-and-data-mining and rightholder reservation; the operational basis
  for the TDM-opt-out machinery named in the copyright-policy section.
- Local anchors: Exercise 01 (NIST AI RMF map), Exercise 02 (ISO walk),
  Exercise 04 (lawful-basis admit function).
