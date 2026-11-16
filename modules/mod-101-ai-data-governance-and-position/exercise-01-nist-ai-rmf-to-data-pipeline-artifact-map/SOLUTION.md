# Exercise 01 — NIST AI RMF → Data Pipeline Artifact Map (Solution)

## 1. Solution overview

This is a **design** exercise. The learner takes the four functions of the NIST
AI Risk Management Framework — **GOVERN, MAP, MEASURE, MANAGE** — and produces a
single deliverable: a table that shows, for every pipeline stage an AI data
engineer actually owns, which NIST function is in force, what tangible
*artifact* satisfies it, who owns the artifact, and when it is refreshed. The
Generative AI Profile (NIST AI 600-1) adds genAI-specific risks that must be
reflected in the same table.

A good solution produces three things:

1. A **pipeline-stage inventory** that is granular enough to pin responsibility
   to (acquire → license-check → curate → dedupe → label → pre-process → version
   → release). Stages a learner cannot name an artifact for are stages they do
   not yet understand.
2. An **artifact map** — the core table — that for each stage lists the
   governance artifact, the NIST AI RMF function and sub-category it satisfies,
   the GenAI-Profile risk it addresses (where applicable), the artifact owner,
   and the refresh cadence.
3. A **gap list** that names artifacts the team does *not* yet have. The point
   of mapping is to surface gaps; a map with no gaps is a map that was not
   stress-tested.

The output of this exercise is the planning document the rest of the module's
exercises build against.

## 2. Implementation — worked answer

### 2.1 Pipeline-stage inventory (worked scope)

The AI data engineer's beat is the training-data pipeline up to and including
dataset release. Model training, evaluation, and deployment are explicitly out
of scope — see Exercise 05 for the deferral contract that formalises this.

| # | Stage                 | One-line description                                                       |
|---|-----------------------|----------------------------------------------------------------------------|
| 1 | Source selection      | Decide which corpora are candidates for the training objective.            |
| 2 | Licence / consent gate | Admit sources only if licence terms and consent/opt-out posture permit.    |
| 3 | Acquisition           | Fetch raw bytes while respecting robots/noai directives and rate limits.   |
| 4 | Curation              | Clean, filter, and bound the content to the training objective.            |
| 5 | Deduplication         | Collapse near-duplicates across and within sources.                        |
| 6 | Decontamination       | Remove overlap with evaluation benchmarks before release.                  |
| 7 | Labelling / annotation | Produce labels or preference pairs via vendors or internal annotators.     |
| 8 | PII / privacy scrub   | Detect and redact or exclude personal data; honour erasure requests.       |
| 9 | Versioning + lineage  | Freeze a content-addressed dataset version with full provenance.           |
| 10 | Release / handoff     | Publish the dataset card to the training team with sign-off attached.      |

### 2.2 Artifact map (worked example)

The table below is the deliverable. "Sub-cat." refers to the NIST AI RMF
sub-category code (e.g. `MAP 4.1`). Codes must be read off the current NIST AI
RMF Core document at authoring time, because they are periodically re-numbered;
the codes used here reflect the Framework 1.0 Core as published and are the
anchor points the exercise expects a learner to cite.

| Stage                 | Artifact (name, filetype)                                | NIST AI RMF function | Sub-cat. anchor(s) | GenAI-Profile risk anchor                        | Owner                        | Refresh cadence             |
|-----------------------|----------------------------------------------------------|----------------------|--------------------|--------------------------------------------------|------------------------------|-----------------------------|
| 1. Source selection   | Source-selection matrix + memo (`sources.md`)            | MAP                  | MAP 1.1, MAP 4.1   | Data privacy; CBRN info (if in-scope)             | AI data engineer             | Per training objective      |
| 2. Licence / consent  | Licence policy + per-source licence record (`licences.yaml`) | GOVERN, MAP      | GOVERN 1.1, MAP 4.1 | IP infringement                                   | Legal + AI data engineer     | Quarterly, and per source   |
| 3. Acquisition        | Robots / `noai` / `ai.txt` compliance log (`acq_log.jsonl`) | GOVERN, MAP        | GOVERN 5.1, MAP 4.1 | Information integrity                             | AI data engineer             | Per crawl                   |
| 4. Curation           | Curation rulebook + filter manifest (`curation.yaml`)    | MAP, MEASURE         | MAP 2.3, MEASURE 2.3 | Harmful bias / content safety                   | AI data engineer             | Per pipeline version        |
| 5. Deduplication      | Dedup report with hash algorithm, thresholds, counts (`dedup.md`) | MEASURE     | MEASURE 2.1        | Confabulation; data privacy                       | AI data engineer             | Per dataset build           |
| 6. Decontamination    | Decontamination report vs. named benchmarks (`decontam.md`) | MEASURE          | MEASURE 2.1, MEASURE 2.3 | Confabulation; evaluation integrity          | AI data engineer             | Per dataset build           |
| 7. Labelling          | Annotator guidelines + inter-annotator agreement report (`label_card.md`) | MAP, MEASURE | MAP 3.4, MEASURE 2.11 | Harmful bias; value alignment               | AI data engineer + vendor PM | Per label task              |
| 8. PII / privacy      | PII scrub report + erasure-request log (`pii.md`)        | MAP, MEASURE, MANAGE | MAP 4.1, MEASURE 2.7, MANAGE 2.2 | Data privacy                           | Privacy + AI data engineer   | Per dataset build; on demand|
| 9. Versioning         | Dataset manifest with content hash + lineage (`manifest.json`) | GOVERN, MANAGE | GOVERN 1.6, MANAGE 4.1 | Information integrity                        | AI data engineer             | Per release                 |
| 10. Release           | Dataset card + sign-off (`datasheet.md`, `signoff.md`)    | GOVERN, MANAGE      | GOVERN 4.1, MANAGE 4.1 | Transparency / accountability                | AI data engineer + training lead | Per release             |

Column notes:

- **Function** vs. **Sub-cat.**: The four functions (GOVERN, MAP, MEASURE,
  MANAGE) are the top-level anchors. Sub-category codes are the fine-grained
  commitments; cite both so the map remains useful when NIST re-numbers a
  sub-category.
- **GenAI-Profile risk anchor**: NIST AI 600-1 enumerates twelve generative-AI
  risks (data privacy; harmful bias; confabulation; CBRN information; dangerous
  / violent recommendations; information integrity; obscene / abusive /
  degrading content; human–AI configuration; information security; intellectual
  property; value chain / component integration; and environmental impacts).
  Not every stage maps to every risk; the map should name only the risks the
  stage *materially moves*.
- **Owner** is a single accountable party. "AI data engineer + legal" is
  acceptable; "the team" is not.
- **Refresh cadence** is the trigger — "per release", "quarterly", "per
  training objective", "on request". A cadence of "as needed" is a red flag.

### 2.3 Gap list (worked example)

Common gaps that this mapping surfaces on a first pass:

| Gap                                                                                      | Why it matters                                                                                   | Candidate owner        |
|------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|------------------------|
| No written GOVERN-layer charter for the data-engineering function                        | Without a charter, GOVERN sub-categories have no custodian; the map is unsigned.                 | Data governance lead   |
| Licence policy exists, but no `noai` / `ai.txt` compliance log is emitted at acquisition | Breaks the audit path the licence policy depends on; MAP 4.1 is unverifiable.                    | AI data engineer       |
| Decontamination report names only public benchmarks; no internal eval overlap tracked    | MEASURE 2.3 can be satisfied for public benchmarks and still ship a contaminated private eval.   | AI data engineer       |
| Erasure-request handling is reactive only; no scheduled re-sweep of training data        | GDPR / CCPA MANAGE obligations require a mechanism; see Exercise 04.                             | Privacy + data eng     |
| No refresh-cadence calendar; artifacts drift without a trigger                           | Without triggers, the map decays into documentation theatre.                                     | Data governance lead   |

### 2.4 Narrative memo (worked example)

> **Position.** The AI data engineer owns stages 1–10 of the training-data
> pipeline and is the accountable party for the artifacts listed above. The
> model-training team is the downstream customer; model red-teaming,
> deployment, and in-production monitoring sit with the model-team roles and
> are out of scope for this map (see Exercise 05).
>
> **Primary anchors.** MAP 4.1 (data appropriateness), MEASURE 2.1 / 2.3
> (robustness and construct validity of the test corpus), MANAGE 2.2 (ongoing
> data-related risk mitigation), and GOVERN 1.1 / 1.6 (policies and roles).
>
> **GenAI-Profile risks tracked at the data layer:** data privacy, harmful
> bias, confabulation, intellectual property, information integrity. CBRN and
> value-chain risks are tracked but owned elsewhere.
>
> **Refresh cadence.** Per-release artifacts rebuild on every dataset version;
> quarterly artifacts (licence policy, charter) are reviewed on the first
> Monday of the quarter; per-request artifacts (erasure logs) are refreshed on
> intake.
>
> **Open items.** The five gaps listed above are tracked in the module
> backlog. Items that touch legal, privacy, or the model team carry explicit
> deferrals.

## 3. Validation steps

1. **Every pipeline stage you own has at least one artifact.** A stage with no
   artifact is a stage with no proof you did the work.
2. **Every artifact names a NIST AI RMF function.** If a learner cannot say
   whether an artifact is GOVERN, MAP, MEASURE, or MANAGE, they do not yet
   understand what the artifact is *for*.
3. **At least one sub-category is cited per artifact.** Top-level function
   names alone are too coarse to defend a decision; the sub-category code is
   the actual anchor.
4. **At least one GenAI-Profile risk is named** for every stage where
   generative models are plausibly downstream. Stages 4–8 almost always have a
   GenAI-Profile hook.
5. **Every artifact has a single named owner.** "The team" is not an owner.
6. **Every artifact has a refresh cadence that is a trigger, not a vibe.**
   "As needed" fails this check.
7. **A gap list exists and is non-empty.** A map that claims completeness on a
   first pass has not been stress-tested against the pipeline.
8. **Scope boundary is explicit.** The memo says what the AI data engineer
   does *not* own, and names the role that does.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 16/20 and loses zero points on the rows marked
**critical**.

| # | Dimension                                                                                                                | Critical? |
|---|--------------------------------------------------------------------------------------------------------------------------|-----------|
| 1 | Pipeline is decomposed into at least 8 stages from source selection through release                                      | Yes       |
| 2 | Every stage has a named artifact with a filetype (not just a prose description)                                          | Yes       |
| 3 | Every artifact maps to one of GOVERN / MAP / MEASURE / MANAGE                                                            | Yes       |
| 4 | At least one NIST AI RMF sub-category code is cited per artifact                                                         | Yes       |
| 5 | GenAI-Profile risks are named for curation, labelling, and privacy stages at minimum                                     | Yes       |
| 6 | Every artifact has a single human owner (role or role + person)                                                          | Yes       |
| 7 | Refresh cadence is a trigger ("per release", "quarterly", "on erasure request"), not a vibe                              | Yes       |
| 8 | A gap list exists, names at least three gaps, and assigns candidate owners                                               | Yes       |
| 9 | Narrative memo names what the AI data engineer does *not* own and who does                                               | No        |
| 10 | Links to downstream exercises in this module where each artifact is elaborated                                           | No        |

## 5. Common mistakes

- **Mapping to functions only, never sub-categories.** "This is GOVERN" is not
  a defensible map; which sub-category, and why?
- **Treating MAP as a one-time stage.** MAP continues through the pipeline —
  source selection *and* curation *and* labelling each have MAP obligations.
  Learners who put every MAP cell in stage 1 have misread the framework.
- **Skipping the GenAI-Profile column.** The GenAI-Profile risks are the
  concrete contemporary failure modes (confabulation, CBRN, IP). An AI RMF map
  that cannot name a GenAI-Profile risk is not a 2024+ map.
- **Owning model deployment.** The AI data engineer owns training-data; model
  evaluation, deployment, and monitoring belong to other roles. Saying
  otherwise will fail Exercise 05 later.
- **Refresh cadence = "ongoing".** Anything "ongoing" has no next-review date,
  which means it will not be reviewed.
- **Zero gaps on a first pass.** The output of a map is a to-do list, not a
  victory lap.
- **Omitting the erasure-request log.** Erasure is a MANAGE obligation and a
  GenAI-Profile data-privacy risk; omitting it is a classic trap. See
  Exercise 04.

## 6. References

- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (Core document; the four functions GOVERN / MAP / MEASURE / MANAGE and their
  sub-categories are the primary anchors for this exercise.)
- NIST AI 600-1 — *Artificial Intelligence Risk Management Framework:
  Generative Artificial Intelligence Profile* —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
  (Source for the twelve GenAI risks cited in column 5.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (Secondary lens for the acquisition and labelling rows — poisoning and
  supply-chain risks concretise MANAGE obligations.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Attacker-technique lens; browse the current matrix for technique IDs under
  the training-data poisoning and data-exfiltration tactics before citing a
  specific ID in a memo.)
- Local anchors: Exercise 02 (ISO/IEC 42001 + 5259 walk), Exercise 03 (EU AI
  Act Article 10), Exercise 04 (lawful-basis admit function), Exercise 05
  (role-scope and deferral contract).
