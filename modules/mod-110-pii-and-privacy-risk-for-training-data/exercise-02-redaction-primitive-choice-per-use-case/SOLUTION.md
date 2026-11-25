# Exercise 02 — Redaction Primitive Choice per Use Case (Solution)

## 1. Solution overview

This is a **design** exercise. The learner authors a per-use-case
redaction-primitive table that maps each PII entity type, each
corpus purpose, and each downstream consumer to one of the documented
primitives (`replace`, `mask`, `hash`, `redact`, `encrypt`,
`pseudonymize`, `suppress-document`). The deliverable is the decision
table, a short rationale per row, and a rubric a reviewer can apply.

The exercise is not "which primitive is correct"; the question has no
single answer. The exercise is "given a declared use case and an
entity, which primitive is *defensible*, and what evidence does the
author owe a reviewer?"

Deliverables (taken verbatim from the exercise contract):

- `decision_table.md` — the primary artifact. A per-entity, per-use-
  case matrix with the chosen primitive and a one-line justification
  that cites the primitive's documented trade-off.
- `rationale.md` — three worked rationales (one per high-risk entity:
  `EMAIL_ADDRESS`, `US_SSN`, `PERSON`), each ≥ 200 words, naming the
  downstream consumer, the attacker model, and the specific trade-off
  the chosen primitive makes.
- `operator_table.yaml` — the decision table rendered into the
  Presidio anonymizer operator schema used by Exercise 01. The schema
  must load in the Presidio anonymizer without error.
- `rubric.md` — the grading rubric for other learners, derived from
  the one in §4.

Governance framing. Redaction primitive choice is where privacy
engineering meets utility engineering. The OWASP Machine Learning
Security Top 10 anchor is **ML04:2023 Membership Inference Attack** —
each primitive has a published informational-loss profile, and a
weak primitive (e.g., consistent `hash` of a short identifier) is
re-identifiable by an attacker who can enumerate the input space.
The MITRE ATLAS technique in scope is **AML.T0024 Exfiltration via
ML Inference API** — the attacker-facing anchor the primitive
choice is designed to blunt. The NIST AI Risk Management Framework
anchors are **GOVERN 1.2** (accountability for pipeline decisions —
the decision table is the written-down accountability) and
**MEASURE 2.10** (privacy risk characterisation — the rationale in
`rationale.md` is the per-entity characterisation).

## 2. Implementation — worked answer

### 2.1 The primitive taxonomy (pinned vocabulary)

Any row in the decision table must map an `(entity, use_case,
consumer)` triple to exactly one of these primitives. The names are
the Presidio anonymizer operator names where they overlap; where
Presidio has no direct primitive, the recipe calls a custom operator.

| Primitive          | What it does                                      | Reversibility     | Preserves length? | Preserves distribution? | Linkability across docs? |
|--------------------|---------------------------------------------------|-------------------|-------------------|--------------------------|---------------------------|
| `replace`          | Overwrites the span with a fixed string           | No                | No                | No                       | No                        |
| `mask`             | Overwrites characters with a fill char            | No                | Yes (partial)     | Partial                  | Low (depends on bits kept)|
| `hash`             | SHA-256 → fixed-length hex                        | No, but pre-image enumerable | No    | No                       | Yes (consistent per value) |
| `redact`           | Deletes the span                                  | No                | No                | No                       | No                        |
| `encrypt`          | Reversible with the key                           | Yes, with the key | No                | No                       | Yes                       |
| `pseudonymize`     | Deterministic surrogate from a keyed map          | Yes (via key)     | Variable          | Variable                 | Yes                       |
| `suppress-document`| Drops the entire document                         | N/A               | N/A               | N/A                      | N/A                       |

Three facts the decision table leans on:

- **`hash` is NOT anonymisation for short / enumerable spaces.** A
  consistent SHA-256 of a US SSN or a phone number is trivially
  invertible by rainbow table. `hash` is defensible only when the
  pre-image space is large and the use case genuinely needs
  consistency across documents.
- **`mask` is a spectrum.** Leaking the last four of an SSN is a
  linkable quasi-identifier in many jurisdictions. `mask` requires
  specifying `chars_to_mask` and `from_end` and justifying the
  number of retained characters.
- **`suppress-document` is sometimes correct.** For small corpora,
  for sensitive-category data (GDPR Art. 9), or for data whose
  provenance cannot be re-established, dropping the whole document
  is the only defensible choice. The decision table must offer this
  row.

### 2.2 `decision_table.md`

The table rows are a Cartesian product of `(entity × use case)`;
columns name the chosen primitive and the required review evidence.
Four use cases cover the common deployment surface.

Use cases:

- **UC-A (pretrain-SFT).** Corpus is used to pretrain or
  supervised-fine-tune a general-purpose model. Consumer is the
  model; the model's output is externally observable to end users.
- **UC-B (eval-and-telemetry).** Corpus is used for offline
  evaluation and operational telemetry; consumer is an internal
  team; outputs are not shipped externally.
- **UC-C (eng-debugging).** Corpus slice is attached to a bug
  ticket; consumer is a specific engineer, within a logged, audited
  access window.
- **UC-D (released-dataset).** Corpus is published as a third-party
  dataset for research use; consumer is the open community.

| Entity              | UC-A (pretrain-SFT)       | UC-B (eval-telemetry) | UC-C (eng-debug)                   | UC-D (released)          |
|---------------------|---------------------------|-----------------------|-------------------------------------|--------------------------|
| `EMAIL_ADDRESS`     | `replace("[EMAIL]")`      | `replace("[EMAIL]")`  | `pseudonymize` (keyed, per-user)    | `replace("[EMAIL]")`     |
| `PHONE_NUMBER`      | `replace("[PHONE]")`      | `replace("[PHONE]")`  | `replace("[PHONE]")`                | `replace("[PHONE]")`     |
| `US_SSN`            | `redact`                  | `redact`              | `redact`                            | `suppress-document`      |
| `CREDIT_CARD`       | `redact`                  | `redact`              | `redact`                            | `suppress-document`      |
| `IBAN_CODE`         | `redact`                  | `redact`              | `redact`                            | `suppress-document`      |
| `IP_ADDRESS`        | `replace("[IP]")`         | `pseudonymize`        | `pseudonymize`                      | `replace("[IP]")`        |
| `URL`               | `hash` (16-char prefix)   | `hash` (16-char)      | `hash` (16-char)                    | `replace("[URL]")`       |
| `PERSON`            | `replace("[PERSON]")`     | `replace("[PERSON]")` | `pseudonymize` (keyed, per-doc)     | `replace("[PERSON]")`    |
| `LOCATION`          | `replace("[LOCATION]")`   | `replace("[LOC]")`    | `replace("[LOC]")`                  | `replace("[LOCATION]")`  |
| `DATE_TIME`         | `generalize` (year-only)  | pass-through          | pass-through                        | `generalize` (year-only) |
| `INTERNAL_USER_ID`  | `hash` (16-char)          | `hash` (16-char)      | `pseudonymize` (keyed)              | `suppress-document`      |
| `UK_NHS_NUMBER`     | `redact`                  | `redact`              | `redact`                            | `suppress-document`      |
| `US_MEDICAL_RECORD` | `suppress-document`       | `redact`              | `redact`                            | `suppress-document`      |
| `AU_ABN` / `UK_NINO`| `redact`                  | `redact`              | `redact`                            | `suppress-document`      |
| `CRYPTO_WALLET`     | `replace("[WALLET]")`     | `hash`                | `hash`                              | `suppress-document`      |

Each row carries a one-line rationale tag; the five tags are the
only vocabulary the table uses so a reviewer can audit coverage.

Rationale tags (defined once, referenced per row):

- **T1 Memorisation-sensitive.** The entity is a short, high-entropy
  string the model is likely to memorise and regurgitate verbatim.
  Primitive must prevent any token-level reconstruction. Evidence:
  cite the Carlini et al. finding that small, duplicated, PII-shaped
  strings have the highest extraction rate (see Exercise 04).
- **T2 Re-identifiable quasi-identifier.** The entity, even when
  partially removed, remains linkable. Pure `mask` with retained tail
  characters is forbidden; use `redact` or `pseudonymize`.
- **T3 Downstream-utility-required.** A downstream consumer needs
  within-document or cross-document consistency to function (e.g.,
  an eng-debug flow needs to know two mentions of the same user are
  the same user). `pseudonymize` with a per-corpus key is the
  default.
- **T4 Public-value preserving.** The primitive strips the personal
  dimension while preserving the public dimension — e.g., `generalize`
  for `DATE_TIME` keeps year-level temporal context.
- **T5 Suppress-document.** The entity's presence invalidates the
  document for this use case regardless of redaction quality.

### 2.3 `rationale.md` — three worked rationales

The learner writes three rationales of ≥ 200 words each. Worked
examples below define the acceptable shape.

#### R1 — `EMAIL_ADDRESS` in UC-A (pretrain-SFT)

**Chosen primitive:** `replace("[EMAIL]")`.

**Downstream consumer:** The pretrained / SFT model, whose output is
visible to external end users.

**Attacker model:** An external user who can elicit completions and
whose goal is to extract memorised personal addresses from the model
(Carlini-style extraction; see Exercise 04). Lower-effort attackers
rely on completion ("my email address is …"); higher-effort
attackers use prefix templates that correlate with training data.

**Why `replace` and not `hash` or `pseudonymize`:**

- `hash` on an email is a weak primitive here. Email addresses are
  drawn from a modestly-sized space (name + known domain). A SHA-256
  prefix of a public-domain email is enumerable in minutes with a
  dictionary of names; the model learns the hash as a stable
  identifier that still leaks the local-part structure in aggregate
  statistics. Tag T2.
- `pseudonymize` with a keyed map is a stronger primitive for
  downstream engineering flows (UC-C) because the key gates
  reversibility; for UC-A, the model should not be able to learn
  that `alice@example.com` and `bob@example.com` are structurally
  different entities, so the fixed placeholder is correct. Tag T1.
- `redact` is defensible but loses the typographic signal ("the user
  wrote an email here"). `replace("[EMAIL]")` preserves that signal
  without preserving the content. Tag T1 + T4.

**Trade-offs made explicit:**

- The model will not learn to emit valid-looking emails at the right
  place in a template. For most pretrain-SFT objectives this is
  acceptable; for an email-assistant fine-tune, the operator needs
  synthetic emails in training via a different primitive.
- The `[EMAIL]` placeholder is itself a learnable token. The
  pretrained model will emit `[EMAIL]` in contexts the training data
  suggests, which is a feature (the model is visibly aware that an
  email went here) and a minor fingerprint of the training set.

**Evidence owed to the reviewer:** Exercise 01's `calibration.md`
precision / recall numbers for `EMAIL_ADDRESS`, and the
`findings.parquet` SHA-256 of the row count from the slice.

#### R2 — `US_SSN` in UC-D (released dataset)

**Chosen primitive:** `suppress-document`.

**Downstream consumer:** Any external researcher.

**Attacker model:** Anyone with the published dataset. Published
datasets receive adversarial attention the engineering team cannot
revisit after release; a mistake published is, practically speaking,
not recallable.

**Why `suppress-document` and not `redact`:**

- `redact` on `US_SSN` is correct for internal uses (UC-A/B/C). For a
  release, the residual risk from scanner misses — even a low-rate
  miss — is unbounded over the lifetime of the released dataset.
  A conservative `suppress-document` costs utility for the slice
  but caps the residual risk. Tag T5.
- The exercise is explicit that `US_SSN` does not appear in any
  defensible training-data release corpus the public community has
  shipped (see HIPAA Safe Harbor's treatment of SSN; see GDPR's
  treatment of national identifiers). The default is to suppress.

**Trade-off made explicit:**

- Documents containing a false-positive `US_SSN` detection are also
  dropped. This is a recall-biased choice; the calibration note in
  Exercise 01 must report the per-entity false-positive rate so a
  reviewer can quantify the utility cost of suppressing.

**Evidence owed to the reviewer:** The `US_SSN` false-positive rate
from `calibration/calibration.md`, and the count of documents
suppressed under this rule in the slice's privacy risk report
(Exercise 05).

#### R3 — `PERSON` in UC-C (eng-debugging)

**Chosen primitive:** `pseudonymize` with a keyed per-document map.

**Downstream consumer:** One or more named engineers, acting within a
logged, time-bounded access window on a bug ticket.

**Attacker model:** An engineer whose access was approved but whose
inference about the subject identities is not authorised. The
pseudonym map must be inside a key the debug environment holds but
the engineer cannot exfiltrate.

**Why `pseudonymize` and not `replace`:**

- UC-C's whole point is cross-mention linkage within a document.
  `replace("[PERSON]")` collapses every person in a thread to the
  same string, and most bug repros lose their reproducibility. Tag
  T3.
- `pseudonymize` with a per-document key preserves the linkage within
  the document ("Alice mentioned Bob to Alice again") without
  linking across documents (two documents' `Alice_1` are not the
  same person). Keyed, per-corpus maps are a weaker primitive here
  because an engineer with access to two documents can build a cross-
  document graph that re-identifies.

**Trade-offs made explicit:**

- Pseudonymisation is reversible with the key. The key must live in
  the debug environment's secret store with an access policy that
  logs every read.
- The map is a privacy liability of its own; it must be rotated
  per-ticket and destroyed at ticket close.

**Evidence owed to the reviewer:** The per-document key rotation
policy, the secret-store access log for the debug window, and the
destroyed-at timestamp.

### 2.4 `operator_table.yaml`

```yaml
# operator_table.yaml
# Rendered from decision_table.md for UC-A (pretrain-SFT).
# One operator_table file per use case; the consumer selects the file.

use_case: UC-A-pretrain-sft
operators:
  DEFAULT:
    type: replace
    new_value: "[REDACTED]"
  EMAIL_ADDRESS:
    type: replace
    new_value: "[EMAIL]"
  PHONE_NUMBER:
    type: replace
    new_value: "[PHONE]"
  US_SSN:
    type: redact
  CREDIT_CARD:
    type: redact
  IBAN_CODE:
    type: redact
  IP_ADDRESS:
    type: replace
    new_value: "[IP]"
  URL:
    type: hash
    hash_type: sha256
    # 16-char hex prefix is enough to deconflict unique URLs in a
    # corpus of ~10M docs without reconstructing the URL.
    truncate_to: 16
  PERSON:
    type: replace
    new_value: "[PERSON]"
  LOCATION:
    type: replace
    new_value: "[LOCATION]"
  DATE_TIME:
    type: custom
    operator_name: generalize_date_year
  INTERNAL_USER_ID:
    type: hash
    hash_type: sha256
    truncate_to: 16
  UK_NHS_NUMBER:
    type: redact
  US_MEDICAL_RECORD:
    type: suppress-document
  AU_ABN:
    type: redact
  UK_NINO:
    type: redact
  CRYPTO_WALLET:
    type: replace
    new_value: "[WALLET]"
```

Two design choices defensible at review:

- **`URL` is hashed, not replaced, in UC-A.** URLs carry both
  identification (they may embed a user token in a path segment) and
  utility signal (domain-level). A 16-char SHA-256 prefix deconflicts
  a ~10M-doc corpus without being invertible for a ~trillion-URL
  input space. The micro-benchmark must contain tokenised-URL
  negatives.
- **`DATE_TIME` is generalised rather than replaced in UC-A.** A
  training corpus that erases all temporal context damages the
  model's ability to reason about time; `generalize_date_year` is a
  documented primitive that keeps the year while dropping sub-year
  resolution. The custom operator is in `code/operators.py`.

### 2.5 `rubric.md`

See §4 for the rubric this file mirrors.

## 3. Validation steps

1. **Every entity in Exercise 01's recognizer registry has a row in
   `decision_table.md` for every declared use case.** `yq '.analyzer |
   .recognizers.registry + .recognizers.custom' recipes/presidio_
   pipeline.yaml` names the exhaustive set.
2. **Every row carries one of the five rationale tags** (T1 – T5).
   A row without a tag fails review.
3. **Every row maps to one and only one primitive** from the pinned
   taxonomy in §2.1.
4. **`operator_table.yaml` loads in the Presidio anonymizer without
   error.** `python -c "from presidio_anonymizer.entities import
   OperatorConfig; …"` constructs an operator for every entry.
5. **No `mask` operator retains more than `n/2` characters of a
   fixed-length identifier.** `mask` with `chars_to_mask: 4` on a
   9-digit SSN fails the pre-publish check; the test is in
   `tests/test_mask_sanity.py`.
6. **`hash` operator is not used on any entity whose pre-image space
   is < 2^40.** US_SSN (10^9), PHONE_NUMBER (10^10) fail; a `hash`
   row on either is a review failure.
7. **UC-D (released) has no row containing `hash`, `mask`,
   `pseudonymize`, or `encrypt`.** Released datasets must be
   one-way. The pre-publish check enforces this.
8. **Each of the three worked rationales in `rationale.md` names a
   downstream consumer, an attacker model, and at least one trade-
   off the primitive makes.** A rationale that only cites a
   primitive's property without naming the consumer fails.
9. **Entity coverage against Exercise 05's privacy risk report.** The
   set of entities whose residual rate is reported in the risk report
   is a subset of the set of entities in `decision_table.md`.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 20/28 with zero zeros
on critical rows.

| #  | Dimension                                                                                                  | Critical? |
|----|------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `decision_table.md` covers every entity registered in Exercise 01 across all four use cases                | Yes       |
| 2  | Every table cell is tagged with exactly one rationale tag (T1–T5)                                          | Yes       |
| 3  | Every table cell maps to one and only one primitive from the §2.1 taxonomy                                 | Yes       |
| 4  | `hash` is never used on a short, enumerable identifier (US_SSN, PHONE_NUMBER)                              | Yes       |
| 5  | UC-D (released) contains no reversible primitive (`hash`, `mask`, `pseudonymize`, `encrypt`)               | Yes       |
| 6  | `mask` operators specify `chars_to_mask` and `from_end`, and retain ≤ `n/2` characters of fixed-length IDs | Yes       |
| 7  | `rationale.md` has three ≥ 200-word rationales, one per the three high-risk entities                       | Yes       |
| 8  | Each rationale names consumer, attacker model, and at least one trade-off                                  | Yes       |
| 9  | `operator_table.yaml` loads in the Presidio anonymizer without error                                       | Yes       |
| 10 | Governance anchors (OWASP ML Top 10 ML04, MITRE ATLAS AML.T0024, NIST AI RMF GOVERN 1.2 / MEASURE 2.10)    | No        |
| 11 | Entity coverage against Exercise 05's risk report matches                                                  | Yes       |
| 12 | Table references the exact Presidio operator names (not paraphrases)                                       | Yes       |
| 13 | Unverifiable claims are marked `<!-- pending-verify: ... -->`                                              | Yes       |
| 14 | `rubric.md` is included and derived from this rubric                                                       | No        |

## 5. Common mistakes

- **Hashing short identifiers as "anonymisation".** SHA-256 is not a
  privacy primitive when the pre-image is enumerable. US_SSN,
  PHONE_NUMBER, and CREDIT_CARD are all enumerable in minutes with
  common tooling.
- **`mask` with the last-four pattern.** Retaining the last four of a
  9-digit SSN leaves roughly 10^4 of the entropy in the output; the
  retained tail correlates with the first-five bits in published
  linkage attacks.
- **Pretending `pseudonymize` is irreversible.** A keyed surrogate is
  reversible with the key; the policy must name where the key lives,
  who holds it, and when it is rotated or destroyed.
- **Flattening the use-case axis.** "One operator table for the
  whole program" is the single biggest error. UC-A and UC-D have
  different attacker models and different acceptable residual risk;
  one table cannot defensibly serve both.
- **Omitting the `DEFAULT` operator.** A new recognizer added in
  Exercise 01 without a matching row in the operator table silently
  falls through to the Presidio soft default. Declare `DEFAULT:
  replace` explicitly.
- **Leaving `DATE_TIME` as pass-through for UC-A.** Fine-grained
  timestamps carry enough linkage information to re-identify
  individuals when combined with other quasi-identifiers. Generalise
  or suppress.
- **Treating `suppress-document` as a failure case.** For UC-D or for
  GDPR Art. 9 special categories, suppressing is sometimes the only
  defensible option. The decision table must offer this row — and the
  rationale must own the utility cost.
- **No negative examples in the micro-benchmark.** If the operator
  table is only exercised against positive spans (entities the
  recognizer flagged), the false-positive cost of `redact` and
  `suppress-document` is invisible.
- **Primitive name drift from Presidio.** The decision table names
  `replace`, `mask`, `hash`, `redact`. If the operator table uses
  `substitute` or `blank_out`, reviewers cannot cross-check against
  the Presidio reference.
- **Missing per-row evidence pointer.** A row that says `redact` with
  no pointer to the calibration file or the risk-report row is an
  unsupported claim.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML04:2023 Membership
  Inference Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  The governance anchor for the attacker-facing re-identification
  risk each primitive makes decisions about.
- **MITRE ATLAS — AML.T0024 Exfiltration via ML Inference API:**
  <https://atlas.mitre.org/>. The adversary technique the primitive
  choice is designed to blunt.
- **NIST AI Risk Management Framework — GOVERN 1.2, MEASURE 2.10:**
  <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN
  1.2 anchors accountability for the decision table; MEASURE 2.10
  anchors the per-entity risk characterisation in `rationale.md`.
- **Microsoft Presidio Anonymizer — Operators reference:**
  <https://microsoft.github.io/presidio/anonymizer/>. The source for
  the exact operator names (`replace`, `mask`, `hash`, `redact`,
  `encrypt`, `custom`) and their parameter signatures.
- **GDPR Article 4(5) — Pseudonymisation:**
  <https://gdpr-info.eu/art-4-gdpr/>. The statutory definition of
  pseudonymisation used in R3's rationale ("keyed, reversible-with-
  the-key, separable additional information").
- **GDPR Recital 26 — Anonymous information:**
  <https://gdpr-info.eu/recitals/no-26/>. The recital that
  distinguishes anonymous data (no longer personal) from
  pseudonymous data (still personal); the basis for the "reversible
  primitives are forbidden in UC-D" rule.
- **HIPAA Safe Harbor — 45 CFR 164.514(b)(2):**
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>.
  The regulatory anchor for medical identifier suppression;
  motivates the `US_MEDICAL_RECORD: suppress-document` row in UC-A.
- **Carlini et al., Extracting Training Data from Large Language
  Models, USENIX Security 2021, arXiv:2012.07805:**
  <https://arxiv.org/abs/2012.07805>. The empirical basis for T1:
  short, high-entropy PII strings have the highest extraction rate.
- **Local anchors:** Exercise 01 (`recipes/presidio_pipeline.yaml`
  consumes `operator_table.yaml`); Exercise 03 (lawful basis — the
  statute that motivates UC-D's reversibility ban); Exercise 04
  (extraction probe — the empirical evidence for T1); Exercise 05
  (privacy risk report — the aggregate evidence package that cites
  the decision table).
