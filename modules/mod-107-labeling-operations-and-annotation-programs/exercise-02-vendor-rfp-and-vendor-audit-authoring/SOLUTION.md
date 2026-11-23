# Exercise 02 — Vendor RFP and Vendor-Audit Authoring (Solution)

> **Exercise-prompt note.** The paired learning file
> (`exercise-02-vendor-rfp-and-vendor-audit-authoring.md` in the
> `ai-data-engineer-learning` repo) is not visible from the
> solutions repo at generation time. This solution is grounded in
> the exercise title, the module README, and the official
> frameworks cited in §6 (NIST AI RMF, OWASP ML Top 10, MITRE
> ATLAS). Any divergence from the exercise prompt once it is
> pinned should re-ground against those sources.
> <!-- pending-verify: re-check against the exercise prompt once
> the learning file is reachable to confirm page budget and
> required vendor categories. -->

## 1. Solution overview

This is a **design** exercise. The learner authors two linked
artifacts: a **Request For Proposal (`RFP.md`)** that the team
sends to candidate annotation vendors, and a **Vendor Audit
Workbook (`AUDIT.md` plus `audit_checklist.yaml`)** that a peer
auditor can run against a signed vendor the team is already
working with. The two artifacts share one data model: everything
the RFP asks a vendor to commit to is something the audit later
measures against.

Deliverables:

- `RFP.md` — the Request For Proposal. Six sections, one signatory
  on the sending side (ai-data-engineer), one signatory on the
  responding side (vendor account manager). SemVer at the document
  level.
- `audit_checklist.yaml` — a machine-readable checklist whose keys
  are the RFP's asks. Status enum: `pass | fail | needs-evidence`.
- `AUDIT.md` — the human-readable audit guide. Walks a peer
  through how to execute each item in `audit_checklist.yaml`.
- `scoring/rubric.md` — the RFP-response scoring rubric. 0/1/2
  per row, with critical rows that any zero fails the whole
  response.
- `rationale.md` — the decision log. For each RFP clause: what it
  prevents, what it does not prevent, and what weakens if the
  clause is dropped.
- `worked_example/` — a redacted, worked example RFP response and
  a corresponding worked audit run. Uses placeholder vendor
  names. No real vendor names unless the learner has primary-source
  citations.

The exercise is design-only. No vendor is actually onboarded; the
learner demonstrates they can specify a vendor relationship that
the rest of `mod-107` (IAA in Exercise 03, active learning in
Exercise 04, DPO labeling in Exercise 06) can plug into
without rewriting.

Governance framing. An annotation vendor is a third-party data
producer whose output enters the training set. The NIST AI Risk
Management Framework anchor is **GOVERN 6.1** (policies and
procedures are in place to address AI risks and benefits arising
from third-party software and data), reinforced by **GOVERN 6.2**
(contingency processes for third-party AI failures). OWASP ML Top
10 **ML06:2023 ML Supply Chain Vulnerabilities** is the harm
anchor — a vendor is an upstream supply-chain link. **ML02:2023
Data Poisoning Attack** is the downstream harm the audit
prevents (a vendor shipping systematically biased or adversarially
labeled data). MITRE ATLAS **AML.T0019 Publish Poisoned Datasets**
is the adversary-facing technique for a hostile or compromised
vendor. Cite the framework URLs rather than paraphrasing the
control text.

## 2. Implementation — worked answer (the two artifacts)

### 2.1 Project layout

```
exercise-02/
├── README.md
├── RFP.md
├── audit_checklist.yaml
├── AUDIT.md
├── scoring/
│   └── rubric.md
├── rationale.md
└── worked_example/
    ├── vendor_response.md
    ├── audit_run.yaml
    └── scorecard.md
```

### 2.2 `RFP.md` — structure

```markdown
# Request For Proposal — Annotation Services, program v1

| Field                  | Value                                                     |
|------------------------|-----------------------------------------------------------|
| RFP version            | 1.0.0                                                     |
| Issued                 | 2026-11-17                                                |
| Issuing team           | ai-data-engineer (project `mod-107`)                      |
| Signatory (issuer)     | <ai-data-engineer email>                                  |
| Signatory (responder)  | vendor account manager                                    |
| Response window        | 21 calendar days from issue                               |
| Confidentiality        | Responses are mutual NDA from issue. Sample data is DO-NOT-RETRAIN.|
| Supersedes             | n/a                                                       |

## 1. Scope of work

Describe the labeling program the vendor would run. Pin one
answer per numbered item; "to be confirmed" is a response-breaking
answer on items (a)–(e).

(a) Program volume: expected monthly task count and per-task
cost unit (per-task, per-hour, per-span).
(b) Label schema version the vendor will implement (see attached
`task_record.schema.json` from Exercise 01).
(c) Languages in scope. For non-English, the vendor commits to
native-speaker annotators only; attestation required.
(d) Modality: text, image, audio, video, or a mix. Each modality
has a separate price line.
(e) PII posture: whether tasks contain PII, and whether the
vendor's workforce may see it. See §4 for the required attestation.

## 2. Workforce composition and governance

The vendor commits to the following, measurable per quarter:

(a) Headcount of trained annotators per language.
(b) Reviewer-to-annotator ratio (minimum 1:5).
(c) Annotator turnover rate (reported as the fraction of
annotators who left the project in the preceding 90 days).
(d) Attested worker compensation model: hourly, per-task, or
hybrid. Per-task-only models require a measured minimum effective
hourly rate at the local statutory wage floor; the vendor
reports this number quarterly.
(e) Worker welfare contact: a named point of contact at the vendor
for wage, safety, and content-exposure concerns. Required to be a
natural person, not an alias.

## 3. Quality program

(a) Inter-annotator agreement baseline: vendor commits to a
baseline IAA (e.g., Cohen's kappa ≥ 0.7) on a double-labeled
cohort, measured per label set, reported weekly.
(b) Gold-set hit rate: vendor commits to a minimum
precision/recall against the issuer-provided gold set, reported
weekly.
(c) Review-rejection rate: vendor caps review-rejection at a
stated ceiling; anything above the ceiling triggers §6.
(d) Guidelines version the workforce is operating under; the
vendor confirms within three business days of a guidelines
update.
(e) Calibration cadence: initial and monthly calibration sessions
between vendor leads and issuer leads.

## 4. Data handling and PII

(a) Data at rest and in transit is encrypted (TLS 1.2+ in
transit, AES-256 at rest). Attest.
(b) Annotators access data only through the labeling platform
(no local downloads, no screenshots allowed).
(c) PII-sensitive projects require workforce under the vendor's
PII-training program, with the training completion record
auditable.
(d) Data residency: issuer names the acceptable countries; vendor
states where the workforce and the infrastructure sit.
(e) Retention: vendor deletes raw task data within N days of
program end, per issuer policy. Attest the deletion window.

## 5. Security posture

(a) Named security certifications in force (e.g., ISO/IEC 27001,
SOC 2 Type II), with certificate numbers and expiry dates. Vendor
provides the certificates on request.
(b) Named recent third-party security audit; summary available
on request under NDA.
(c) Incident-response SLA: stated time to notify issuer of a
suspected data incident.
(d) Subprocessor list: current, with roles and data-exposure
scope per subprocessor.

## 6. Hand-off, measurement, and dispute

(a) Hand-off format: the normalised task-record schema from
Exercise 01. Vendor does not define its own export format.
(b) Measurement surface: vendor agrees to the rows in §4 of the
`CONTRACT` sibling document (see mod-106 exercise-04 for an
example measurement-surface table; this RFP uses the same shape).
(c) Dispute path: a disputed batch is held; vendor and issuer
root-cause within five business days.
(d) Termination-for-cause triggers: (i) a batch found to be
systematically poisoned, (ii) workforce welfare violation, (iii)
a security incident the vendor failed to disclose within the §5
SLA.
(e) Termination-for-convenience notice: 30 calendar days.

## 7. Pricing and payment

(a) Per-unit price per modality.
(b) Volume tiers.
(c) Payment cadence: net 30 default, open to negotiation.
(d) Price escalation: at most CPI-indexed, written notice 60 days
in advance.

## 8. Response format

- A written response to §§1–7 in Markdown, PDF, or DOCX.
- A completed `audit_checklist.yaml` with status
  `needs-evidence` on every item the vendor will provide at
  onboarding, and `pass` or `fail` on every item answerable
  at RFP time.
- One reference customer the issuer may contact directly.
- Three illustrative vendor outputs (or synthetic equivalents),
  so issuer leads can calibrate the vendor's label quality
  before any contract is signed.

## 9. Signatures

<cosign signature paths or equivalent>
```

The RFP is intentionally section-numbered and cross-referenced
from `audit_checklist.yaml`. Section numbers are stable across
minor version bumps; renumbering is a major bump.

### 2.3 `audit_checklist.yaml`

Each check has a stable key, a reference back to the RFP section
that defined the ask, an evidence type (`doc | observation |
query | attestation`), and a default status of `needs-evidence`.
The auditor fills the status in during an audit run.

```yaml
# audit_checklist.yaml  — v1.0.0
rfp_version: 1.0.0
checks:
  - id: scope.volume
    rfp_ref: "1(a)"
    description: Vendor reports monthly task volume and per-unit cost.
    evidence_type: doc
    status: needs-evidence

  - id: scope.label_schema
    rfp_ref: "1(b)"
    description: Vendor outputs match `task_record.schema.json` v1.0.0.
    evidence_type: query
    evidence_query: "jsonschema -i vendor_batch.json task_record.schema.json"
    status: needs-evidence

  - id: workforce.ratio
    rfp_ref: "2(b)"
    description: Reviewer-to-annotator ratio ≥ 1:5.
    evidence_type: doc
    status: needs-evidence

  - id: workforce.turnover
    rfp_ref: "2(c)"
    description: Annotator 90-day turnover rate stated.
    evidence_type: doc
    status: needs-evidence

  - id: workforce.compensation
    rfp_ref: "2(d)"
    description: Effective hourly rate at or above local statutory wage.
    evidence_type: attestation
    status: needs-evidence

  - id: workforce.welfare_contact
    rfp_ref: "2(e)"
    description: Named natural person at the vendor for welfare concerns.
    evidence_type: doc
    status: needs-evidence

  - id: quality.iaa_baseline
    rfp_ref: "3(a)"
    description: IAA baseline (kappa or Krippendorff) committed per label set.
    evidence_type: query
    evidence_query: "see mod-107 exercise-03"
    status: needs-evidence

  - id: quality.gold_hit
    rfp_ref: "3(b)"
    description: Weekly gold-set hit-rate committed and reported.
    evidence_type: observation
    status: needs-evidence

  - id: quality.review_rejection
    rfp_ref: "3(c)"
    description: Review-rejection rate ceiling stated.
    evidence_type: doc
    status: needs-evidence

  - id: data.encryption
    rfp_ref: "4(a)"
    description: TLS 1.2+ in transit, AES-256 at rest.
    evidence_type: attestation
    status: needs-evidence

  - id: data.no_local_copy
    rfp_ref: "4(b)"
    description: Annotators cannot download, export, or screenshot tasks.
    evidence_type: observation
    status: needs-evidence

  - id: data.pii_training
    rfp_ref: "4(c)"
    description: Workforce under the PII-training program (auditable record).
    evidence_type: doc
    status: needs-evidence

  - id: data.residency
    rfp_ref: "4(d)"
    description: Workforce and infrastructure sit in approved countries.
    evidence_type: doc
    status: needs-evidence

  - id: data.retention
    rfp_ref: "4(e)"
    description: Deletion window stated and enforced in writing.
    evidence_type: attestation
    status: needs-evidence

  - id: security.certifications
    rfp_ref: "5(a)"
    description: ISO/IEC 27001 or SOC 2 Type II or equivalent, in force.
    evidence_type: doc
    status: needs-evidence

  - id: security.audit
    rfp_ref: "5(b)"
    description: Recent independent security audit summary available.
    evidence_type: doc
    status: needs-evidence

  - id: security.incident_sla
    rfp_ref: "5(c)"
    description: Incident-response notification SLA ≤ 72 hours.
    evidence_type: attestation
    status: needs-evidence

  - id: security.subprocessors
    rfp_ref: "5(d)"
    description: Subprocessor list current and reviewed.
    evidence_type: doc
    status: needs-evidence

  - id: handoff.schema
    rfp_ref: "6(a)"
    description: Vendor exports via `task_record.schema.json` without custom fields.
    evidence_type: query
    evidence_query: "jsonschema -i vendor_batch.json task_record.schema.json"
    status: needs-evidence

  - id: handoff.measurement
    rfp_ref: "6(b)"
    description: Vendor emits the measurement-surface columns weekly.
    evidence_type: query
    status: needs-evidence

  - id: handoff.dispute
    rfp_ref: "6(c)"
    description: Dispute path and 5-business-day SLA accepted.
    evidence_type: doc
    status: needs-evidence

  - id: handoff.termination_triggers
    rfp_ref: "6(d)"
    description: Named termination-for-cause triggers accepted.
    evidence_type: doc
    status: needs-evidence
```

### 2.4 `AUDIT.md` — how to run the checklist

```markdown
# Vendor audit workbook — v1.0.0

## 1. When to run

- Quarterly for every vendor on an active program.
- Unscheduled, within five business days, on any of:
  (i) a batch flagged for systematic quality regression by
  mod-111;
  (ii) a disclosed security incident at the vendor;
  (iii) a workforce-welfare complaint received by the issuer.

## 2. Preparation

- Load `audit_checklist.yaml`. Confirm the `rfp_version` matches
  the contract in force.
- Pull the vendor's most recent `vendor_batch.json` export.
- Pull the measurement-surface rows for the last four weeks.

## 3. Executing a check

For each entry:

- `evidence_type: doc` — request the document; record its SHA-256
  in `audit_run.yaml`.
- `evidence_type: query` — run the command in `evidence_query`;
  attach the exit code and a short summary of the output.
- `evidence_type: observation` — the auditor observes a live
  labeling session or a recorded sample; record who observed and
  when.
- `evidence_type: attestation` — written attestation signed by
  the vendor; record the signer and the signature SHA-256.

Set the check's status to `pass`, `fail`, or (if evidence is
still outstanding) `needs-evidence`. Every `fail` requires a
root-cause note and an owner.

## 4. Scoring the run

Audit passes if:
- Every critical check (`scope.*`, `handoff.schema`,
  `handoff.measurement`, `data.encryption`, `data.no_local_copy`,
  `security.incident_sla`, `workforce.compensation`) is `pass`.
- Overall fail rate across non-critical checks is below 20%.

Otherwise the audit fails and the vendor enters the §6-dispute
path.

## 5. Writing the audit report

A one-page summary of pass/fail totals, open findings, and the
auditor's recommendation: {continue, continue-with-remediation,
suspend, terminate}. Signed by the auditor (not the ai-data-
engineer running the program).
```

### 2.5 `scoring/rubric.md`

```markdown
# RFP-response scoring rubric — v1.0.0

Each row scored 0/1/2 (0 = missing, 1 = present but shallow,
2 = defensible with evidence). Pass threshold: ≥ 20/28 with no
zeros on critical rows.

| #  | Row                                                                                             | Critical? |
|----|-------------------------------------------------------------------------------------------------|-----------|
| 1  | Scope section pins volume, modality, languages, and label-schema version                        | Yes       |
| 2  | Workforce compensation line answers the attestation (rate, local wage floor)                    | Yes       |
| 3  | Workforce welfare contact is a natural person                                                   | Yes       |
| 4  | IAA baseline and gold-hit baseline stated per label set                                         | Yes       |
| 5  | PII attestation present if any task may contain PII                                             | Yes       |
| 6  | Data-residency answer names countries, not regions                                              | Yes       |
| 7  | Security certifications in force and dated                                                      | Yes       |
| 8  | Incident-response SLA ≤ 72 hours                                                                | Yes       |
| 9  | Subprocessor list present and not "on request"                                                  | No        |
| 10 | Hand-off uses `task_record.schema.json` without custom fields                                   | Yes       |
| 11 | Dispute SLA ≤ 5 business days accepted                                                          | Yes       |
| 12 | Termination-for-cause triggers listed verbatim                                                  | No        |
| 13 | Pricing structure is per-unit plus volume tiers, not opaque                                     | No        |
| 14 | Response includes one contactable reference and three example outputs                           | No        |
```

### 2.6 `rationale.md` — decision log

Representative entries:

- **§2(d) workforce compensation attestation is load-bearing.**
  Per-task-only pay with no measured effective hourly rate
  creates pressure to rush low-quality labels at or below a
  statutory wage floor. The attestation closes the "the market
  sets the price" loophole that some vendors invoke to avoid
  disclosing worker pay.
- **§3(a) IAA baseline committed before onboarding.** Measuring
  IAA post-hoc ("we'll tell you what our IAA was last quarter")
  lets a vendor cherry-pick metrics. Committing a baseline in the
  RFP makes the target prospective.
- **§4(b) no local downloads is a hard line.** Any vendor that
  allows local copies of task data has, in practice, an
  uncontrolled retention policy — the data can be exfiltrated
  through a worker's device even if the vendor's servers are
  compliant.
- **§5(c) 72-hour incident-response SLA.** A longer SLA (seven
  days, "best effort") makes it impossible for the issuer to
  meet its own downstream notification obligations.
- **§6(a) hand-off format is the issuer's schema.** This prevents
  the "vendor emits a custom CSV" pattern that forces the issuer
  to maintain N bespoke adapters. The schema is version-pinned
  in Exercise 01; renaming or dropping fields is a contract-
  version bump.
- **§6(d) termination-for-cause triggers listed explicitly.**
  Reserving termination-for-cause to "the issuer's reasonable
  judgement" is a non-starter. A named list prevents disputes
  about whether a given failure is in scope.
- **Audit `workforce.compensation` is `attestation`, not `doc`.**
  Vendors rarely share pay stubs. A signed attestation with the
  vendor's name on it is the enforceable surface.

### 2.7 `worked_example/`

`worked_example/vendor_response.md` is a short, redacted vendor
response that touches every RFP section with placeholder values
(e.g., company name `VendorCo`, pricing in `$X per task`). It is
illustrative, not a template to be shipped verbatim.

`worked_example/audit_run.yaml` fills `audit_checklist.yaml` with
realistic statuses — most `pass`, a few `fail`, one
`needs-evidence` for a subprocessor list that is "available on
request".

`worked_example/scorecard.md` applies the rubric and lands at a
single recommendation (`continue-with-remediation`), citing the
specific failing checks.

## 3. Validation steps

1. **RFP version bumps are documented.** `grep "RFP version"
   RFP.md` returns one SemVer string that equals
   `audit_checklist.yaml`'s `rfp_version`. Mismatch fails
   validation.
2. **Every audit check has an RFP cross-reference.** `yq '.checks
   | map(select(.rfp_ref == null)) | length'
   audit_checklist.yaml` returns `0`.
3. **Critical rows are covered in both artifacts.** For every
   critical row in the scoring rubric, an entry with the same ask
   exists in `audit_checklist.yaml`. Missing coverage fails
   review.
4. **PII clause is non-empty when PII is in scope.** If
   `scope.pii` in the vendor's response is `true`, both the RFP
   §4 answer and the matching `audit_checklist.yaml` entries
   `data.pii_training` and `data.no_local_copy` must be
   populated — a blank here is a response-breaking omission.
5. **Dispute SLA ≤ 5 business days.** `grep "5 business days"
   RFP.md` returns the dispute SLA line; weaker SLAs fail
   validation.
6. **Workforce welfare contact is a natural person.** The vendor
   response names an individual, not an alias or a function
   mailbox. If the worked example redacts the name, the redaction
   itself must be flagged as redacted-for-review.
7. **Security certifications are current.** Each cert name has a
   non-null expiry date in the future.
8. **Hand-off schema exercise.** A sample `vendor_batch.json`
   round-trips through `jsonschema -i vendor_batch.json
   task_record.schema.json` and exits 0.
9. **No invented facts.** Any vendor name referenced in
   `worked_example/` is clearly flagged as placeholder.
   Specific statutory wages or jurisdictions cited elsewhere
   carry a primary-source footnote in `rationale.md`; if no
   primary source is yet at hand, leave the figure out rather
   than invent one, and open a tracking item for the citation.
10. **Audit is runnable by a peer.** A second engineer running
    `AUDIT.md` against the worked example lands at the same
    recommendation (`continue-with-remediation`) without
    consulting the author.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with no zeros
on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `RFP.md` has §1 scope, §2 workforce, §3 quality, §4 data, §5 security, §6 hand-off, §7 pricing, §8 response, §9 sig | Yes       |
| 2  | `audit_checklist.yaml` has one entry per critical RFP ask with `rfp_ref` populated                                   | Yes       |
| 3  | Hand-off clause binds vendor to `task_record.schema.json` and forbids custom fields                                 | Yes       |
| 4  | Workforce-welfare contact is a natural person, not a function alias                                                  | Yes       |
| 5  | Compensation attestation is required and has a measurable floor                                                     | Yes       |
| 6  | PII clause is present and covers no-local-copy, encryption, retention                                               | Yes       |
| 7  | Security certifications line names concrete certs (ISO 27001 / SOC 2 / equivalent)                                  | Yes       |
| 8  | Incident-response SLA ≤ 72 hours                                                                                     | Yes       |
| 9  | Dispute SLA ≤ 5 business days                                                                                        | Yes       |
| 10 | Termination-for-cause triggers listed explicitly                                                                    | No        |
| 11 | Scoring rubric uses 0/1/2 with critical-row zeros = auto-fail                                                       | Yes       |
| 12 | Rationale log answers "what does this clause prevent" and "what weakens if dropped" per clause                      | Yes       |
| 13 | Worked example runs end-to-end (RFP response → audit run → scorecard)                                               | Yes       |
| 14 | Governance anchors (NIST AI RMF GOVERN 6.1 / 6.2, OWASP ML Top 10 ML02 / ML06, MITRE ATLAS AML.T0019) cited          | No        |

## 5. Common mistakes

- **Letting the vendor set the hand-off format.** A vendor that
  defines its own CSV or JSON export leaves the issuer carrying
  the integration debt. Pin `task_record.schema.json` as the only
  acceptable hand-off.
- **Omitting workforce welfare.** A labeling program is a labour
  program. An RFP that treats the workforce as a hidden
  subprocess is a reputational and compliance risk; §2(d) and
  §2(e) are not optional.
- **Scoring with a single aggregate number.** A rubric that lets a
  vendor pass by scoring high on "pricing clarity" while failing
  "no local copies" is designed to let bad vendors through.
  Critical-row zeros must be auto-fails.
- **"Available on request" subprocessor lists.** The list is in
  scope for the RFP; a lazy pointer to a web portal is not an
  answer.
- **Treating certifications as a yes/no.** Record the cert number
  and expiry. An expired SOC 2 is worse than none because it
  invites complacency.
- **Writing the RFP without the audit.** The two must be written
  together; if the audit cannot measure a clause, the clause is
  aspirational, not operational.
- **No termination-for-convenience path.** A program with no exit
  ramp is a program the issuer is locked into when the vendor
  starts missing SLAs.
- **Citing specific vendor names without primary sources.** Any
  named vendor in the worked example must be labelled as
  placeholder, or carry a verifiable citation in `rationale.md`.
- **Pre-signing the audit.** Audit sign-off is a separate function
  from running the program; the auditor is not the ai-data-
  engineer who authored the RFP.

## 6. References

- **NIST AI Risk Management Framework — GOVERN 6.1 (third-party
  data and software), GOVERN 6.2 (contingency processes for
  third-party AI failures):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. The
  governance anchors for an annotation-vendor relationship; the
  RFP operationalises GOVERN 6.1 and the termination-for-cause
  path operationalises GOVERN 6.2.
- **OWASP Machine Learning Security Top 10 — ML02:2023 Data
  Poisoning Attack, ML06:2023 ML Supply Chain Vulnerabilities:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML06 is the direct anchor (the vendor is the supply chain);
  ML02 is the downstream harm the audit is designed to prevent.
- **MITRE ATLAS — AML.T0019 Publish Poisoned Datasets:**
  <https://atlas.mitre.org/>. The adversary-facing technique for
  a hostile or compromised vendor.
- **ISO/IEC 27001 and SOC 2 Type II:** these are the concrete
  certification regimes the RFP §5 names. The RFP does not
  require a specific regime; it requires one that is in force,
  dated, and auditable.
- **Local anchors:** Exercise 01 (`task_record.schema.json` is
  the hand-off format §6 binds the vendor to); Exercise 03 (the
  IAA baseline §3(a) commits the vendor to); Exercise 04 (active-
  learning queue depends on the vendor honouring §3 quality
  commitments); `mod-106` exercise-04 (the measurement-surface
  table shape this RFP reuses); `mod-109` (records the vendor
  attestations in lineage); `mod-110` (PII clause §4 defers to
  `mod-110`'s PII evaluation).
