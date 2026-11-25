# Exercise 03 — GDPR and HIPAA Lawful-Basis Evidence Authoring (Solution)

## 1. Solution overview

This is a **design + documentation** exercise. The learner authors a
lawful-basis evidence packet for one slice of training data. The packet
has two legs: a GDPR leg that names the Article 6(1) lawful basis
(and, where special-category data is present, the Article 9(2)
derogation) and a HIPAA leg that names whether the slice contains
Protected Health Information (PHI) and, if so, which de-identification
pathway — Safe Harbor (45 CFR 164.514(b)(2)) or Expert Determination
(45 CFR 164.514(b)(1)) — is being claimed.

The packet is written for two audiences: counsel (who has to sign the
memo) and the pipeline reviewer (who has to verify that the
pipeline's filters match the memo's claims). The exercise is not a
legal education; its point is that each statute's claim leaves a
mechanical footprint in the pipeline, and the memo has to make that
footprint explicit.

> Nothing in this exercise is legal advice. The packet records the
> organisation's documented lawful-basis analysis for a specific
> slice; counsel must sign off before the slice is approved for
> training.

Deliverables (taken verbatim from the exercise contract):

- `GDPR_LAWFUL_BASIS.md` — the per-source memo under GDPR Article
  6(1) and, where applicable, Article 9(2). One section per data
  source in the slice; the balancing-test or explicit-consent record
  for each legitimate-interests or consent basis.
- `HIPAA_DEIDENTIFICATION.md` — the PHI-posture memo for the slice.
  If no PHI is present, a signed assertion of that fact; if PHI is
  present, the full Safe Harbor identifier table or an expert-
  determination report.
- `evidence/` — supporting artifacts: the opt-out registry snapshot,
  the DPIA (Data Protection Impact Assessment) excerpt, any vendor
  attestations, and the HIPAA expert-determination report if used.
- `sign_off.yaml` — the sign-off block naming counsel, the data
  controller (GDPR), the covered-entity contact (HIPAA, if
  applicable), and the pipeline reviewer; each with a timestamp and
  a document hash.
- `admit_policy.yaml` — the machine-readable policy that Exercise 01
  and Exercise 05 both read. Per-source constraints the ingestion
  pipeline must enforce (opt-out honoured, SCD excluded, etc.).

Governance framing. Lawful basis is the precondition for any
downstream privacy claim. The OWASP Machine Learning Security Top 10
anchor is **ML04:2023 Membership Inference Attack** — membership
inference against personal data processed without a lawful basis
multiplies the regulatory consequence of a technical vulnerability.
MITRE ATLAS does not enumerate statutory bases; the memo's role is
to make the defender-side evidence reviewable. The NIST AI Risk
Management Framework anchors are **GOVERN 1.2** (accountability
structures), **GOVERN 1.4** (processes for transparency), and
**MEASURE 2.10** (privacy risk characterisation across the
lifecycle). Cite the statute URLs verbatim in the References
section.

## 2. Implementation — worked answer

### 2.1 Packet layout

```
exercise-03/
├── README.md
├── GDPR_LAWFUL_BASIS.md
├── HIPAA_DEIDENTIFICATION.md
├── admit_policy.yaml
├── sign_off.yaml
└── evidence/
    ├── opt_out_registry_snapshot.jsonl
    ├── opt_out_registry_snapshot.jsonl.sha256
    ├── dpia_excerpt.md
    ├── vendor_attestations/
    │   ├── vendor_acme.pdf
    │   └── vendor_acme_attestation.yaml
    └── expert_determination_report.pdf        # only if that pathway is used
```

### 2.2 `GDPR_LAWFUL_BASIS.md` — worked content

Each source in the slice gets one section. The scaffold below is the
worked pattern; the learner replaces the placeholder values with the
slice's actual sources and justifications.

```markdown
# GDPR Lawful-Basis Memo — slice `<slice-id>` v<slice-version>

| Field                     | Value                                                 |
|---------------------------|-------------------------------------------------------|
| Slice id                  | <slice-id>                                            |
| Slice version             | <slice-version>                                       |
| Controller                | <controller legal name, address, DPO contact>         |
| Jurisdictional scope      | EU/UK-domiciled subjects observed in this slice: YES  |
| Processor(s)              | <processor-1, processor-2>                            |
| Memo version              | 1.0.0                                                 |

## Executive summary

This memo records, for each data source contributing to slice
`<slice-id>`, the GDPR Article 6(1) lawful basis under which personal
data is being processed for training. Where the source contains any
special-category data within the meaning of Article 9(1), the memo
also records the Article 9(2) derogation. The memo is written before
training; the pipeline's admission filter (`admit_policy.yaml`) is
the mechanical implementation of its conclusions.

## A. Internal support-ticket corpus

**Personal data present:** yes (customer names, email addresses,
service account IDs, free-text that may contain further identifiers).

**Special-category data (Article 9(1)):** excluded by filter; the
pipeline drops records flagged as containing health, religion,
political-opinion, or trade-union category terms per the SCD
classifier in `admit_policy.yaml`.

**Article 6(1) basis claimed:** Article 6(1)(b) — **performance of a
contract** — for records where the customer accepted the service
terms that cover "using transcripts to improve the service", and
Article 6(1)(f) — **legitimate interests** — for the residual.

**Article 6(1)(f) balancing test:** documented at
`evidence/dpia_excerpt.md` §3. Three-step test:

1. *Purpose test* — training the model on customer transcripts
   improves the service the customer purchased; the purpose is
   lawful and specific.
2. *Necessity test* — anonymised aggregate statistics would not
   provide the linguistic coverage the model needs; the
   less-intrusive alternative was rejected on evidence.
3. *Balancing test* — the pipeline's redaction (Exercise 01),
   suppression-on-request (opt-out), and no-public-release posture
   (UC-A, not UC-D) materially reduce the risk to the subject.
   The subject's reasonable expectation is covered by the service
   notice referenced in Appendix A of the DPIA excerpt.

**Rights posture:**

- *Right to object (Art. 21).* Honoured through the opt-out registry
  snapshot (`evidence/opt_out_registry_snapshot.jsonl`). The admit
  policy requires the pipeline to exclude every `subject_id` present
  in that snapshot.
- *Right to erasure (Art. 17).* Honoured at every slice rebuild. A
  standing erasure list is consumed at admit time; records matching
  are dropped.
- *Right to information (Arts. 13-14).* Satisfied by the service
  privacy notice v`<notice-version>`.

**Pipeline constraints (consumed by `admit_policy.yaml`):**

- `requires_pii_scrub: true`
- `honours_opt_out: true`
- `special_category_data_allowed: false`
- `erasure_list_path: evidence/erasure_standing_list.jsonl`

## B. Public web crawl (news and technical documentation)

**Personal data present:** incidental (bylines, quoted names,
occasional contact details on author pages).

**Special-category data (Article 9(1)):** excluded by filter.

**Article 6(1) basis claimed:** Article 6(1)(f) — legitimate
interests. The balancing test is documented in `evidence/dpia_
excerpt.md` §4.

**Robots / noai / opt-out signals honoured:** per-domain
`robots.txt`, `noai` meta directives, publisher-level Do-Not-Train
(DNT-AI) opt-outs. The admit policy enforces this at acquisition
time; records whose provenance sidecar lacks a crawl-time compliance
hash are rejected.

**Right to object posture:** honoured at next slice rebuild through
a published contact URL and a reconciled suppression list.

**Pipeline constraints (consumed by `admit_policy.yaml`):**

- `requires_pii_scrub: true`
- `honours_opt_out: true`
- `special_category_data_allowed: false`
- `crawl_time_robots_hash_required: true`

## C. Vendor-supplied conversation dataset

**Personal data present:** yes (end-user utterances).

**Special-category data (Article 9(1)):** contract warrants the
absence of SCD; a secondary SCD classifier is still run at ingest
as a defence-in-depth.

**Article 6(1) basis claimed:** basis depends on the vendor's
documented collection posture. Vendor must warrant the Article 6(1)
basis under which it collected the data (`vendor_attestations/
vendor_acme_attestation.yaml`) and must warrant that the onward
transfer to the controller for training is covered.

**Article 28 posture:** Data Processing Agreement (DPA) in place,
with sub-processor list, audit rights, breach-notification SLA, and
deletion-on-termination clause. Pin the DPA hash.

**Sanctions for non-compliance:** the admit policy rejects every
record from a vendor whose attestation is missing, unsigned, or
whose contract has lapsed. An unsigned attestation fails `admit`.

**Pipeline constraints (consumed by `admit_policy.yaml`):**

- `requires_pii_scrub: true`
- `honours_opt_out: true`
- `special_category_data_allowed: false`
- `vendor_attestation_required: true`
- `vendor_attestation_path: evidence/vendor_attestations/vendor_acme_attestation.yaml`

## D. Synthetic conversation corpus (seeded by B)

**Personal data present:** low — generated. Residual personal data
in the seed slice (B) carries through the lawful-basis chain from
that slice. The synthetic layer does not acquire a new lawful basis
of its own; it inherits.

**Pipeline constraints:**

- `inherits_lawful_basis_from: B`
- `requires_pii_scrub: true` (defence-in-depth against seed leakage)

## Appendix — DPIA excerpt

The excerpt at `evidence/dpia_excerpt.md` is the published portion of
the full DPIA. Sections §3 and §4 (balancing tests) are the two
load-bearing parts for this memo. The full DPIA is maintained
separately per the controller's record-keeping obligations.
```

### 2.3 `HIPAA_DEIDENTIFICATION.md` — worked content

If no PHI is present in the slice, the memo is a one-section signed
assertion. If PHI is present, the memo must name which pathway —
Safe Harbor or Expert Determination — is being claimed.

```markdown
# HIPAA De-identification Memo — slice `<slice-id>` v<slice-version>

| Field                       | Value                                               |
|-----------------------------|-----------------------------------------------------|
| Slice id                    | <slice-id>                                          |
| Slice version               | <slice-version>                                     |
| Covered entity / business associate | <legal-name or N/A>                           |
| Pathway claimed             | <"Safe Harbor" | "Expert Determination" | "No PHI"> |
| Memo version                | 1.0.0                                               |

## 1. PHI presence determination

The slice has been scanned for the 18 Safe Harbor identifiers (§2
below). The scan results are at `evidence/hipaa_scan_report.yaml`;
the scanner config hash is pinned in `admit_policy.yaml`.

Conclusion: `<"PHI present" | "No PHI identified">`.

## 2. Safe Harbor pathway (if claimed)

Safe Harbor requires that all 18 identifier classes be removed from
every record. The table below is the per-class treatment the
pipeline enforces; "Primitive" names the Exercise 02 operator used.

| # | Identifier class (45 CFR 164.514(b)(2)(i))                   | Pipeline primitive                                      | Evidence                              |
|---|---------------------------------------------------------------|---------------------------------------------------------|---------------------------------------|
| A | Names                                                         | `replace("[PERSON]")`                                    | Ex01 calibration PERSON row           |
| B | Geographic subdivisions smaller than a state                  | `generalize` (state-only); zip-3 pass-through per rule  | Ex01 calibration LOCATION row         |
| C | All elements of dates (except year) directly related to indiv. | `generalize_date_year`                                   | Ex01 recognizer DATE_TIME row         |
| D | Telephone numbers                                             | `replace("[PHONE]")`                                     | Ex01 calibration PHONE_NUMBER row     |
| E | Vehicle identifiers and serial numbers                        | `replace("[VEHICLE_ID]")` (custom recognizer)            | Ex01 recognizer VEHICLE_ID            |
| F | Device identifiers and serial numbers                         | `replace("[DEVICE_ID]")`                                 | Ex01 recognizer DEVICE_ID             |
| G | URLs                                                          | `hash` (truncate 16, UC-A) or `replace` (UC-D)           | Ex02 operator_table.yaml              |
| H | IP addresses                                                  | `replace("[IP]")`                                        | Ex01 calibration IP_ADDRESS row       |
| I | Biometric identifiers                                         | `suppress-document` (not permissible in training slice)  | Ex02 decision table                   |
| J | Full-face photographic images and any comparable images       | `suppress-document`                                      | Ex02 decision table                   |
| K | Fax numbers                                                   | `replace("[FAX]")`                                       | Ex01 recognizer FAX                   |
| L | Email addresses                                               | `replace("[EMAIL]")`                                     | Ex01 calibration EMAIL_ADDRESS row    |
| M | Social security numbers                                       | `redact`                                                 | Ex01 calibration US_SSN row           |
| N | Medical record numbers                                        | `suppress-document`                                      | Ex02 US_MEDICAL_RECORD row            |
| O | Health plan beneficiary numbers                               | `redact`                                                 | Ex02 US_HEALTH_PLAN_NUMBER row        |
| P | Account numbers                                               | `redact`                                                 | Ex02 ACCOUNT_NUMBER row               |
| Q | Certificate / license numbers                                 | `redact`                                                 | Ex02 CERTIFICATE_NUMBER row           |
| R | Any other unique identifying number, characteristic, or code  | Custom recognizer "R-residual" + `redact`                | Ex01 recognizer R_RESIDUAL            |

Residual-risk attestation (164.514(b)(2)(ii)): the covered entity
does not have actual knowledge that the information could be used
alone or in combination with other information to identify an
individual. The attestation is counter-signed in `sign_off.yaml`.

## 3. Expert Determination pathway (if claimed)

Expert Determination (45 CFR 164.514(b)(1)) requires a qualified
person who, applying generally accepted statistical and scientific
principles, determines that the risk is very small, and who
documents the methods and results of the analysis.

If this pathway is used, the report at
`evidence/expert_determination_report.pdf` includes:

1. The expert's credentials and the engagement letter.
2. The dataset description and the attacker model(s) analysed.
3. The disclosure-risk measure used (e.g., k-anonymity, cell-size,
   identification rates against a published linkage dataset).
4. The quantitative result and the "very small risk" threshold used.
5. A signed attestation on the covered entity's letterhead.

The expert's attestation is counter-signed in `sign_off.yaml`; the
report hash is pinned in `admit_policy.yaml`.

## 4. No-PHI assertion (if claimed)

If the slice contains no PHI, the memo records the scanner config
used to make the determination, the scanner config hash, and the
scan report hash. A false negative on a Safe Harbor identifier is the
risk; the scanner precision / recall on the micro-benchmark is the
evidence.
```

### 2.4 `admit_policy.yaml`

```yaml
# admit_policy.yaml
# Machine-readable policy consumed by Exercise 01 (ingest) and
# Exercise 05 (risk report). This file is the mechanical form of the
# two memos above.

slice:
  id: slice-example-v1
  version: 1.0.0

sources:
  - id: A-support-tickets
    gdpr_basis: "6(1)(b)+6(1)(f)"
    scd_allowed: false
    requires_pii_scrub: true
    honours_opt_out: true
    opt_out_list_path: evidence/opt_out_registry_snapshot.jsonl
    opt_out_list_sha256: <64-hex>
    erasure_list_path: evidence/erasure_standing_list.jsonl
    erasure_list_sha256: <64-hex>

  - id: B-public-web-crawl
    gdpr_basis: "6(1)(f)"
    scd_allowed: false
    requires_pii_scrub: true
    honours_opt_out: true
    crawl_time_robots_hash_required: true
    noai_meta_honoured: true

  - id: C-vendor-conversations
    gdpr_basis: "depends-on-vendor-attestation"
    scd_allowed: false
    requires_pii_scrub: true
    vendor_attestation_required: true
    vendor_attestation_path: evidence/vendor_attestations/vendor_acme_attestation.yaml
    vendor_attestation_sha256: <64-hex>
    dpa_sha256: <64-hex>

  - id: D-synthetic
    inherits_lawful_basis_from: B-public-web-crawl
    requires_pii_scrub: true

hipaa:
  pathway: Safe-Harbor
  phi_present: false
  scan_report_path: evidence/hipaa_scan_report.yaml
  scan_report_sha256: <64-hex>
  scanner_config_sha256: <64-hex>
```

### 2.5 `sign_off.yaml`

```yaml
# sign_off.yaml
# Counter-signed sign-off block. Each signer is a named person with
# a timestamp and a document-hash binding; the slice is not approved
# for training until every required row is signed.

slice:
  id: slice-example-v1
  version: 1.0.0

signers:
  - role: controller
    name: "<legal name>"
    org:  "<entity legal name>"
    email: "<controller-signer@org.example>"
    signed_document: GDPR_LAWFUL_BASIS.md
    document_sha256: <64-hex>
    signed_at_utc: 2026-11-25T14:00:00Z

  - role: counsel
    name: "<counsel of record>"
    org:  "<firm>"
    email: "<counsel@firm.example>"
    signed_document: GDPR_LAWFUL_BASIS.md
    document_sha256: <64-hex>
    signed_at_utc: 2026-11-25T14:05:00Z

  - role: covered-entity-contact        # only if PHI pathway used
    name: "<privacy officer>"
    org:  "<covered entity>"
    email: "<privacy@covered-entity.example>"
    signed_document: HIPAA_DEIDENTIFICATION.md
    document_sha256: <64-hex>
    signed_at_utc: 2026-11-25T14:10:00Z

  - role: pipeline-reviewer
    name: "<AI risk engineer>"
    org:  "<entity>"
    email: "<ai-risk@org.example>"
    signed_document: admit_policy.yaml
    document_sha256: <64-hex>
    signed_at_utc: 2026-11-25T14:15:00Z
```

Signatures are not merely ceremonial. The pipeline reviewer's role
is to verify that every constraint in `admit_policy.yaml` is in fact
enforced by Exercise 01's ingest filter. A sign-off whose
`document_sha256` does not match the file on disk fails validation.

### 2.6 `evidence/vendor_attestations/vendor_acme_attestation.yaml`

```yaml
# Vendor attestation template.
vendor:
  legal_name: "Acme Conversations Ltd."
  dpo_contact: "dpo@acme.example"

dataset:
  id: acme-conv-2026Q3
  item_count: 2341000
  collection_window_utc: 2024-01-01 .. 2026-09-30

lawful_basis:
  gdpr_article_6: "6(1)(a) explicit consent"
  gdpr_article_9_derogation: n/a
  consent_record_path: s3://acme-privacy/consents/2024-2026.parquet
  consent_record_sha256: <64-hex>

onward_transfer:
  purpose: "training of a general-purpose assistant"
  permitted: true
  dpa_version: 2.3.0
  dpa_sha256: <64-hex>

deletion_sla_days: 30

signed_by:
  name: "<acme-signer>"
  title: "Data Protection Officer"
  signed_at_utc: 2026-10-01T09:00:00Z
```

## 3. Validation steps

1. **Every source in the slice has a `GDPR_LAWFUL_BASIS.md` section
   AND a row in `admit_policy.yaml.sources`.** The two sides must
   match by `source_id`; any drift fails review.
2. **Every Article 6(1)(f) claim cites a balancing test** whose
   document path is in `evidence/`. A legitimate-interests claim
   without a referenced balancing test fails validation.
3. **Opt-out registry has a pinned hash.** `admit_policy.yaml`'s
   `opt_out_list_sha256` matches the contents of
   `evidence/opt_out_registry_snapshot.jsonl.sha256`.
4. **HIPAA memo names exactly one pathway.** `HIPAA_
   DEIDENTIFICATION.md` is one of {`Safe Harbor`, `Expert
   Determination`, `No PHI`}. A memo claiming two pathways fails.
5. **Safe Harbor table covers all 18 identifier classes.** If the
   pathway is Safe Harbor, every row A–R has a pipeline primitive
   and an evidence pointer. Missing rows fail validation.
6. **Expert Determination report has a counter-signature block.** If
   the pathway is Expert Determination, the expert's credentials and
   attestation are captured in `evidence/expert_determination_report.
   pdf`; the sign-off block references the report hash.
7. **Every vendor source has an attestation.** For every `vendor_
   attestation_required: true` row, the attestation path exists and
   the file's SHA-256 matches the pinned value.
8. **`sign_off.yaml` lists every required role.** Controller,
   counsel, and pipeline-reviewer are mandatory; covered-entity
   contact is required when the HIPAA pathway is not `No PHI`.
9. **Document hashes in `sign_off.yaml` match on-disk files.** Any
   document whose SHA-256 drifted from the signed hash invalidates
   the signature.
10. **Exercise 01's `admit()` function reads `admit_policy.yaml`
    without errors.** The policy's shape is byte-stable across the
    two exercises; a load-time shape failure fails validation.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros
on critical rows.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `GDPR_LAWFUL_BASIS.md` has a dedicated section per data source in the slice                                     | Yes       |
| 2  | Each GDPR source names an Article 6(1) basis (and Article 9(2) derogation if SCD present)                       | Yes       |
| 3  | Every 6(1)(f) claim cites a balancing test with a referenced evidence file                                      | Yes       |
| 4  | Right-to-object and right-to-erasure postures named per source                                                  | Yes       |
| 5  | `HIPAA_DEIDENTIFICATION.md` declares exactly one pathway (Safe Harbor / Expert Determination / No PHI)          | Yes       |
| 6  | If Safe Harbor: all 18 identifier classes covered with a pipeline primitive + evidence pointer                  | Yes       |
| 7  | If Expert Determination: report present with expert credentials, methods, threshold, and signed attestation     | Yes       |
| 8  | If No PHI: scanner config + scan-report hash pinned and precision / recall evidence referenced                  | Yes       |
| 9  | `admit_policy.yaml` has one row per source and matches memo claims                                              | Yes       |
| 10 | Vendor attestation files are referenced and SHA-256-pinned for every vendor source                              | Yes       |
| 11 | `sign_off.yaml` names controller, counsel, pipeline-reviewer (and covered-entity contact if PHI pathway is set) | Yes       |
| 12 | Every signed document has a SHA-256 in `sign_off.yaml` that matches the file on disk                            | Yes       |
| 13 | Cross-references to Exercise 01 / Exercise 02 are specific (recognizer name or operator table row)              | No        |
| 14 | Governance anchors (OWASP ML Top 10 ML04, NIST AI RMF GOVERN 1.2 / 1.4 / MEASURE 2.10)                          | No        |
| 15 | Unverifiable claims are marked `<!-- pending-verify: ... -->`                                                   | Yes       |

## 5. Common mistakes

- **Claiming "consent" as the catch-all lawful basis.** GDPR consent
  under Article 6(1)(a) is a high bar: freely given, specific,
  informed, unambiguous, and withdrawable at any time. Training on
  historical data under newly-claimed consent is usually not
  defensible; use the correct basis.
- **A legitimate-interests claim without a balancing test.** The
  balancing test is the mechanism by which Article 6(1)(f) is
  defensible; omitting it collapses the entire basis.
- **Treating "pseudonymised" data as out-of-scope.** GDPR Recital 26
  is explicit — pseudonymous data is still personal. The memo must
  cover pseudonymised data and the key-management posture.
- **Running one memo for a multi-source slice.** Each source has its
  own lawful basis. A single section for the whole slice silently
  flattens the analysis.
- **Mixing Safe Harbor and Expert Determination.** Only one pathway
  is claimed per slice. A memo that says "Safe Harbor for most
  identifiers, Expert Determination for the residual" is not a valid
  HIPAA de-identification claim.
- **Omitting the residual-knowledge attestation.** Safe Harbor
  requires the covered entity to attest it has no actual knowledge
  that the de-identified data could be used to identify an
  individual. Without the attestation, Safe Harbor is incomplete.
- **Over-trusting the scanner.** The No-PHI assertion depends on the
  scanner's recall; without a precision / recall number on a
  labelled benchmark, the assertion is unsupported.
- **Not signing the file hash.** A signature that names a document
  but not its SHA-256 is cosmetic; the document can be edited after
  signature. Pin the hash.
- **Not refreshing the opt-out snapshot on slice rebuild.** The
  right-to-object posture requires the ingest step to consume the
  *current* snapshot; a stale snapshot is a right-to-object breach.
- **Vendor contract without a deletion SLA.** The admit policy must
  encode the deletion-on-termination clause; a vendor whose contract
  has expired or whose deletion window has passed must be excluded
  on the next rebuild.
- **`dpia_excerpt.md` with no public form.** The excerpt must be
  the portion of the DPIA that is reviewable by the auditor; a memo
  that references the DPIA but never attaches the excerpt is
  unverifiable.
- **Ignoring Article 22 / automated decision-making.** If the trained
  model will be used in a way that constitutes solely-automated
  decision-making with legal or similarly significant effects on a
  natural person, the lawful-basis memo must call this out so the
  controller can also satisfy Article 22's safeguards.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML04:2023 Membership
  Inference Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchor for the attacker-facing risk that a missing
  lawful basis amplifies.
- **MITRE ATLAS:** <https://atlas.mitre.org/>. The adversary-facing
  technique index the pipeline's controls are designed to blunt.
- **NIST AI Risk Management Framework — GOVERN 1.2 / 1.4,
  MEASURE 2.10:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN
  1.2 and 1.4 anchor accountability and transparency; MEASURE 2.10
  anchors the privacy-risk characterisation that the memo supports.
- **GDPR Article 6 — Lawfulness of processing:**
  <https://gdpr-info.eu/art-6-gdpr/>. The statute for the Article
  6(1)(a)–(f) bases cited in §2.2.
- **GDPR Article 9 — Processing of special categories of personal
  data:** <https://gdpr-info.eu/art-9-gdpr/>. The statute for the
  Article 9(1) prohibition and the 9(2) derogations.
- **GDPR Article 17 — Right to erasure:**
  <https://gdpr-info.eu/art-17-gdpr/>. The statute for the erasure
  posture the admit policy must honour.
- **GDPR Article 21 — Right to object:**
  <https://gdpr-info.eu/art-21-gdpr/>. The statute for the
  right-to-object posture (opt-out registry).
- **GDPR Article 22 — Automated individual decision-making:**
  <https://gdpr-info.eu/art-22-gdpr/>. The statute the memo calls
  out when the trained model's downstream use falls under
  Article 22.
- **GDPR Article 35 — Data Protection Impact Assessment (DPIA):**
  <https://gdpr-info.eu/art-35-gdpr/>. The statute for the DPIA the
  memo's balancing-test excerpt references.
- **GDPR Recital 26 — Anonymous information:**
  <https://gdpr-info.eu/recitals/no-26/>. The recital that
  distinguishes anonymous from pseudonymous data.
- **HIPAA Privacy Rule — 45 CFR 164.514(b):**
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>.
  The regulatory anchor for both Expert Determination (b)(1) and
  Safe Harbor (b)(2); the Safe Harbor identifier list A–R comes
  verbatim from (b)(2)(i).
- **HHS Office for Civil Rights — Guidance Regarding Methods for
  De-identification of Protected Health Information:**
  <https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html>.
  The authoritative interpretive guidance for Safe Harbor and
  Expert Determination; cite when defending the choice of pathway.
- **Local anchors:** Exercise 01 (the pipeline whose admit filter
  consumes `admit_policy.yaml`); Exercise 02 (the operator table
  the HIPAA Safe Harbor table calls into); Exercise 04 (the
  extraction probe whose results strengthen or weaken the memo's
  residual-risk assertions); Exercise 05 (the privacy risk report
  that aggregates the lawful-basis evidence into the published
  packet).
