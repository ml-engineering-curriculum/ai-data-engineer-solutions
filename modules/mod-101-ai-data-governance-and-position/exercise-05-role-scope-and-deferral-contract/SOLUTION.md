# Exercise 05 — Role Scope and the Deferral Contract (Solution)

## 1. Solution overview

This is a **design** exercise and the capstone for mod-101. The learner
produces two documents that together define what the AI data engineer *is*
and what the AI data engineer *is not*:

1. A **role-scope statement** — the one-page description of what the AI
   data engineer owns end-to-end, what they co-own with other roles, and
   what they explicitly do not own.
2. A **deferral contract** — the written agreement with the roles upstream
   and downstream of data engineering that names which decisions default to
   another owner, what information the data engineer must hand over to let
   the other owner decide, and what the SLAs on the handoff are.

Together these two documents make the module's earlier artifacts
*enforceable*. The NIST AI RMF map (Ex-01), the ISO walk (Ex-02), the AI
Act deliverables (Ex-03), and the admit function (Ex-04) each rely on
knowing who owns what. Without a written scope and a written deferral
contract, the data engineer eventually absorbs decisions that belong
elsewhere — bias triage, red-team scoping, deployment acceptance — and
the governance stack collapses into a single ambiguous role.

A good solution refuses scope creep *in writing* and names the counterparty
every refusal depends on.

## 2. Implementation — worked answer

### 2.1 Role-scope statement (worked example)

```markdown
# AI Data Engineer — Role Scope (v1.0)

## 1. Mission
Deliver training, validation, and testing datasets that are lawful, well
documented, and well understood — on a cadence the model team can plan
against — without displacing decisions that belong to Model, Safety, Legal,
Privacy, or Deployment.

## 2. Primary ownership (I own these end-to-end)
- Source selection and licence/consent gating (see mod-102).
- Acquisition, including `robots.txt` / `ai.txt` / `noai` compliance and
  rate-limit handling (see mod-103).
- Curation, filtering, deduplication, and benchmark decontamination
  (mod-104–106).
- Annotation vendor relationships and label-quality tracking (mod-107).
- PII scrub-pipeline operation and standing-erasure enforcement (mod-110).
- Dataset versioning, lineage, and release-time dataset cards (mod-109).
- The governance artifacts enumerated in mod-101 (NIST AI RMF map,
  ISO/IEC 42001 + 5259 crosswalk, AI Act Article 10 package, lawful-basis
  memo + admit function).

## 3. Co-owned (I contribute; another role signs)
- Licence policy — I draft, Legal signs.
- Lawful-basis memo — I draft, Privacy signs; Legal reviews.
- AI impact assessment (Clause 8.4 / Article 10) — I contribute the data
  section; Product/Compliance owns the whole assessment.
- Dataset card — I emit; the Model team acknowledges receipt.
- Decontamination posture vs. internal evals — I decontaminate, Model
  team names the evals.

## 4. Explicitly out of scope (I do not own)
- Model training, including hyperparameters and loss design.
- Model evaluation methodology beyond the training data's construct
  validity (I own the data side; Model owns the eval scorecard).
- Red-teaming and model safety evaluation (Safety role).
- Deployment decisions, including when a model is fit to ship (Model +
  Product + Compliance).
- Incident response for model outputs in production (Safety + On-call).
- Downstream user-facing policy (Product + Legal + Trust).

## 5. Handoff artifacts (what I deliver to whom)
- To Model team: dataset manifest, dataset card, decontamination report,
  DQ report.
- To Safety: redteam-ready dataset metadata, known-sensitive-source list,
  genAI-risk map from mod-101.
- To Legal: licence policy, per-source licence records, copyright-policy
  package (AI Act Annex XI if applicable).
- To Privacy: lawful-basis memo, admit-function policy, erasure-handling
  log, PII-scrub report.
- To Compliance: ISO/IEC 42001 crosswalk, AI Act Article 10 package,
  AIIA data section.

## 6. SLAs (default)
- Dataset build on request: 10 business days for a tracked training
  objective; 30 business days for a new training objective (new source
  selection required).
- Erasure request propagation: next scheduled rebuild within 30 days;
  emergency path within 5 business days.
- Legal / privacy sign-off requests: I provide a deliverable with
  ≥5 business days' lead time.

## 7. Escalation
- Any request that touches model behaviour, user-facing policy, or
  production incidents is deferred to the named owner. See the deferral
  contract.
```

### 2.2 Deferral contract (worked example)

```markdown
# AI Data Engineer — Deferral Contract (v1.0)

> A deferral is a standing agreement that a particular class of question
> is answered by someone else. Each deferral names the counterparty, the
> question class, the information I must provide, and the SLA.

## D-1. Red-team scope and findings triage → Safety
- **Deferred to:** Safety team lead.
- **I provide:** known-sensitive source list; data-side risks from the
  mod-101 NIST AI RMF + GenAI-Profile map; decontamination report.
- **Counterparty provides:** red-team plan; threat model; residual-risk
  verdict.
- **SLA:** data-side package ready ≥10 business days before the
  red-team's scoping meeting.

## D-2. Bias determination → Compliance (Fairness)
- **Deferred to:** Compliance (fairness lead), with Legal input where
  the system is EU-high-risk.
- **I provide:** representativeness and coverage report;
  bias-examination methodology; disaggregated metrics on splits the
  learner defines in writing.
- **Counterparty provides:** acceptance / rejection; remediation ask.
- **SLA:** report hand-over 15 business days before Article 10 sign-off.

## D-3. Lawful basis for processing → Privacy + Legal
- **Deferred to:** Privacy team, counter-signed by Legal.
- **I provide:** lawful-basis memo (Exercise 04); admit-function policy;
  erasure-request log.
- **Counterparty provides:** sign-off per source; exception approvals.
- **SLA:** draft memo at least 10 business days before any new source
  enters acquisition.

## D-4. Deployment readiness → Model + Product + Compliance
- **Deferred to:** Model team (release owner).
- **I provide:** dataset manifest, dataset card, DQ report,
  decontamination report, lineage manifest.
- **Counterparty provides:** the deployment decision.
- **SLA:** release package delivered 5 business days before the
  deployment-readiness review.

## D-5. Incident response → Safety + On-call
- **Deferred to:** Safety team (owner of model-output incidents).
- **I provide:** data-side forensics on request — which records could
  plausibly explain the behaviour (dedup/decontam gaps,
  source-provenance anomalies, labelling drift).
- **Counterparty provides:** incident owner, status, resolution.
- **SLA:** data-side forensics within 1 business day of the incident
  owner's request.

## D-6. User-facing data-rights intake → Trust/Privacy portal
- **Deferred to:** Privacy team (portal operator).
- **I provide:** mechanism (admit function + rebuild hook) to honour
  verified erasure and opt-out requests.
- **Counterparty provides:** verified request list on a cadence.
- **SLA:** request-list ingest once per sprint; emergency path 5
  business days.

## D-7. Vendor contractual obligations → Procurement + Legal
- **Deferred to:** Procurement (contract owner), Legal (terms author).
- **I provide:** technical obligations the vendor must meet (lawful-
  basis attestation, deletion SLA, breach notification windows).
- **Counterparty provides:** signed vendor paperwork.
- **SLA:** data-side requirements delivered before the first vendor
  conversation.

## D-8. Publication of the training-data summary → Communications + Legal
- **Deferred to:** Legal (sign-off) and Communications (publication).
- **I provide:** draft Annex XI summary (Exercise 03).
- **Counterparty provides:** final-form sign-off, publication location.
- **SLA:** draft ready 10 business days before release target.

## Exceptions
- Any deferral may be waived for a specific decision only with a
  written exception signed by both parties. Exceptions expire after 30
  days.

## Review cadence
- This contract is reviewed quarterly. Changes to ownership structure
  trigger an interim review.
```

### 2.3 Boundary diagram (textual form)

A compact textual diagram the learner can paste into a design doc; a
reviewer should be able to trace every artifact to exactly one owner:

```
          +------------------- Compliance ----------------+
          |  AIIA; EU AI Act sign-off; fairness verdicts  |
          +-----------------------------------------------+
                            ^              ^
                            | (verdict)    | (sign)
                            |              |
+---------- Legal -----+   +------- Privacy -----+
| licence policy sign; |   | lawful-basis sign;   |
| TDM-opt-out posture; |   | admit-function sign; |
| copyright policy     |   | erasure routing      |
+----------------------+   +----------------------+
         ^                                   ^
         | (draft)                           | (draft)
         |                                   |
   +===============  AI Data Engineer  =============+
   |  sources - acquisition - curation - dedup -    |
   |  decontam - labelling - PII - versioning -     |
   |  dataset card - handoff                        |
   +================================================+
         |                                   |
         v (manifest+card)                   v (sensitive-source list)
+------- Model team ------+       +--------- Safety -----------+
| eval scorecard; release |       | red-team; incident response|
+-------------------------+       +----------------------------+
```

## 3. Validation steps

1. **Role scope has a "not mine" section.** A scope statement that only
   enumerates what the engineer owns is a scope statement that has not been
   stress-tested.
2. **Every "co-owned" item names the signer.** "Co-owned with Legal" is
   not enough — who at Legal signs?
3. **Every deferral names a counterparty, a payload, and an SLA.** All
   three are needed; a deferral missing the payload is a wish.
4. **Deferrals reference the deliverables from Exercises 01–04.** If the
   deferral contract cannot cite the artifacts the module already built,
   the capstone has not integrated.
5. **The admit function's obligations are honoured in the deferral
   contract.** PII-scrub, lawful-basis attestation, and erasure routing
   must each have an owner on paper.
6. **Deployment is deferred, not owned.** If the role-scope section
   reads "I decide the model is ready to ship", the exercise has failed.
7. **The contract has a review cadence and an exceptions clause.** Without
   both, the contract decays silently.
8. **Boundary diagram is traceable.** Pick any artifact; one and only one
   owner is named.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 18/24 and loses zero points on the rows marked
**critical**.

| #  | Dimension                                                                                                              | Critical? |
|----|------------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Role-scope statement enumerates owned, co-owned, and *not owned* responsibilities                                       | Yes       |
| 2  | Every co-owned responsibility names a signer role (not a team-at-large)                                                 | Yes       |
| 3  | Deferral contract has at least six distinct deferrals, including red-team, lawful basis, deployment, and incident       | Yes       |
| 4  | Every deferral has counterparty, payload, SLA                                                                           | Yes       |
| 5  | Deferral contract cites the artifacts from Exercises 01–04                                                              | Yes       |
| 6  | Contract names a review cadence and an exception mechanism                                                              | Yes       |
| 7  | Role scope explicitly refuses deployment-readiness ownership                                                            | Yes       |
| 8  | SLAs are numeric (days or sprints), not vibes                                                                           | Yes       |
| 9  | Boundary diagram or equivalent is included and traceable                                                                | No        |
| 10 | AI impact assessment ownership is allocated correctly (contributor, not owner, from the data-engineering seat)          | Yes       |
| 11 | Vendor contractual obligations are deferred to Procurement + Legal, not absorbed                                        | No        |
| 12 | Open follow-up notes are used where counterparty roles do not yet exist in the organisation                             | No        |

## 5. Common mistakes

- **Owning everything that touches data.** "If it has data, it's mine."
  Bias triage touches data; it still belongs to Compliance / Fairness.
- **Deferring without a payload.** "Legal signs the licence policy" is
  not actionable — what do you hand to Legal, and by when?
- **Deferrals without SLAs.** "On request" is a non-SLA. "10 business
  days before the scoping meeting" is an SLA.
- **No exceptions clause.** Deferrals must bend for real incidents; a
  contract that cannot bend is bypassed and becomes decorative.
- **Collapsing Privacy and Legal.** They have overlapping but distinct
  scopes; GDPR / CCPA sign-off is Privacy's, licence-policy sign-off is
  Legal's.
- **Making the data engineer the incident owner.** The data engineer
  provides forensics; the incident owner is Safety.
- **Making the data engineer the fairness arbiter.** Representativeness
  and bias examination are data-engineering deliverables; the verdict is
  Compliance's.
- **No review cadence.** A quarterly review is the minimum credible
  cadence for a role-scope contract.
- **Treating the contract as internal only.** The deferral contract is
  the artifact a new team member reads on day one; keep it readable.

## 6. References

- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (GOVERN function sub-categories cover roles, responsibilities, and
  accountability; the role-scope statement is a GOVERN artifact.)
- NIST AI 600-1 — *Generative AI Profile* —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
  (Human–AI configuration and value-chain risks are the two
  GenAI-Profile risks most directly affected by role clarity.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (Supply-chain and operational-practice risks that a deferral contract
  is the organisational control for.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Techniques that cross role boundaries — data poisoning, supply-chain
  compromise — are easier to defend against when roles are written down.)
- Regulation (EU) 2024/1689 (the AI Act) — the AIIA ownership question
  under Article 10 is the clearest external forcing function for a
  deferral contract.
- VeriSwarm Trust Center — <https://veriswarm.ai/trust>
  (Practitioner reference: example of how a vendor publishes a
  responsibility-matrix / deferral posture externally. Use only as a
  format example.)
- Local anchors: Exercise 01 (NIST AI RMF map — owners column),
  Exercise 02 (ISO walk — clause 5 "Leadership" ownership rows),
  Exercise 03 (AI Act deliverable owners), Exercise 04 (admit-function
  signer and payload for the Privacy deferral).
