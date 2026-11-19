# Exercise 02 — Vendor RFP and Vendor-Audit Authoring (Solution)

## 1. Solution overview

This is a **design** exercise. The learner authors two documents:

1. A **Request for Proposal (RFP)** that a labeling vendor replies to
   with evidence (not marketing). The RFP is the written selection
   control: a vendor that cannot answer it is not eligible to
   handle training data.
2. A **vendor audit plan** that runs quarterly against the selected
   vendor. The audit converts the RFP's promises into measured
   evidence — annotator compensation, pool composition, quality
   outcomes, security posture, data-handling behaviour on real
   samples.

Both documents are artifacts, not informational writeups. A reviewer
should be able to run the RFP as a scorecard and run the audit as a
checklist against vendor-provided evidence files.

Governance framing. Labeling vendors are a **third-party data supply
chain** for the training set, so the control anchor is **NIST AI RMF
GOVERN 4.1** (accountability and oversight of third parties) combined
with **MANAGE 3.1** (third-party risk). The adversary-facing anchor
is **MITRE ATLAS AML.T0020 Poison Training Data** (a compromised or
coerced vendor is a plausible poisoning path) and **AML.T0010 ML
Supply Chain Compromise**. **OWASP ML Top 10 ML02:2023 Data
Poisoning Attack** is the control objective, and **ML06:2023 AI
Supply Chain Attacks** names the threat surface directly. Cite the
URLs under §6; do not paraphrase.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── rfp/
│   ├── 00_cover.md
│   ├── 01_scope_and_eligibility.md
│   ├── 02_commercial.md
│   ├── 03_quality.md
│   ├── 04_workforce.md
│   ├── 05_security_and_data_handling.md
│   ├── 06_governance_and_compliance.md
│   ├── 07_exit_and_portability.md
│   ├── 08_scoring_rubric.md
│   └── 99_evidence_manifest.md
├── audit/
│   ├── audit_plan.md
│   ├── sample_selection.md
│   ├── interview_questions.md
│   ├── canary_protocol.md
│   ├── findings_template.md
│   └── remediation_tracker.md
└── artifacts/
    ├── decision_log.md             # vendor selection decision + sign-offs
    └── quarterly_audit_report.md   # filled from findings_template each quarter
```

### 2.2 RFP — scope and eligibility (`rfp/01_scope_and_eligibility.md`)

```markdown
# Section 1 — Scope and eligibility

## 1.1 Program description
Program <program-id> labels <N> items/week for the <task> task family
(see `taxonomy.yaml` and `policy_note.md` from mod-107 ex-01 — attached).

## 1.2 Eligibility prerequisites
A vendor is **ineligible** (no score) if any of the following is false:

- Can commit, in writing, to the per-annotator per-batch cap in
  §3.3 of our labeling policy (≤ 15% share).
- Operates within jurisdictions that accept our DPA template
  (GDPR + CCPA counterpart; signed before any sample data flows).
- Has a documented information-security certification in force —
  SOC 2 Type II, ISO/IEC 27001:2022, or equivalent — issued within
  the last 12 months.
- Can accept sample data via a customer-controlled bucket with
  logged, scoped IAM — **no attachments, no public shares, no
  "test" credentials**.
- Does not sub-contract labeling to crowdsourced platforms without
  prior written approval for *that* program.

## 1.3 Submission mechanics
- Respond section by section; do not merge sections.
- Attach evidence (certificates, redacted sample QA reports, pay
  schedules) via the evidence manifest in §99. Unsupported claims
  score zero in that row.
```

### 2.3 RFP — commercial (`rfp/02_commercial.md`)

```markdown
# Section 2 — Commercial

| # | Question                                                             | Evidence required                    |
|---|----------------------------------------------------------------------|--------------------------------------|
| 2.1 | Unit cost per annotation at p50 complexity in our taxonomy          | Pricing schedule, sample complexity map |
| 2.2 | Price for a rework cycle triggered by failed IAA                    | Pricing schedule                     |
| 2.3 | Minimum batch commitment and surge capacity (weeks to +50%)          | Capacity letter from operations lead |
| 2.4 | Payment terms, invoice cadence, and currency                         | Draft MSA excerpt                    |
| 2.5 | What the vendor does when we pause or cancel a batch mid-flight      | Termination clause draft             |
```

### 2.4 RFP — quality (`rfp/03_quality.md`)

```markdown
# Section 3 — Quality

| #   | Question                                                                                            | Evidence required                                     |
|-----|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| 3.1 | Historical IAA on comparable tasks — report **Cohen's κ** and **Krippendorff's α** per batch, p50 / p90 | Last 6 batches, anonymised                            |
| 3.2 | Gold-set calibration procedure for seating a new annotator                                           | Internal SOP, redacted                                |
| 3.3 | Review (adjudication) coverage policy and reviewer-to-annotator ratio                                | Ratio history for the last 90 days                    |
| 3.4 | What triggers a batch reopen on your side (not ours)                                                 | Policy document                                        |
| 3.5 | Rationale-field or free-text capture for every annotation (yes/no)                                   | Screenshot of task UI                                 |
| 3.6 | How vendor detects and handles an annotator whose canary accuracy drops mid-batch                     | Written procedure + redacted example                  |
```

The IAA questions must ask for both κ **and** α — vendors that can
only report one of the two have not instrumented the pipeline mod-107
ex-03 will require.

### 2.5 RFP — workforce (`rfp/04_workforce.md`)

```markdown
# Section 4 — Workforce

| #   | Question                                                                                                     | Evidence required                   |
|-----|--------------------------------------------------------------------------------------------------------------|-------------------------------------|
| 4.1 | Hourly take-home pay for labelers on this program (gross and net of platform fees), by jurisdiction          | Pay schedule, redacted              |
| 4.2 | Does take-home pay meet or exceed the local living-wage benchmark cited by vendor? Provide the benchmark URL.| Benchmark citation + attestation    |
| 4.3 | Hours worked per labeler per week, p50 and p95, measured over last 90 days                                   | HR extract, redacted                |
| 4.4 | Mental-health / well-being programs for annotators exposed to sensitive content                              | Program description + uptake data   |
| 4.5 | Representation of language / dialect / geographic coverage in the annotator pool for our languages           | Pool composition statement          |
| 4.6 | Right-to-refuse policy: can a labeler decline a task and not lose session pay?                                | Written policy + example invocation |
| 4.7 | Employment classification (W-2/equivalent vs. contractor) per jurisdiction                                    | Employment summary                  |
```

Workforce questions are **not optional** and are not low-weight.
Under-paid or over-worked annotators produce lower-quality labels
and raise OWASP ML Top 10 ML02 insider-risk; this section is both an
ethics and a quality control.

### 2.6 RFP — security and data handling (`rfp/05_security_and_data_handling.md`)

```markdown
# Section 5 — Security and data handling

| #   | Question                                                                                           | Evidence required                       |
|-----|----------------------------------------------------------------------------------------------------|-----------------------------------------|
| 5.1 | Security certifications and their effective dates                                                  | SOC 2 / ISO 27001 report excerpt         |
| 5.2 | Where is sample data stored — region, provider, encryption at rest, encryption in transit          | Infra diagram + control statement        |
| 5.3 | Annotator client controls — clipboard, screen capture, VDI, personal-device usage                  | Workstation policy + configuration proof |
| 5.4 | Logging of annotator actions (who-saw-what) — retention and query access for the customer          | Log schema sample                        |
| 5.5 | How is sample data destroyed at batch close or contract end                                        | Deletion SOP + destruction attestation   |
| 5.6 | Breach-notification SLA (hours) and 24-hour contact                                                | Incident-response policy                  |
| 5.7 | Sub-processor list for data in scope (hosting, QA tools, messaging)                                 | Sub-processor register                   |
| 5.8 | Does the vendor use any data (ours or inferred) to train or improve a model?                        | Written attestation (binding)            |
```

Row 5.8 is the training-use lock: a vendor who reserves a right to
train on customer data is disqualified under our policy. The
attestation must be binding in the MSA, not just the RFP.

### 2.7 RFP — governance (`rfp/06_governance_and_compliance.md`)

```markdown
# Section 6 — Governance and compliance

| #   | Question                                                                                                      | Evidence required                           |
|-----|---------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| 6.1 | Jurisdictions where annotators work and applicable labour / data-protection regimes                            | Jurisdiction map                            |
| 6.2 | DPA / Standard Contractual Clauses for cross-border transfers                                                  | Draft DPA                                   |
| 6.3 | Child-safety policy for annotators under 18 (default: not permitted on this program)                            | Age-verification policy                     |
| 6.4 | Hate/violence/abuse content rotation policy and content-exposure caps per labeler per day                      | Policy document                              |
| 6.5 | AI-assistance policy for annotators — are they permitted to paste tasks into external LLMs?                    | Policy document + enforcement mechanism     |
| 6.6 | Whistleblower / anonymous reporting channel accessible to annotators                                            | Channel description                         |
| 6.7 | Named data-protection officer (DPO) and their jurisdiction                                                     | Contact                                     |
```

Row 6.5 is non-negotiable: unapproved LLM assistance leaks our prompts
and injects synthetic labels. The *enforcement mechanism* matters more
than the policy — "we trust our labelers" is not an enforcement
mechanism.

### 2.8 RFP — exit and portability (`rfp/07_exit_and_portability.md`)

```markdown
# Section 7 — Exit and portability

| #   | Question                                                                 | Evidence required         |
|-----|--------------------------------------------------------------------------|---------------------------|
| 7.1 | Data return format at contract exit (JSONL / Parquet / CSV + schema)      | Sample export             |
| 7.2 | Deletion attestation deliverable at exit — content and signing authority  | Attestation template      |
| 7.3 | How long are labels, logs, and sample data retained after exit            | Retention schedule        |
| 7.4 | Named successor path — can our next vendor ingest your exports as-is?     | Export schema + documentation |
```

### 2.9 RFP — scoring rubric (`rfp/08_scoring_rubric.md`)

Score each row 0 / 1 / 2 / 3; weights below sum to 100. Any
eligibility failure in §1.2 is terminal.

| Section | Weight | What a 3 looks like                                                                                           |
|---------|--------|---------------------------------------------------------------------------------------------------------------|
| 2 (Commercial)              | 10 | Price schedule itemised by complexity; rework priced separately; capacity letter signed by ops lead.           |
| 3 (Quality)                 | 25 | κ and α reported per batch for 6 batches; reviewer ratio > 1:10; mid-batch canary-drop response documented.    |
| 4 (Workforce)               | 25 | Pay > cited local living-wage benchmark; hours cap enforced; right-to-refuse invoked at least once in logs.    |
| 5 (Security + data)         | 20 | SOC 2 Type II or ISO 27001 current; no-training attestation binding; destruction SOP demonstrates witness.      |
| 6 (Governance + compliance) | 15 | AI-assistance enforcement tooled (clipboard controls, DLP); whistleblower channel independently administered.   |
| 7 (Exit + portability)      | 5  | Schema-documented export; deletion attestation template already drafted.                                        |

Sections 3, 4, and 5 are dominant by design — labeling quality,
workforce conditions, and data security are the three axes that
distinguish responsible vendors from marketing.

### 2.10 Audit plan (`audit/audit_plan.md`)

```markdown
# Vendor audit — quarterly

## Purpose
Convert RFP promises into measured evidence. An audit finding is
either **confirmed**, **refuted**, or **inconclusive** — never a
narrative.

## Cadence
- **Quarterly** full audit (this document).
- **Triggered** audit on: a canary-accuracy drop ≥ 10 pp in a single
  batch; a pay or hours complaint through the whistleblower channel;
  a reported sub-processor change; a security incident at the vendor
  or any of its sub-processors.

## Scope
1. Quality — sample 50 adjudicated items per batch across the
   quarter; re-adjudicate by an independent reviewer; compute κ and
   α against the vendor-reported numbers. Delta > 0.05 is a finding.
2. Workforce — request redacted pay stubs / hours extracts for a
   random 5% of annotators on our program; compare against the
   vendor's RFP answers and the local living-wage benchmark.
3. Security — request fresh evidence for every §5 item; witness one
   destruction event; replay 24 hours of annotator-action logs for
   a chosen batch; check sub-processor register deltas.
4. Governance — interview three annotators (via the vendor's
   independent channel) with the questions in `interview_questions.md`;
   re-verify DPA signatures; check AI-assistance enforcement logs.

## Deliverables
- `findings_template.md` filled per section.
- Remediation items opened in `remediation_tracker.md` with owners
  and 30/60/90-day deadlines.
- Sign-off from the AI-data-engineer, the data-governance lead, and
  the vendor's account owner on the quarterly report.
```

### 2.11 Sample selection (`audit/sample_selection.md`)

```markdown
# Sample selection

## Items (quality)
- Stratified random sample over `(batch_id, class, annotator_cohort)`.
- 50 items per batch; at least 5 per class present in the batch.
- RNG seed is recorded in the quarterly report.

## Annotators (workforce)
- Random 5% of annotators active on our program in the quarter.
- Request is for **redacted** pay / hours — annotator identity is
  never required; the vendor confirms the cohort size matches
  payroll.

## Batches (security + governance)
- The most recent closed batch + one batch from the quarter's start
  (bounds the window).
- One batch where a canary-accuracy drop was logged (if any).
```

### 2.12 Canary protocol (`audit/canary_protocol.md`)

```markdown
# Canary protocol

## Purpose
The auditor controls the canary set independently of the ex-01
canary rotation and can inject canaries without the vendor knowing
which items are canaries.

## Procedure
1. Auditor prepares a canary set of N items with gold answers; items
   are indistinguishable from production records (metadata routes
   through the standard `source` key, without a canary marker in
   the fields displayed to annotators).
2. Auditor ingests canaries via the normal batch submission path.
3. After batch close, auditor computes vendor accuracy on canaries
   from the exported results.

## Thresholds
- **≥ 0.92 accuracy:** pass.
- **0.85 – 0.92:** finding, "quality drift"; remediation within 30 days.
- **< 0.85:** critical finding; batch returns for rework; vendor placed
  on watch for the next quarter.
```

### 2.13 Interview questions (`audit/interview_questions.md`)

Keep anonymised; the annotator should not be identifiable to the vendor
after the interview. Ten representative prompts:

```markdown
1. How did you learn about this program? (Expect: vendor's platform.)
2. How many hours did you work on this program in the last week?
3. Have you ever used an external LLM to complete a task on this
   program? Were you aware of a policy that forbids this?
4. When you encountered content you did not want to label, were you
   aware you could refuse? Did you refuse? What happened?
5. What tools do you use on your workstation — is it VDI or your own
   computer?
6. Have you been paid on time for every pay period this quarter?
7. Is your take-home pay sufficient for your local cost of living?
8. Have you ever been reassigned to a different program against your
   stated preference?
9. Have you been asked to label content that felt outside the stated
   scope of this program?
10. Do you know how to reach a reporting channel outside your direct
    manager? Have you ever used it?
```

The interviewer records answers without annotator identifiers; raw
responses are retained for 90 days then destroyed.

### 2.14 Findings template (`audit/findings_template.md`)

```markdown
# Finding <id>

- **Date:** YYYY-MM-DD
- **Section:** <RFP §x.y or audit scope item>
- **Classification:** confirmed | refuted | inconclusive
- **Severity:** informational | low | medium | high | critical
- **Evidence refs:** <artifact paths / log IDs / interview summary IDs>
- **Impact:** <paragraph — e.g., κ under-reporting masks batch failure>
- **Remediation:** <paragraph with owner + deadline, tracked in `remediation_tracker.md`>
- **Reviewer sign-off:** <name, UTC timestamp>
```

## 3. Validation steps

1. **RFP is self-contained.** A fresh reader opens the `rfp/` tree
   and can score a vendor response without a side-channel briefing.
   Evidence requirements are concrete file types, not adjectives.
2. **Weights sum to 100.** `yq '.[].weight'` across the rubric rows
   (or simple arithmetic on the table) sums to 100.
3. **Sections 3+4+5 dominate.** Combined weight ≥ 70 (quality +
   workforce + security are the three load-bearing axes).
4. **Eligibility is terminal, not weighted.** A §1.2 failure cannot
   be offset by a strong §2 price.
5. **Audit has a stable cadence and clear triggers.** `audit_plan.md`
   names both quarterly and triggered audits; triggers are
   observable events, not vibes.
6. **Canary protocol is independent.** The auditor's canaries are
   separate from the ex-01 program canaries; the vendor cannot
   see-through a canary at label time.
7. **Interview channel is independent.** The annotator-interview
   contact point is not controlled by the vendor's account team.
8. **Deletion witness exists.** The audit witnesses at least one
   data-destruction event per quarter; a vendor that cannot
   demonstrate destruction is a critical finding.
9. **Findings are classified.** Every finding in the quarterly
   report resolves to `confirmed | refuted | inconclusive`. A
   "mostly yes" narrative is not a finding.
10. **Remediation ages out.** `remediation_tracker.md` shows
    deadline status; an item past 90 days with no owner is a
    reportable escalation to the data-governance lead.

## 4. Rubric or review checklist

Score each row 0–2. Pass threshold: ≥ 24/32 with zero zeros on
critical rows.

| #  | Dimension                                                                                                          | Critical? |
|----|--------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | RFP sections 1–7 are drafted as runnable scorecards with explicit evidence requirements                             | Yes       |
| 2  | Rubric weights sum to 100 and quality + workforce + security sum to ≥ 70                                            | Yes       |
| 3  | Eligibility prerequisites are listed and marked terminal                                                            | Yes       |
| 4  | Workforce section asks for pay, hours, right-to-refuse, and well-being — with evidence                              | Yes       |
| 5  | No-training-on-our-data attestation is required and documented as MSA-binding                                       | Yes       |
| 6  | Audit plan names quarterly + triggered cadence with concrete trigger events                                         | Yes       |
| 7  | Canary protocol is independent of program canaries and uses a vendor-opaque injection path                          | Yes       |
| 8  | Annotator interview channel is independent of the vendor account team                                               | Yes       |
| 9  | Destruction / exit evidence is demonstrable (not merely attested)                                                   | Yes       |
| 10 | Findings use a formal classification (confirmed/refuted/inconclusive) and severity scale                             | Yes       |
| 11 | Remediation tracker has owners, deadlines, and an escalation path                                                   | Yes       |
| 12 | Governance anchors (NIST AI RMF GOVERN 4.1 / MANAGE 3.1, OWASP ML02 / ML06, MITRE ATLAS AML.T0020 / T0010) cited     | No        |
| 13 | AI-assistance policy (§6.5) asks for enforcement mechanism, not just a written statement                             | Yes       |
| 14 | DPA / jurisdiction section (§6.1–6.3) binds cross-border transfer and age policy                                    | Yes       |
| 15 | Sub-processor register is required and reviewed each quarter for deltas                                             | Yes       |
| 16 | Exit section ensures schema-documented portability and deletion attestation                                         | No        |

## 5. Common mistakes

- **Asking vendors for *pricing* and calling it an RFP.** An RFP that
  scores primarily on price selects for low-pay vendors and buys
  insider-poisoning risk. Quality + workforce + security must
  dominate the rubric.
- **Accepting SOC 2 / ISO 27001 *in progress* as evidence.** Current
  effective dates only. "In progress" is a schedule, not a control.
- **Letting workforce questions be answered with marketing.** If a
  pay-schedule row is answered by "competitive wages", score zero.
  The RFP is evidence-driven.
- **Omitting the no-training attestation.** A vendor that trains
  on customer samples is a data-exfiltration path — the attestation
  must be binding and audit-verifiable in the MSA.
- **One-off audits.** A single audit cannot measure drift. Cadence
  must be quarterly plus triggered; the trigger list must name
  observable events, not sentiments.
- **Using the vendor's own canary set for the audit.** A vendor who
  curated the canaries can game canary accuracy. The auditor
  controls the independent canary protocol.
- **Interviewing annotators through the vendor's account team.**
  Annotators cannot answer freely if their livelihood is on the
  line with the person reading their answers.
- **Treating "exit" as a legal appendix.** Portability is a
  training-data continuity property; if the exit section is not
  tested, the vendor becomes a lock-in.
- **Scoring workforce alongside price.** Workforce is a safety and
  quality control; mixing it into price invites gaming. Separate
  weight, separate review.
- **Skipping the enforcement-mechanism question on AI-assistance.**
  Policy without enforcement is not policy; a reviewer who sees a
  policy document without clipboard / DLP / monitoring evidence
  scores that row zero.
- **Letting a finding be "mostly good".** Every finding must
  resolve to confirmed / refuted / inconclusive; qualitative
  summaries lose their teeth over time.
- **No remediation ownership.** An unowned remediation is an
  unresolved risk; the tracker names an owner and a deadline or
  the item is closed with a documented accept-risk.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack; ML06:2023 AI Supply Chain Attacks:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Vendor compromise is the canonical supply-chain poisoning path.
- **MITRE ATLAS — AML.T0020 Poison Training Data; AML.T0010 ML
  Supply Chain Compromise:** <https://atlas.mitre.org/>.
  Adversary techniques the RFP eligibility prerequisites and
  canary protocol are designed to deny.
- **NIST AI Risk Management Framework — GOVERN 4.1 (accountability
  and oversight); MANAGE 3.1 (third-party risk):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. Anchors
  the formal third-party oversight responsibility.
- **ISO/IEC 27001:2022** (information-security management systems)
  and **SOC 2 Type II** — the two certifications the §5.1 evidence
  requirement accepts. Issuance within the last 12 months is the
  control for currency.
- **NIST SP 800-88 Rev. 1** — media sanitisation reference for the
  §5.5 destruction SOP and the audit's destruction-witness event.
- **Partnership on AI — Responsible Sourcing of Data Enrichment
  Services:** practitioner guidance on workforce ethics the
  workforce section draws on; useful starting point for the
  living-wage benchmark question in §4.2. Use a current, named
  benchmark (e.g., MIT Living Wage Calculator for US jurisdictions)
  rather than the vendor's self-selected one.
- **Local anchors:** `mod-107` ex-01 (program policy note and
  taxonomy the RFP attaches); `mod-107` ex-03 (κ / α computation
  the audit re-runs on sampled items); `mod-101` ex-04/ex-05
  (GDPR/CCPA lawful-basis gate the DPA question depends on);
  `mod-109` (lineage signing of the batch exports the vendor
  produces); `mod-110` (PII handling within the labeling pipeline).
