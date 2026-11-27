# Exercise 05 — Certifications Portfolio Planner (Solution)

## 1. Solution overview

This is a **design + artifact** exercise. The learner authors a single
file — `certifications_portfolio.yaml` — plus a mapping table and an
18-month rollout plan that together constitute the certification
strategy for one small data-engineering team. The output is deliberately
**portfolio-level**, not a per-engineer career ladder. The question it
answers is "does the team, collectively, hold the credentials it needs
to be legally defensible and technically credible?" — not "how does
engineer X get promoted?" Promotion ladders are a people-manager
artifact in a different document. A certifications portfolio is a
**risk-management** artifact and belongs on the same shelf as the
dataset monitoring contract (mod-111 ex-01), the EU AI Act Annex XI
training-data summary (mod-112 ex-03), and the vendor-labelling RFP
(mod-107 ex-02). If the portfolio is vague, the "do we have anyone who
can answer the regulator's privacy question?" conversation happens in
email at 10pm the night before the audit.

Deliverables (per the exercise contract):

- `certifications_portfolio.yaml` — the pinned portfolio artifact:
  roster, cert catalogue, portfolio-level coverage targets, role-to-cert
  matrix, rollout plan quarter-by-quarter, budget block, renewal
  tracker, non-goals block, and responsibilities mapping.
- A "design rationale" subsection defending the hardest 3-4 decisions:
  why portfolio-level targets instead of per-engineer ladders, why IAPP
  AIGP is the center of gravity instead of a cloud-vendor AI cert, why
  ISO 42001 lead-implementer is included (and what is deliberately
  excluded), and why some certs sit at a bus factor of exactly one.
- A "mapping to responsibilities" table that ties each team
  responsibility from the mod-112 ex-01 charter to the certifications
  that primarily support it.

Governance framing. A certifications portfolio is a workforce-side
control, and three frameworks name workforce credentialing explicitly:

- **NIST AI Risk Management Framework — GOVERN function.** The GOVERN
  pillar of the NIST AI RMF is explicitly about "cultures, policies,
  processes, and practices" that make AI risk visible and owned, and
  its sub-categories call out AI literacy and workforce training as
  inputs. The certifications portfolio is the operational artifact that
  makes workforce literacy concrete: it names which credentials the
  team collectively holds, who holds each, and how renewal keeps the
  knowledge fresh. We cite the top-level GOVERN function rather than
  invent sub-category numbers; the framework URL is in §6.
- **OWASP Machine Learning Security Top 10.** The portfolio's most
  direct anchor is **ML06:2023 AI Supply Chain Attacks** — credentialed
  engineers are a mitigation for vendor-side risk because the people
  who read the labelling vendor's SOC 2 report and the cloud provider's
  shared-responsibility model need to understand what they are reading.
  A CIPM-credentialed engineer reviewing a labelling vendor's privacy
  controls is a different conversation from an un-credentialed engineer
  doing the same review, and auditors notice.
- **MITRE ATLAS.** ATLAS is a threat-knowledge base; its defender-side
  mitigations include workforce readiness. A portfolio that names the
  credentials covering adversarial-ML, data-poisoning, and model-theft
  response is a tangible defender-coverage artifact.

The EU AI Act angle is not optional. **Article 4 of the EU AI Act — "AI
literacy"** — obliges providers and deployers of AI systems to ensure a
sufficient level of AI literacy among their staff and contractors. The
obligation has applied since **2 February 2025**. A certifications
portfolio is not the only way to discharge Article 4 (internal training
programmes count too), but a portfolio that names credentials held and
in-progress is the simplest audit-defensible evidence a data-engineering
team can produce. The portfolio also carries the Article 4 argument
across to downstream model risk — the ML training team can point at the
SRDE portfolio when asked whether its upstream data pipeline is run by
people who understand their obligations.

Why this exercise deserves care. A portfolio drawn up in a hurry
over-indexes on cloud-vendor certifications (familiar, easy to budget)
and under-indexes on governance and privacy credentials (less familiar,
harder exams to source). The predictable outcome is a team excellent at
writing Spark jobs on its cloud and unable to answer a basic question
about GDPR-based deletion obligations when a regulator calls. The
portfolio exists so the EM and the VP of Engineering can see that bias
on one page and correct for it before it costs a finding. Deliverables
in priority order: coverage targets, role-to-cert matrix, rollout plan,
budget, renewal tracker.

## 2. Implementation — worked answer (a full certifications portfolio)

### 2.1 Chosen team

We stay on one concrete team for the entire portfolio, and we reuse the
team from mod-112 ex-01 so reviewers can cross-reference the charter
this portfolio supports.

- **Team name:** `search-ranking-data-engineering` (SRDE).
- **Parent org:** Search Platform, reporting to the VP of Engineering.
- **Headcount:** 6 engineers (1 tech lead, 2 senior, 2 mid, 1 junior) +
  1 engineering manager (not counted in the cert roster below — the EM
  holds no technical cert slot in this portfolio).
- **Mission snippet (from mod-112 ex-01 charter):** "owns the data
  pipelines, feature tables, and data-quality tripwires that make the
  consumer search re-ranker trainable, auditable, and legally
  defensible."
- **Why this team for a portfolio exercise:** it has a privacy surface
  (query-log PII), a vendor-labelling surface (the `cat_query_class`
  relabelling audit), an EU AI Act surface (feeds a high-risk model in
  the search product), and a cloud-platform surface (feature store,
  Airflow, object storage). That is the minimum cross-functional
  surface area where a certification portfolio actually earns its keep.

### 2.2 Project layout

```
exercise-05/
├── certifications_portfolio.yaml       # the pinned portfolio artifact
├── rationale.md                        # design-rationale note (§2.9)
├── mapping/
│   └── responsibilities_to_certs.md    # the responsibilities table (§2.8)
├── renewals/
│   └── renewal_tracker.csv             # exported from §2.7 for the EM
└── changelog.md                        # amendments, dated
```

### 2.3 The portfolio artifact

```yaml
# certifications_portfolio.yaml
# Portfolio-level certification plan for search-ranking-data-engineering
# (SRDE). One team, one plan, 18-month horizon. This document is
# versioned; promotions, hires, and departures update section 1
# (roster) and may update sections 5-7 but do not require a full
# re-approval unless a coverage target drops below floor.
portfolio_version: 1.0.0
portfolio_effective: 2026-12-01
portfolio_review_due: 2027-12-01   # annual review, re-sign at 2028-06 if plan extends
portfolio_sha256: null             # filled by the sign-off script
countersigned_by:
  - role: SRDE Engineering Manager
  - role: VP of Engineering
  - role: Legal / Privacy Lead                 # signs off on the privacy-cert slate
  - role: Learning & Development partner       # signs off on study-time budget

# -----------------------------------------------------------------------
# 1. Roster — roles, not humans
# -----------------------------------------------------------------------
# Headcount at portfolio sign. Promotions and hires update this block
# via the amendment process; the artifact never names an individual.
roster:
  - role_id: tech_lead
    level: L6-equivalent
    tenure_years_in_data_eng: 10
    current_certs: [AWS-SAA]              # pre-existing; see cert catalogue below
  - role_id: senior_a
    level: L5-equivalent
    tenure_years_in_data_eng: 7
    current_certs: [GCP-ACE]
  - role_id: senior_b
    level: L5-equivalent
    tenure_years_in_data_eng: 6
    current_certs: []
  - role_id: mid_a
    level: L4-equivalent
    tenure_years_in_data_eng: 4
    current_certs: []
  - role_id: mid_b
    level: L4-equivalent
    tenure_years_in_data_eng: 3
    current_certs: [AWS-CCP]
  - role_id: junior
    level: L3-equivalent
    tenure_years_in_data_eng: 1
    current_certs: []

# -----------------------------------------------------------------------
# 2. Portfolio-level coverage targets
# -----------------------------------------------------------------------
# For each cert category we declare the MINIMUM number of currently
# certified engineers the team needs to keep the bus factor above 1.
# "Bus factor" here is defined as: if the single most-credentialed
# engineer leaves tomorrow, can the team still answer an auditor,
# a regulator, or a production incident that requires this knowledge?
# If min_certified_holders == 1 we treat that explicitly as a risk,
# with a mitigation in section 9.
coverage_targets:
  - category: ai_governance
    min_certified_holders: 2
    rationale: Dataset governance + EU AI Act Article 4 coverage must survive one departure. Three preferred for a team of six.
  - category: privacy
    min_certified_holders: 2
    rationale: GDPR-adjacent work (query logs, DSAR re-emission) is routine. Bus factor of 1 on privacy is how a team misses a 30-day DSAR deadline during PTO season.
  - category: security_foundations
    min_certified_holders: 2
    rationale: Security+ or equivalent is the floor for anyone with write access to the Redis online mirror or the S3 feature-store bucket.
  - category: cloud_data_platform
    min_certified_holders: 3
    rationale: Day-job stack. Three out of six is operational, not aspirational.
  - category: ml_engineering
    min_certified_holders: 1
    rationale: Not an ML team; support one. One credentialed holder suffices to argue with the model team as peers. Bus-factor risk accepted (§9).
  - category: ai_management_system
    min_certified_holders: 1
    rationale: ISO 42001 lead-implementer is a specialist credential. Central governance holds the lead-auditor variant. Bus-factor risk accepted.
  - category: program_management
    min_certified_holders: 0
    rationale: We do not hold PMP-class certs on this team by policy — see §2.10. The EM's org has its own program function.

# -----------------------------------------------------------------------
# 3. Cert catalogue — the named credentials in scope
# -----------------------------------------------------------------------
# Every entry has: issuer, (optional) exam code if real and confidently
# known, prerequisites, renewal mechanism, approximate exam cost in USD
# (always flagged as approximate — exam fees change), and typical study
# hours for a mid-career engineer. Anything we are not confident about
# is marked research-note and left named rather than deleted.
cert_catalogue:

  # ---- 3.1 AI governance / responsible AI ----
  - cert_id: IAPP-AIGP
    display_name: IAPP Artificial Intelligence Governance Professional
    category: ai_governance
    issuer: International Association of Privacy Professionals (IAPP)
    exam_code: AIGP
    prerequisites: No formal prerequisite; body of knowledge assumes familiarity with AI/ML and privacy or risk frameworks.
    renewal_mechanism: CPE credits via IAPP membership
    renewal_cadence_months: 24   # <!-- research-note: confirm current IAPP maintenance cadence -->
    approx_cost_usd: 650         # <!-- research-note: verify at iapp.org/certify before each cohort -->
    study_hours_mid_career: 60
    notes: Center of gravity. Covers NIST AI RMF and ISO 42001 awareness, EU AI Act (incl. Article 4), risk management, AI lifecycle. Pairs with ISO 42001 lead-implementer.

  - cert_id: IEEE-CertifAIEd
    display_name: IEEE CertifAIEd (practitioner level)
    category: ai_governance
    issuer: IEEE Standards Association
    exam_code: null                        # <!-- research-note: confirm exact credential naming and levels -->
    prerequisites: Working familiarity with AI ethics assessment and IEEE 7000-series standards.
    renewal_mechanism: <!-- research-note: confirm renewal cadence for CertifAIEd -->
    renewal_cadence_months: null
    approx_cost_usd: null                  # <!-- research-note: current pricing from IEEE SA -->
    study_hours_mid_career: 40
    notes: Optional; included so the role-to-cert matrix can reference it. Not on the must-hold slate.

  # ---- 3.2 Privacy ----
  - cert_id: IAPP-CIPP-E
    display_name: IAPP Certified Information Privacy Professional / Europe
    category: privacy
    issuer: IAPP
    exam_code: CIPP/E
    prerequisites: None
    renewal_mechanism: CPE credits via IAPP membership
    renewal_cadence_months: 24             # <!-- research-note: confirm current IAPP CPE cycle length -->
    approx_cost_usd: 550                   # <!-- research-note: verify at iapp.org/certify -->
    study_hours_mid_career: 60
    notes: GDPR + e-Privacy Directive + EU data-protection law. The cert SRDE reaches for on query-log retention and DSAR re-emission SLAs.

  - cert_id: IAPP-CIPM
    display_name: IAPP Certified Information Privacy Manager
    category: privacy
    issuer: IAPP
    exam_code: CIPM
    prerequisites: None; pairs well with CIPP/E
    renewal_mechanism: CPE credits via IAPP membership
    renewal_cadence_months: 24             # <!-- research-note: confirm current IAPP CPE cycle length -->
    approx_cost_usd: 550                   # <!-- research-note: verify at iapp.org/certify -->
    study_hours_mid_career: 50
    notes: Program-management variant. Privacy-programme governance, incident response, vendor-management lifecycle. Directly applicable to mod-107 labelling-vendor RFP.

  - cert_id: IAPP-CIPT
    display_name: IAPP Certified Information Privacy Technologist
    category: privacy
    issuer: IAPP
    exam_code: CIPT
    prerequisites: None; technical-leaning privacy cert
    renewal_mechanism: CPE credits via IAPP membership
    renewal_cadence_months: 24             # <!-- research-note: confirm current IAPP CPE cycle length -->
    approx_cost_usd: 550                   # <!-- research-note: verify at iapp.org/certify -->
    study_hours_mid_career: 55
    notes: Engineering-focused. Pseudonymisation, k-anonymity, differential-privacy awareness, PETs. Pairs with mod-110 work.

  # ---- 3.3 Security foundations ----
  - cert_id: CompTIA-SecurityPlus
    display_name: CompTIA Security+
    category: security_foundations
    issuer: CompTIA
    exam_code: SY0-701                     # <!-- research-note: verify current exam code at comptia.org -->
    prerequisites: None
    renewal_mechanism: CEUs or re-exam
    renewal_cadence_months: 36
    approx_cost_usd: 400                   # <!-- research-note: verify at comptia.org -->
    study_hours_mid_career: 50
    notes: Floor-level security. Any engineer with prod S3 write permission should hold this.

  - cert_id: ISC2-CISSP
    display_name: (ISC)² Certified Information Systems Security Professional
    category: security_foundations
    issuer: (ISC)²
    exam_code: CISSP
    prerequisites: ~5 years cumulative paid experience in 2+ CISSP CBK domains (waivers possible). <!-- research-note: verify at isc2.org -->
    renewal_mechanism: CPE credits + annual maintenance fee
    renewal_cadence_months: 36
    approx_cost_usd: 750                                     # <!-- research-note: verify at isc2.org -->
    study_hours_mid_career: 150
    notes: Senior security generalist. Held by the security guild, not SRDE. Listed so §2.10 can declare non-pursuit explicitly.

  - cert_id: SANS-GIAC-data-security
    display_name: SANS GIAC data-security track (specific cert to be selected)
    category: security_foundations
    issuer: Global Information Assurance Certification (GIAC)
    exam_code: null                                          # <!-- research-note: pick specific GIAC (GCED / GSEC / GCDA) -->
    prerequisites: SANS course attendance strongly recommended but not mandatory
    renewal_mechanism: CPE credits
    renewal_cadence_months: 48
    approx_cost_usd: null                                    # <!-- research-note: GIAC+SANS combo varies widely; price before cohort -->
    study_hours_mid_career: 80
    notes: Specialist option. Not slated. Included for reference when the security guild asks.

  # ---- 3.4 Cloud / data platform ----
  - cert_id: AWS-Data-Engineer-Associate
    display_name: AWS Certified Data Engineer – Associate
    category: cloud_data_platform
    issuer: Amazon Web Services
    exam_code: DEA-C01                                       # <!-- research-note: verify active code at aws.amazon.com/certification -->
    prerequisites: AWS recommends 2-3 years data-engineering experience and 1-2 years hands-on AWS.
    renewal_mechanism: Re-exam (AWS associate certs do not use CEUs)
    renewal_cadence_months: 36
    approx_cost_usd: 150                                     # <!-- research-note: verify at aws.amazon.com/certification -->
    study_hours_mid_career: 60
    notes: Primary cloud cert for the day job. Stack runs on AWS; baseline.

  - cert_id: GCP-Professional-Data-Engineer
    display_name: Google Cloud Professional Data Engineer
    category: cloud_data_platform
    issuer: Google Cloud
    exam_code: null                                          # Google does not use a short exam code the same way AWS does
    prerequisites: Recommended 3+ years industry experience including 1+ year of GCP solutions.
    renewal_mechanism: Re-exam (Google Cloud recertifies by re-taking)
    renewal_cadence_months: 24                               # <!-- research-note: cadence has changed; verify at cloud.google.com/learn/certification -->
    approx_cost_usd: 200                                     # <!-- research-note: verify at cloud.google.com/learn/certification -->
    study_hours_mid_career: 80
    notes: Second-cloud cert. One holder so a migration or GCP vendor audit is not cold-started.

  - cert_id: Azure-Data-Engineer-Associate
    display_name: Microsoft Certified — Azure Data Engineer Associate
    category: cloud_data_platform
    issuer: Microsoft
    exam_code: null                                          # <!-- research-note: DP-203 retired; Fabric Data Engineer Associate (DP-700) is the current successor at time of writing — verify at learn.microsoft.com/credentials -->
    prerequisites: Azure fundamentals (AZ-900) strongly recommended
    renewal_mechanism: Annual free online renewal assessment
    renewal_cadence_months: 12
    approx_cost_usd: 165                                     # <!-- research-note: verify at learn.microsoft.com/credentials -->
    study_hours_mid_career: 80
    notes: Not slated — not an Azure shop. Listed for parent-org M&A scenarios.

  # ---- 3.5 ML-specific ----
  - cert_id: AWS-ML-Specialty
    display_name: AWS Certified Machine Learning – Specialty
    category: ml_engineering
    issuer: Amazon Web Services
    exam_code: MLS-C01                                       # <!-- research-note: AWS announced a successor "AI/ML Engineer – Associate" track; verify current recommended cert at aws.amazon.com/certification -->
    prerequisites: Recommended 2+ years developing, architecting, or running ML workloads on AWS.
    renewal_mechanism: Re-exam
    renewal_cadence_months: 36
    approx_cost_usd: 300                                     # <!-- research-note: verify at aws.amazon.com/certification -->
    study_hours_mid_career: 100
    notes: Held so one engineer can argue with the ML-training team as peers. Not a core SRDE skill but a core SRDE *responsibility*.

  - cert_id: GCP-ML-Engineer
    display_name: Google Cloud Professional Machine Learning Engineer
    category: ml_engineering
    issuer: Google Cloud
    exam_code: null
    prerequisites: Recommended 3+ years of industry experience including 1+ year of GCP ML solutions.
    renewal_mechanism: Re-exam
    renewal_cadence_months: 24                               # <!-- research-note: verify at cloud.google.com/learn/certification -->
    approx_cost_usd: 200                                     # <!-- research-note: verify at cloud.google.com/learn/certification -->
    study_hours_mid_career: 100
    notes: Optional alternative to AWS-ML-Specialty. Listed, not slated.

  - cert_id: Databricks-ML-Engineer
    display_name: Databricks Certified Machine Learning Engineer (Associate / Professional)
    category: ml_engineering
    issuer: Databricks
    exam_code: null                                          # <!-- research-note: confirm current level naming at databricks.com/learn/certification -->
    prerequisites: Hands-on Databricks + Spark ML experience
    renewal_mechanism: Re-exam
    renewal_cadence_months: 24                               # <!-- research-note: verify at databricks.com/learn/certification -->
    approx_cost_usd: 200                                     # <!-- research-note: verify at databricks.com/learn/certification -->
    study_hours_mid_career: 80
    notes: Optional; relevant if feature-build DAG moves to Databricks. Not slated in this cycle.

  # ---- 3.6 AI management system / quality / audit ----
  - cert_id: ISO-42001-Lead-Implementer
    display_name: ISO/IEC 42001:2023 AI Management System — Lead Implementer
    category: ai_management_system
    issuer: Multiple accredited training bodies (e.g. PECB, BSI, Exemplar Global)
    exam_code: null
    prerequisites: Familiarity with management-system standards (ISO 27001 awareness helpful, not required).
    renewal_mechanism: CPD credits via the certifying body
    renewal_cadence_months: 36                               # <!-- research-note: cadence varies by certifying body; confirm with chosen training provider -->
    approx_cost_usd: 1500                                    # <!-- research-note: training-provider dependent; range observed 1000-2500 USD -->
    study_hours_mid_career: 50
    notes: ISO/IEC 42001:2023 is the AI management-system standard. Lead-implementer fits SRDE's "we help design the AIMS" posture. Lead-auditor variant is held by central governance (see non-goals).

  # ---- 3.7 Program management (reference only) ----
  - cert_id: PMI-PMP
    display_name: Project Management Professional
    category: program_management
    issuer: Project Management Institute
    exam_code: PMP
    prerequisites: 35 hours of PM education + documented project-leading experience. <!-- research-note: verify at pmi.org -->
    renewal_mechanism: PDUs (Professional Development Units)
    renewal_cadence_months: 36
    approx_cost_usd: 575                                     # <!-- research-note: PMI member vs non-member rates differ; verify at pmi.org -->
    study_hours_mid_career: 150
    notes: Listed for completeness; not slated. SRDE declines this family — see §2.10.
```

### 2.4 Role-to-cert matrix

Legend: **M** = must-hold (role is not fully onboarded without this),
**S** = should-hold (team expects this within 18 months in role),
**O** = optional (encouraged, team pays if requested), **—** = out of
scope for this role.

| Cert ID                     | tech_lead | senior_a | senior_b | mid_a | mid_b | junior |
|-----------------------------|-----------|----------|----------|-------|-------|--------|
| IAPP-AIGP                   | **M**     | **M**    | **S**    | **S** | O     | O      |
| IEEE-CertifAIEd             | O         | O        | O        | O     | O     | O      |
| IAPP-CIPP-E                 | **S**     | **M**    | **S**    | O     | O     | O      |
| IAPP-CIPM                   | **M**     | O        | O        | O     | O     | —      |
| IAPP-CIPT                   | **S**     | **S**    | **M**    | **S** | **S** | O      |
| CompTIA-SecurityPlus        | **S**     | **M**    | **M**    | **M** | **M** | **M**  |
| ISC2-CISSP                  | —         | O        | O        | —     | —     | —      |
| SANS-GIAC-data-security     | O         | O        | O        | —     | —     | —      |
| AWS-Data-Engineer-Associate | **M**     | **M**    | **M**    | **M** | **M** | **S**  |
| GCP-Professional-Data-Eng.  | O         | **S**    | O        | O     | O     | —      |
| Azure-Data-Eng.-Associate   | O         | O        | O        | O     | O     | —      |
| AWS-ML-Specialty            | O         | **S**    | O        | O     | O     | —      |
| GCP-ML-Engineer             | O         | O        | O        | O     | O     | —      |
| Databricks-ML-Engineer      | O         | O        | O        | O     | O     | —      |
| ISO-42001-Lead-Implementer  | **M**     | O        | O        | O     | O     | —      |
| PMI-PMP                     | —         | —        | —        | —     | —     | —      |

Reading rules:

- A role's **M** set is the floor. Someone hired into senior_b who does
  not yet hold CIPT has 12 months to earn it; the EM funds it and
  schedules study time (see §2.6).
- An **S** entry is tracked in the rollout plan — if a role has an S
  and not an M for the same category, the S must turn into an M within
  18 months in role or the EM flags the role for a career-conversation.
- An **O** entry is funded on request as long as budget allows. The EM
  may deny O requests that do not map to a team responsibility.
- A **—** entry means the team actively does *not* want the role to
  spend budget here. The junior does not pursue PMP. The tech lead does
  not pursue CISSP; that is the security guild's lane.

### 2.5 18-month rollout plan (quarter by quarter)

The plan starts the quarter after sign-off. Study hours are per role
per week during the prep quarter; exam attempts are paid from the exam
fee budget.

| Quarter    | Dates              | Role        | Cert in prep               | Study hrs/wk | Target exam | Exam cost (USD, approx) |
|------------|--------------------|-------------|----------------------------|--------------|-------------|--------------------------|
| **Q1 2027**| 2027-01 → 2027-03  | tech_lead   | IAPP-AIGP                  | 5            | 2027-03     | 650                      |
|            |                    | senior_a    | AWS-Data-Engineer-Assoc.   | 5            | 2027-03     | 150                      |
|            |                    | junior      | CompTIA-SecurityPlus       | 4            | 2027-03     | 400                      |
| **Q2 2027**| 2027-04 → 2027-06  | tech_lead   | ISO-42001-Lead-Implementer | 4            | 2027-06     | 1500                     |
|            |                    | senior_a    | IAPP-AIGP                  | 5            | 2027-06     | 650                      |
|            |                    | mid_b       | AWS-Data-Engineer-Assoc.   | 5            | 2027-06     | 150                      |
| **Q3 2027**| 2027-07 → 2027-09  | tech_lead   | IAPP-CIPM                  | 4            | 2027-09     | 550                      |
|            |                    | senior_b    | IAPP-CIPT                  | 5            | 2027-09     | 550                      |
|            |                    | mid_a       | AWS-Data-Engineer-Assoc.   | 5            | 2027-09     | 150                      |
| **Q4 2027**| 2027-10 → 2027-12  | senior_a    | IAPP-CIPP-E                | 5            | 2027-12     | 550                      |
|            |                    | mid_a       | CompTIA-SecurityPlus       | 4            | 2027-12     | 400                      |
|            |                    | mid_b       | CompTIA-SecurityPlus       | 4            | 2027-12     | 400                      |
| **Q1 2028**| 2028-01 → 2028-03  | senior_b    | CompTIA-SecurityPlus       | 4            | 2028-03     | 400                      |
|            |                    | senior_a    | AWS-ML-Specialty           | 6            | 2028-03     | 300                      |
|            |                    | mid_a       | IAPP-CIPT                  | 5            | 2028-03     | 550                      |
| **Q2 2028**| 2028-04 → 2028-06  | mid_b       | IAPP-CIPT                  | 5            | 2028-06     | 550                      |
|            |                    | senior_b    | IAPP-AIGP                  | 5            | 2028-06     | 650                      |
|            |                    | junior      | AWS-Data-Engineer-Assoc.   | 5            | 2028-06     | 150                      |

End-of-plan state (projected, 2028-06-30):

- AI governance: 3 AIGP holders (tech_lead, senior_a, senior_b) + 1
  ISO 42001 lead-implementer — **target met (3 ≥ 2), ISO 42001 bus
  factor 1 (accepted risk).**
- Privacy: 2 CIPP/E (tech_lead if carried from §2.4 S, senior_a),
  3 CIPT (senior_b, mid_a, mid_b), 1 CIPM (tech_lead) — **targets met**,
  CIPM bus factor 1 (accepted risk; the external privacy counsel the
  team retains is a mitigation).
- Security foundations: 5 Security+ holders (all except tech_lead who
  holds adjacent AWS/Azure security experience) — **target met (5 ≥ 2).**
- Cloud data platform: 5 AWS Data Engineer holders by end of plan —
  **target met (5 ≥ 3).**
- ML engineering: 1 AWS ML Specialty holder (senior_a) — **target met
  (1 ≥ 1); bus factor 1, accepted risk, mitigated by the ML-training
  team being the primary owner of that knowledge.**
- AI management system: 1 ISO 42001 lead-implementer — **target met**;
  bus factor 1.

### 2.6 Budget block

Budget is stated in approximate USD. All cost figures are approximate
and subject to issuer-side price changes; the EM revisits the budget
at each quarterly review.

```yaml
budget:
  currency: USD
  horizon_quarters: 6                   # Q1 2027 through Q2 2028
  per_quarter_breakdown:
    - quarter: Q1-2027
      exam_fees_approx: 1200            # 650 + 150 + 400 — see §2.5
      prep_materials_approx: 450        # books, practice exams, IAPP prep courses
      conference_ceu_approx: 0          # no renewals this quarter
      study_time_hours_total: 182       # 3 roles × ~14 weeks × avg 4.3 hrs
    - quarter: Q2-2027
      exam_fees_approx: 2300            # 1500 + 650 + 150
      prep_materials_approx: 600
      conference_ceu_approx: 0
      study_time_hours_total: 182
    - quarter: Q3-2027
      exam_fees_approx: 1250            # 550 + 550 + 150
      prep_materials_approx: 450
      conference_ceu_approx: 1500       # IAPP Global Privacy Summit attendance, 1 seat
      study_time_hours_total: 182
    - quarter: Q4-2027
      exam_fees_approx: 1350            # 550 + 400 + 400
      prep_materials_approx: 450
      conference_ceu_approx: 0
      study_time_hours_total: 169
    - quarter: Q1-2028
      exam_fees_approx: 1250            # 400 + 300 + 550
      prep_materials_approx: 500
      conference_ceu_approx: 2000       # ISO 42001 CPD course for the holder
      study_time_hours_total: 195
    - quarter: Q2-2028
      exam_fees_approx: 1350            # 550 + 650 + 150
      prep_materials_approx: 500
      conference_ceu_approx: 1500       # IAPP summit, 1 seat
      study_time_hours_total: 195
  totals_approx:
    exam_fees: 8700
    prep_materials: 2950
    conference_ceu: 5000
    grand_total_cash: 16650
    grand_total_study_hours: 1105
  notes:
    - All exam-fee figures carry the approx-flag from §2.3; re-check
      at each quarter start.
    - Study-time hours are a resource ask on the EM's quarterly plan,
      not a cash number; we show them so sprint planning absorbs the
      capacity cost.
    - The IAPP summit attendance replaces per-head CEU subscriptions
      for one engineer per year; others renew via free IAPP member
      CPE webinars.
```

### 2.7 Renewal tracker

> Dates below are illustrative placeholders. The holder-role column
> intentionally references roles; the earned-date and expires-date are
> the lines a holder updates when they pass the exam. `next_action_due`
> is set to 90 days before expiry so renewal work is scheduled, not
> reactive.

| Cert                        | Holder role | Earned (placeholder) | Expires (placeholder) | Renewal mechanism                 | Next action due |
|-----------------------------|-------------|----------------------|-----------------------|-----------------------------------|-----------------|
| IAPP-AIGP                   | tech_lead   | 2027-03-15           | 2029-03-15            | CPEs via IAPP                     | 2028-12-15      |
| IAPP-AIGP                   | senior_a    | 2027-06-20           | 2029-06-20            | CPEs via IAPP                     | 2029-03-20      |
| IAPP-AIGP                   | senior_b    | 2028-06-10           | 2030-06-10            | CPEs via IAPP                     | 2030-03-10      |
| IAPP-CIPP-E                 | senior_a    | 2027-12-05           | 2029-12-05            | CPEs via IAPP                     | 2029-09-05      |
| IAPP-CIPM                   | tech_lead   | 2027-09-20           | 2029-09-20            | CPEs via IAPP                     | 2029-06-20      |
| IAPP-CIPT                   | senior_b    | 2027-09-25           | 2029-09-25            | CPEs via IAPP                     | 2029-06-25      |
| IAPP-CIPT                   | mid_a       | 2028-03-18           | 2030-03-18            | CPEs via IAPP                     | 2029-12-18      |
| IAPP-CIPT                   | mid_b       | 2028-06-14           | 2030-06-14            | CPEs via IAPP                     | 2030-03-14      |
| CompTIA-SecurityPlus        | junior      | 2027-03-20           | 2030-03-20            | CEUs or re-exam                   | 2029-12-20      |
| CompTIA-SecurityPlus        | mid_a       | 2027-12-12           | 2030-12-12            | CEUs or re-exam                   | 2030-09-12      |
| CompTIA-SecurityPlus        | mid_b       | 2027-12-12           | 2030-12-12            | CEUs or re-exam                   | 2030-09-12      |
| CompTIA-SecurityPlus        | senior_b    | 2028-03-15           | 2031-03-15            | CEUs or re-exam                   | 2030-12-15      |
| AWS-Data-Engineer-Associate | senior_a    | 2027-03-25           | 2030-03-25            | Re-exam                           | 2029-12-25      |
| AWS-Data-Engineer-Associate | mid_b       | 2027-06-28           | 2030-06-28            | Re-exam                           | 2030-03-28      |
| AWS-Data-Engineer-Associate | mid_a       | 2027-09-30           | 2030-09-30            | Re-exam                           | 2030-06-30      |
| AWS-Data-Engineer-Associate | junior      | 2028-06-29           | 2031-06-29            | Re-exam                           | 2031-03-29      |
| AWS-ML-Specialty            | senior_a    | 2028-03-30           | 2031-03-30            | Re-exam                           | 2030-12-30      |
| ISO-42001-Lead-Implementer  | tech_lead   | 2027-06-30           | 2030-06-30            | CPD credits via certifying body   | 2030-03-30      |

### 2.8 Mapping to responsibilities

Every responsibility in the mod-112 ex-01 SRDE charter is paired with
one or more certifications that the portfolio commits to covering.
This is the table the EM shows the VP of Engineering when asked "are
we credentialed to do the work we said we would do?"

| Responsibility (from SRDE charter)                                            | Primary certs                                          | Supporting certs                      |
|-------------------------------------------------------------------------------|--------------------------------------------------------|---------------------------------------|
| Own `features.search_reranker_v3` and its producer pipeline                   | AWS-Data-Engineer-Associate                            | GCP-Professional-Data-Engineer (opt.) |
| Dataset governance / dataset card (mod-112 ex-02)                             | IAPP-AIGP, ISO-42001-Lead-Implementer                  | IAPP-CIPT                             |
| EU AI Act Annex XI training-data summary (mod-112 ex-03)                      | IAPP-AIGP, ISO-42001-Lead-Implementer                  | IAPP-CIPM                             |
| Model-card data-slice inputs (mod-112 ex-04)                                  | IAPP-AIGP                                              | AWS-ML-Specialty                      |
| Monitoring contract SLOs + on-call (mod-111 ex-01)                            | AWS-Data-Engineer-Associate, CompTIA-SecurityPlus      | ISO-42001-Lead-Implementer            |
| Query-log PII handling and DSAR re-emission (mod-110 ex-03)                   | IAPP-CIPP-E, IAPP-CIPT                                 | IAPP-CIPM                             |
| Vendor-labelling RFP and ongoing oversight (mod-107 ex-02)                    | IAPP-CIPM, IAPP-CIPT                                   | IAPP-AIGP (vendor AI governance)      |
| Shared categorical vocabulary management (`cat_vertical`)                     | AWS-Data-Engineer-Associate                            | —                                     |
| Benchmark de-contamination check (mod-106)                                    | AWS-ML-Specialty                                       | IAPP-AIGP                             |
| Synthetic-data provenance tagging (mod-108 territory)                         | IAPP-AIGP                                              | ISO-42001-Lead-Implementer            |
| Production S3 bucket + Redis mirror security hygiene                          | CompTIA-SecurityPlus                                   | AWS-Data-Engineer-Associate           |
| EU AI Act Article 4 literacy obligation (team-wide, 2 Feb 2025 effective)     | IAPP-AIGP (at least one per sub-team)                  | all privacy certs reinforce           |

Every row in this table must have at least one `Primary` cert in
"must-hold" state in the role-to-cert matrix (§2.4). If a cell is
blank in Primary, the responsibility is uncovered and is a §9 risk.

### 2.9 Design rationale

The portfolio makes four decisions a reviewer is likely to push back
on. The rationale block defends each one.

**Decision 1 — Portfolio-level targets, not per-engineer ladders.**
Per-engineer ladders are a career-development artifact; they answer
"what does engineer X do to earn promotion?" The portfolio answers a
different question: "does the team have the credentials it needs to
be legally defensible and technically credible, independent of who
holds what?" The two artifacts have different owners (people-manager
owns ladders; EM + VP + Legal co-own the portfolio) and different
failure modes (a ladder fails on misaligned promotion criteria; a
portfolio fails when coverage breaks on a departure). Blending them
yields a document weak at both jobs. The §2.4 matrix is the interface
— it tells a future ladder author which certs count for which role —
but the ladder itself is out of scope here.

**Decision 2 — IAPP AIGP is the center of gravity, not a cloud-vendor
AI cert.** Cloud-vendor AI certs (AWS AI Practitioner, AWS ML
Specialty, GCP ML Engineer, Azure AI Engineer) test the mechanics of
a vendor platform. Useful for delivery; not what an auditor, regulator,
or VP needs the team to know. AIGP tests AI-governance concepts — NIST
AI RMF, ISO 42001 awareness, EU AI Act (incl. Article 4), risk
management, lifecycle, bias and fairness vocabulary — which is what
the SRDE charter obligates the team to apply. AWS ML Specialty is
support, not center. If the team can earn exactly one cert in year one,
AIGP is the one.

**Decision 3 — ISO 42001 lead-implementer in, ISO 27001 lead-auditor
out.** ISO/IEC 42001:2023 is the AI management-system standard. The
lead-implementer variant fits SRDE's posture: we help design and run
the AIMS for our slice, we do not *audit* it (that is the central
governance function's job and requires independence from the function
being audited). ISO 27001 is a general information-security management
standard; SRDE is a consumer of the company's ISMS, not a contributor,
so we delegate 27001 coverage to the security guild. Lead-auditor of
either standard is a conflict of interest if held by the audited team.
"Hold both?" — no: budget and bus-factor. Six engineers cannot sustain
renewals across both MS families.

**Decision 4 — Some certs sit at a bus factor of exactly 1.** IAPP
CIPM (tech_lead only), ISO 42001 lead-implementer (tech_lead only),
and AWS ML Specialty (senior_a only) are each held by one role. We
chose bus factor 1 on these three specifically because:

- CIPM is a program-management privacy cert; the external privacy
  counsel the company retains is the mitigation when the holder is
  unavailable.
- ISO 42001 lead-implementer is a specialist credential whose
  day-to-day use is quarterly at most; the central governance
  function's lead-auditor is the mitigation.
- AWS ML Specialty is a *peer-with-the-ML-team* credential, not an
  SRDE-core one; the ML training team holds the actual ML expertise
  and is the mitigation.

All three are flagged in §9 (coverage risks accepted) and reviewed at
the annual portfolio review. If the team grows to eight engineers, the
second holder of CIPM is the next hire's onboarding cert; AIGP remains
the first.

### 2.10 Non-goals — certs we are deliberately NOT pursuing

We publish non-goals so the portfolio cannot be read as implying
anything it does not. SRDE deliberately does not slate:

- **SOC 2 auditor / CPA-track certifications.** SOC 2 attestation
  work belongs to a different team (Compliance / IT-GRC). If SRDE
  needs SOC 2 input, it buys the service from that team or from an
  external auditor; it does not hold the credential itself. Holding it
  in-team would be a conflict of interest.
- **CISA (ISACA Certified Information Systems Auditor).** Same
  reasoning. The audit function is separate from the function being
  audited.
- **(ISC)² CISSP for all engineers.** CISSP is a senior security
  generalist credential; SRDE's security needs are met by Security+
  at breadth and by the security guild at depth. Over-credentialing
  SRDE on CISSP displaces budget from AI-governance and privacy.
- **PMP / PMI-ACP / SAFe POPM.** Program-management credentials are
  not an SRDE lane. The EM carries that weight. Over-credentialing
  the team here produces process-heavy IC-time, which is the opposite
  of what the charter commits to.
- **TensorFlow / PyTorch vendor-educational certs.** Framework-specific
  certifications are weak evidence of ML-governance competence and
  rotate out of currency quickly. We prefer AWS/GCP ML certs if we
  slate any ML cert at all, and we slate the AIGP as a stronger
  general-AI credential.
- **"AI ethics" certificates from unaccredited issuers.** Any claim
  of AI-ethics competency that is not backed by an accredited body
  (IAPP, ISO, IEEE) does not count toward portfolio coverage. We will
  pay for employees to *attend* the course if they ask, but we will
  not record the completion as a portfolio-level cert.

### 2.11 Coverage risks accepted (section 9 of the artifact)

```yaml
accepted_coverage_risks:
  - risk_id: BUS-FACTOR-CIPM
    description: IAPP-CIPM held by one role (tech_lead). On departure/unavailability, privacy-program coverage falls to the external counsel retainer until a successor onboards.
    mitigation: external_counsel_retainer
    review_date: 2027-12-01
  - risk_id: BUS-FACTOR-ISO42001
    description: ISO-42001-Lead-Implementer held by one role (tech_lead). Central governance's lead-auditor is the interim coverage source until SRDE re-credentials a successor.
    mitigation: central_governance_lead_auditor
    review_date: 2027-12-01
  - risk_id: BUS-FACTOR-AWS-ML
    description: AWS-ML-Specialty held by one role (senior_a). ML-training team is the primary owner of ML expertise; SRDE's holder is a peer-to-ML credential, not primary ownership.
    mitigation: ml_training_team_primary_ownership
    review_date: 2027-12-01
  - risk_id: NO-SECOND-CLOUD
    description: One GCP-Professional-Data-Engineer holder on team. A regulated GCP migration or GCP-vendor audit would be cold-start.
    mitigation: Parent org commits: any cloud migration triggers a 90-day re-planning cycle; SRDE slates a second GCP cert in that cycle.
    review_date: 2027-12-01
```

## 3. Validation steps

A reviewer can mechanically check the portfolio. Not all checks are
automatable; where a check requires human judgment, that is called
out.

1. **YAML lints clean.** `yamllint certifications_portfolio.yaml`
   returns zero errors and zero warnings on default rules (adjust
   line-length to project convention). No tab characters, consistent
   two-space indent, no trailing whitespace.
2. **Schema fields are present on every cert.** For each entry in
   `cert_catalogue` the reviewer confirms the presence (not
   necessarily the value) of: `cert_id`, `display_name`, `category`,
   `issuer`, `prerequisites`, `renewal_mechanism`,
   `renewal_cadence_months`, `approx_cost_usd`, `study_hours_mid_career`.
   Any slot left unfilled must carry an inline
   `<!-- research-note: ... -->` comment that names exactly what is
   uncertain. A null value with no research-note comment is a reject.
3. **Issuer strings are plausible.** Each `issuer` resolves to an
   organisation whose URL is in §6 or is otherwise well-known. This is
   a human check; the aim is to catch invented issuers.
4. **Cost claims are approximate-flagged.** Every numeric
   `approx_cost_usd` is called out as approximate in the containing
   notes or via a `research-note` comment. No cost claim appears as
   if authoritative.
5. **Roster contains roles, not humans.** `grep -i` the artifact for
   first-name-like tokens (common first names, @-handles, email
   addresses) and confirm none appear. Every entry is a role_id like
   `tech_lead`, `senior_a`, `mid_b`, `junior`.
6. **Role-to-cert matrix uses the same role_ids as the roster.** The
   column headers of §2.4 match the `role_id` values in the roster
   block exactly. No row refers to a role that is not in the roster,
   and no roster role is missing from the matrix.
7. **Coverage targets are met or risk-flagged.** For each category in
   `coverage_targets`, count the roles in §2.4 with an **M** or **S**
   in that category (or count projected end-of-plan holders from §2.5)
   and confirm the count is ≥ `min_certified_holders`. Any category
   with count == 1 must appear in §2.11 `accepted_coverage_risks`
   with a stated mitigation.
8. **Budget sums match declared totals.** Sum `exam_fees_approx`,
   `prep_materials_approx`, and `conference_ceu_approx` across all
   quarters and confirm each sum equals the declared total in
   `totals_approx`. A mismatch is a reject (arithmetic errors in a
   budget are an audit smell).
9. **Study-hours allocation is realistic.** For each role listed in
   §2.5, confirm no single quarter asks the role to study for more
   than ~8 hrs/week sustained. If any row exceeds that, the row needs
   a note on how the EM protects capacity or the row is reject.
10. **Every responsibility has at least one primary cert.** For each
    row of §2.8 (mapping to responsibilities), confirm the `Primary
    certs` column is non-empty and that each named cert appears in the
    role-to-cert matrix as **M** for at least one role. If a Primary
    cert is optional or absent in §2.4, the responsibility is
    uncovered and must be a §2.11 risk.
11. **Renewal-date math is consistent.** For each row of §2.7, confirm
    `expires = earned + renewal_cadence_months`, and
    `next_action_due = expires − 90 days`. Off-by-one in the renewal
    column has historically been the way teams miss renewals.
12. **EU AI Act Article 4 is explicitly addressed.** `grep` the
    artifact for the string "Article 4" and confirm it appears at
    least once, with a named cert slated as its primary coverage
    (AIGP in this portfolio). Absence is a reject.
13. **Non-goals block is present and specific.** §2.10 names at least
    three cert families that are deliberately not pursued and gives a
    reason for each. A portfolio with no non-goals is almost always
    one where scope creep has not been contemplated.
14. **Changelog exists.** `changelog.md` exists and is non-empty; at
    minimum the 1.0.0 entry is recorded with the sign-off date.

## 4. Rubric or review checklist

Scores are 0, 1, or 2 per row. Pass threshold: ≥ 32/40 total AND no
`critical` row scores 0. A critical-row 0 fails the whole review
regardless of the total.

| # | Dimension                                                                                              | 0 (fail)                                                | 1 (partial)                                                | 2 (pass)                                                                                | Critical |
|---|--------------------------------------------------------------------------------------------------------|---------------------------------------------------------|------------------------------------------------------------|-----------------------------------------------------------------------------------------|----------|
| 1 | Artifact exists and is YAML-parsable                                                                   | Missing or invalid YAML                                 | Parses with warnings                                        | Clean parse under `yamllint`                                                             | **yes**  |
| 2 | Roster uses roles, not humans (no names, no emails, no @-handles)                                      | Any human name appears                                  | Some roles, some initials                                   | Fully role-based; passes grep sweep                                                      | **yes**  |
| 3 | Every cert carries issuer + renewal cadence + cost field (or research-note comment)                   | Slots missing with no research-note tag                | Some slots unmarked                                         | Every slot either filled or `research-note`-tagged                                      | **yes**  |
| 4 | Every cost or renewal claim is either sourced or marked approximate / research-note                   | Unsourced authoritative-looking numbers                 | Some marked, some bare                                      | All numeric claims carry approx-flag or research-note                                   | **yes**  |
| 5 | Portfolio-level coverage targets stated for each category, with a bus-factor rationale                 | No targets                                              | Targets without rationale                                  | Targets + explicit bus-factor reasoning                                                 | yes      |
| 6 | Role-to-cert matrix uses M/S/O/— semantics and covers every role × every cert                          | Matrix missing                                          | Partial matrix                                             | Full matrix, legend stated, every cell filled                                            | yes      |
| 7 | 18-month rollout plan quarter-by-quarter with role-cert-quarter triples                                | No rollout plan                                         | Plan but missing quarters or roles                         | Full plan with study hours, exam dates, and costs                                        | yes      |
| 8 | Budget block sums match declared totals, broken down by exam fee / prep / conference-CEU               | No budget                                               | Budget without breakdown                                   | Full breakdown + totals; arithmetic consistent                                           | **yes**  |
| 9 | Renewal tracker exists and renewal-date math is consistent                                             | No tracker                                              | Tracker with inconsistent dates                            | Tracker with `expires = earned + cadence`, `next_action_due = expires − 90d`             | yes      |
| 10 | Mapping to responsibilities ties each charter responsibility to ≥1 cert                                | Mapping missing                                         | Some responsibilities unmapped                             | Every responsibility has ≥1 Primary cert, named                                          | yes      |
| 11 | Non-goals block names cert families deliberately excluded                                              | No non-goals                                            | Non-goals without reasons                                  | Non-goals with reasons and (where relevant) owner delegated                              | yes      |
| 12 | Design rationale defends ≥3 hard decisions (portfolio-vs-ladder, AIGP center, bus-factor-1s)           | No rationale                                            | Rationale touches 1-2 decisions                            | Rationale defends ≥3 decisions explicitly                                                | yes      |
| 13 | EU AI Act Article 4 ("AI literacy", in force 2 Feb 2025) is explicitly addressed                       | Not mentioned                                           | Mentioned but no cert mapped                               | Named cert slated as primary coverage for Article 4                                      | **yes**  |
| 14 | NIST AI RMF GOVERN function is cited (top-level function, no invented sub-category numbers)            | Not mentioned                                           | Mentioned with fabricated sub-category IDs                 | Cited at function level with URL in §6                                                   | yes      |
| 15 | OWASP ML Top 10 and MITRE ATLAS referenced as workforce-side controls                                  | Neither mentioned                                       | One mentioned without connection                           | Both mentioned, each with a named control connection                                     | yes      |
| 16 | Coverage risks accepted (bus-factor-1 cases) are explicitly listed with mitigations                    | Hidden or not declared                                  | Listed without mitigations                                 | Each risk has a named mitigation and a review date                                       | yes      |
| 17 | Portfolio versioned + reviewed date + review-due date set                                              | No version metadata                                     | Partial                                                    | `portfolio_version`, `effective`, `review_due`, and sign-off roles all present           | yes      |
| 18 | Cross-references to sibling mod-112 and adjacent mod exercises (ex-01 charter, mod-107, mod-110, etc.) | None                                                   | Some                                                        | Charter + mod-107 + mod-110 + mod-111 all referenced with specifics                      | no       |
| 19 | Prose quality — artifact readable by a non-engineer VP in one sitting                                  | Dense jargon with no glossary                           | Readable with effort                                       | Readable in 30 minutes, with tables and plain language                                   | no       |
| 20 | Markdown lints clean (headings monotonically increasing, no broken tables)                             | Broken tables / heading skips                            | Minor lint issues                                           | Clean under `markdownlint` default rules                                                  | no       |

Total possible: 40. Pass: ≥ 32 AND every critical row scores 2.

## 5. Common mistakes

1. **Treating the portfolio as a career ladder.** Learners default to
   "senior = CISSP, mid = Security+, junior = CCP." That is a
   promotion ladder. A portfolio is about team-level coverage, not
   individual progression. Why this bites: a departure breaks
   coverage you did not know you had.
2. **Inventing certifications or exam codes.** "AWS Certified Data
   Scientist", "Google Cloud AI Ethics Specialist", "IEEE AI-Ops
   Certified" — none of those exist, or they existed briefly and were
   renamed. Why this bites: anyone reviewing the artifact who holds a
   real cert from the issuer will notice immediately and lose
   confidence in everything else.
3. **Quoting exam fees as if authoritative.** Exam fees change every
   12-18 months. An artifact that cites "$300" without a flag reads as
   out of date the week after it is signed. Why this bites: renewal
   budgets that assume today's price break when the issuer raises.
4. **Omitting the renewal mechanism.** A cert without renewal
   information is a one-shot credential that silently expires 24-36
   months later. Why this bites: the first expiry is usually the one
   that catches an auditor's question.
5. **Everyone holds everything.** Treating the matrix as "we would
   love everyone to hold everything so let's mark it all must" is how
   a team burns 400 hours of IC time per quarter on exam prep and
   nothing ships. Why this bites: study-hours shown in §2.6 become
   fiction, and the team's charter SLAs slip.
6. **No bus-factor accounting.** Every cert held by exactly one role,
   or every cert held by the tech lead. Why this bites: you have
   written a portfolio that collapses on first departure.
7. **Over-credentialing in cloud, under-credentialing in governance.**
   Four AWS certs, no AIGP, no CIPP/E. Why this bites: the auditor
   question is never "can you resize an EMR cluster?"; it is "who on
   your team can defend Article 4 compliance in a meeting with
   Legal?"
8. **Missing the EU AI Act Article 4 angle.** Treating the Act as a
   downstream model-team concern and not a data-team concern. Why
   this bites: Article 4 applies to providers *and deployers*, and
   the data-engineering team is squarely in scope as the operator of
   the upstream pipeline.
9. **Scoping in SOC 2 auditor / CISA / PMP without a reason.** These
   are prestige certifications that look good on a page and do not
   match the team's charter. Why this bites: budget displacement
   from governance credentials that *do* match the charter.
10. **No non-goals block.** A portfolio that will not say what it is
    *not* for is one that implicitly claims everything. Why this
    bites: when the security guild asks "wait, do you own SOC 2
    now?" the answer should be in writing.
11. **Fabricated framework sub-category IDs.** Citing "NIST AI RMF
    GOVERN 1.3.2" when NIST's current publication does not use that
    exact ID. Why this bites: anyone holding AIGP has read the RMF
    and will spot the fabrication.
12. **Treating certifications as a substitute for operational
    training.** An AIGP-credentialed engineer who has never read the
    team's own dataset card is not a substitute for a runbook. Why
    this bites: certifications are a floor and a hiring signal, not
    an operational control on their own.

## 6. References

Primary standards and authoritative sources (linked below). Cross-cite
local anchors to sibling exercises so a reviewer can traverse the
curriculum without leaving the artifact.

- **NIST AI Risk Management Framework** —
  https://www.nist.gov/itl/ai-risk-management-framework . Cited at the
  GOVERN function level; GOVERN names AI literacy and workforce
  training as inputs. Do not cite invented sub-category numbers.
- **OWASP Machine Learning Security Top 10** —
  https://owasp.org/www-project-machine-learning-security-top-10/ .
  Certifications are a workforce-side control for ML06:2023 AI Supply
  Chain Attacks (credentialed engineers are a mitigation for
  vendor-side risk).
- **MITRE ATLAS** — https://atlas.mitre.org/ . Workforce-side defender
  coverage. The portfolio is the credential half of defender
  readiness; runbooks are the operational half.
- **EU AI Act (Regulation (EU) 2024/1689)** —
  https://eur-lex.europa.eu/eli/reg/2024/1689/oj . Article 4 is the
  "AI literacy" obligation on providers and deployers of AI systems,
  applicable from 2 February 2025. The portfolio treats AIGP as the
  primary coverage credential for Article 4; internal training
  programmes are a complementary mechanism.
- **IAPP certification catalogue** — https://iapp.org/certify/ .
  Includes AIGP, CIPP/E, CIPM, CIPT. All prices and CPE cycle lengths
  on this portfolio are flagged approximate; verify at the IAPP site
  before each cohort.
- **ISO/IEC 42001:2023 (AI management system)** —
  https://www.iso.org/standard/81230.html . The standard is purchasable
  from ISO; training and lead-implementer/lead-auditor exams are run
  by accredited bodies (PECB, BSI, Exemplar Global, and others).
- **AWS Certification catalogue** —
  https://aws.amazon.com/certification/ . Verify active exam codes and
  prices; AWS updates its catalogue frequently (notably: the
  Data-Engineer Associate track launched after the legacy Data-Analytics
  Specialty retired; the ML-Specialty track is in transition toward an
  AI/ML Engineer associate at time of writing).
- **Google Cloud Certification catalogue** —
  https://cloud.google.com/learn/certification . Verify active
  recertification cadence; Google has adjusted cert validity windows
  historically.
- **Microsoft Learn credentials catalogue** —
  https://learn.microsoft.com/credentials/ . Microsoft retires and
  renames certifications more frequently than AWS or GCP; the
  DP-203 → DP-700 Fabric transition is one such example.
- **CompTIA Security+ exam page** —
  https://www.comptia.org/certifications/security .
  <!-- research-note: verify the current SY0-7xx exam code at exam booking -->
- **(ISC)² CISSP** — https://www.isc2.org/certifications/cissp .
  Included for reference; not slated on this portfolio.
- **PMI PMP** — https://www.pmi.org/certifications/project-management-pmp .
  Included for reference; non-goal.

Local curriculum anchors:

- `modules/mod-112-cross-functional-data-program-and-communication/exercise-01-data-program-charter-for-one-team/SOLUTION.md`
  — the SRDE charter this portfolio supports. The responsibilities in
  §2.8 are drawn from this charter.
- `modules/mod-112-cross-functional-data-program-and-communication/exercise-02-dataset-card-across-three-formats/`
  — the dataset-card deliverable that AIGP + CIPT credentials support.
- `modules/mod-112-cross-functional-data-program-and-communication/exercise-03-eu-ai-act-annex-xi-training-data-summary-draft/`
  — Annex XI summary work, supported by AIGP + ISO 42001.
- `modules/mod-112-cross-functional-data-program-and-communication/exercise-04-model-card-data-slice-authoring/`
  — model-card slice work; AIGP is the primary support cert.
- `modules/mod-111-data-quality-monitoring-and-regression/exercise-01-dataset-monitoring-contract-for-one-pipeline/SOLUTION.md`
  — the monitoring contract this portfolio's cloud-data-platform
  certifications support operationally.
- `modules/mod-110-pii-and-privacy-risk-for-training-data/`
  — privacy work (CIPP/E, CIPT, CIPM are the credential half).
- `modules/mod-107-labeling-operations-and-annotation-programs/`
  — vendor-labelling RFP work (CIPM + CIPT are the credential half).
