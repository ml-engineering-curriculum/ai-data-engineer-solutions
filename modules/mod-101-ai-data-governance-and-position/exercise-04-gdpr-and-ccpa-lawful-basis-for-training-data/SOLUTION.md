# Exercise 04 — GDPR / CCPA Lawful Basis and the Admit Function (Solution)

## 1. Solution overview

This is a **design + implementation** exercise. The learner writes a decision
function (`admit(record) -> AdmitDecision`) that decides whether a candidate
training record is admissible, grounded in a documented lawful-basis analysis
under the EU GDPR and, where applicable, the CCPA/CPRA (California).

Two deliverables:

1. A **lawful-basis memo** that, for each candidate source, names the legal
   basis under which personal data in that source is being processed for
   training, documents the balancing / notice tests that basis requires, and
   names the per-source constraints the admit function must enforce.
2. A **worked `admit` function** — a small, statically-valid Python module —
   that consumes a record and the per-source policy and returns an
   `AdmitDecision(admit: bool, reason: str, obligations: list[str])`. The
   function is the mechanical side of the memo: its branches must match
   the memo's conclusions.

The point is not a clever policy engine. The point is that the legal analysis
is written down *and* reflected in code that an auditor can read — and that
the record-level answer matches the source-level memo.

> Nothing in this exercise is legal advice. The `admit` function expresses
> an organisation's documented policy; it does not interpret the statute on
> its own. Have counsel sign off on the memo before shipping.

## 2. Implementation — worked answer

### 2.1 Lawful-basis memo (worked example, four sources)

For each source, name:

- Whether personal data is likely present (even incidentally).
- The GDPR Article 6(1) lawful basis being claimed, if any.
- The GDPR Article 9(2) derogation, if any special-category data is present.
- The CCPA posture (notice at collection; opt-out of "sale/share" and of
  profiling / ADMT where applicable).
- Operational constraints the admit function must enforce.

| Source                               | Personal data present?       | GDPR Art. 6(1) basis       | GDPR Art. 9(2) derogation (if SCD) | CCPA posture                                       | Operational constraints                                                                                                                      |
|--------------------------------------|------------------------------|----------------------------|-------------------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| A. Internal support-ticket corpus    | Yes — employee / customer    | Legitimate interests (f), balancing test documented; alternative: performance of contract (b) where the service contract covers improvement | Not used (SCD excluded by filter) | Covered by employer/service privacy notice; opt-out honoured | PII scrub (names, emails, phone, IDs); exclude records flagged `do-not-train`; honour per-subject erasure requests on rebuild              |
| B. Public web crawl (news / docs)    | Incidental — bylines, quotes | Legitimate interests (f), balancing test documented | Excluded by SCD filter      | Not primarily CA-subject data; general-notice posture; respect US opt-out signals where present | Robots/noai honoured at acquisition; per-domain opt-out list; drop records containing non-public contact info                                |
| C. Vendor-provided conversation data | Yes — end-users              | Depends on vendor contract — vendor must warrant the lawful basis under which it collected and shared the data | Contract must warrant SCD handling | Vendor warrants CCPA notice & opt-out posture | Reject the dataset if the vendor cannot evidence the lawful basis; preserve vendor attestation; enforce contractual deletion deadlines     |
| D. Synthetic-conversation corpus     | Low — generated              | Not applicable to the synthetic layer directly; applies to any seed data ingested | N/A                         | N/A                                                 | Treat the generator's terms as the licence; store seed-data lawful-basis chain                                                               |

### 2.2 The admit function (worked implementation)

A single-file Python module. No I/O, no mutable globals; a pure function on
a record and a per-source policy object. The intent is auditability — a
reviewer should be able to read the function top-to-bottom and reconcile it
with the memo above.

```python
# admit.py
"""Lawful-basis admit function for training-data records.

This module implements the policy expressed in the lawful-basis memo.
It is intentionally small and dependency-free so that a reviewer can
reconcile it with the memo on a single screen.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from enum import Enum
from typing import Mapping, Sequence


class LawfulBasisGDPR(str, Enum):
    CONSENT = "6(1)(a)"
    CONTRACT = "6(1)(b)"
    LEGAL_OBLIGATION = "6(1)(c)"
    VITAL_INTERESTS = "6(1)(d)"
    PUBLIC_TASK = "6(1)(e)"
    LEGITIMATE_INTERESTS = "6(1)(f)"
    NOT_APPLICABLE = "n/a"


class SpecialCategoryDerogation(str, Enum):
    EXPLICIT_CONSENT = "9(2)(a)"
    MANIFESTLY_PUBLIC = "9(2)(e)"
    SUBSTANTIAL_PUBLIC_INTEREST = "9(2)(g)"
    NOT_APPLICABLE = "n/a"


class CCPACategory(str, Enum):
    COVERED_BUSINESS_PURPOSE = "cpra-business-purpose"
    SALE_OR_SHARE = "cpra-sale-or-share"
    NOT_APPLICABLE = "n/a"


@dataclass(frozen=True)
class SourcePolicy:
    """Per-source policy; this is where the memo meets the code."""
    source_id: str
    gdpr_basis: LawfulBasisGDPR
    special_category_derogation: SpecialCategoryDerogation
    ccpa_category: CCPACategory
    special_category_data_allowed: bool
    requires_pii_scrub: bool
    honours_opt_out: bool  # True iff the acquisition pipeline runs the
                           # opt-out suppression list against this source.
    vendor_attested: bool  # True iff a vendor-sourced dataset has a signed
                           # lawful-basis attestation on file.


@dataclass(frozen=True)
class Record:
    """A candidate training record, already classified upstream."""
    record_id: str
    source_id: str
    contains_pii: bool
    contains_special_category: bool
    opt_out_flag: bool               # subject or publisher opt-out detected
    erasure_requested: bool          # standing erasure request matches
    jurisdiction_hint: str           # best guess at subject jurisdiction


@dataclass(frozen=True)
class AdmitDecision:
    admit: bool
    reason: str
    obligations: Sequence[str] = field(default_factory=tuple)


def admit(record: Record, policy: SourcePolicy) -> AdmitDecision:
    """Decide whether `record` may enter the training corpus."""
    # 1. Standing erasure request is absolute — always reject.
    if record.erasure_requested:
        return AdmitDecision(
            admit=False,
            reason="erasure-request:standing",
            obligations=("log:erasure-honoured",),
        )

    # 2. Subject or publisher opt-out — reject unless the source policy
    #    explicitly waives (it should not for personal data).
    if record.opt_out_flag:
        return AdmitDecision(
            admit=False,
            reason="opt-out:subject-or-publisher",
            obligations=("log:opt-out-honoured",),
        )

    # 3. Vendor data must carry a signed attestation.
    if (
        policy.source_id.startswith("vendor:")
        and not policy.vendor_attested
    ):
        return AdmitDecision(
            admit=False,
            reason="vendor:no-lawful-basis-attestation",
        )

    # 4. GDPR lawful-basis gate.
    if (
        record.contains_pii
        and policy.gdpr_basis is LawfulBasisGDPR.NOT_APPLICABLE
    ):
        return AdmitDecision(
            admit=False,
            reason="gdpr:no-lawful-basis-for-pii",
        )

    # 5. Special-category data: policy must permit AND a derogation
    #    must be named.
    if record.contains_special_category:
        if not policy.special_category_data_allowed:
            return AdmitDecision(
                admit=False,
                reason="gdpr:special-category-not-permitted",
            )
        if (
            policy.special_category_derogation
            is SpecialCategoryDerogation.NOT_APPLICABLE
        ):
            return AdmitDecision(
                admit=False,
                reason="gdpr:no-art9-derogation",
            )

    # 6. CCPA/CPRA: refuse records that would require sale/share treatment
    #    unless the source policy claims that category explicitly.
    if (
        record.jurisdiction_hint == "US-CA"
        and policy.ccpa_category is CCPACategory.NOT_APPLICABLE
        and record.contains_pii
    ):
        return AdmitDecision(
            admit=False,
            reason="ccpa:pii-without-category",
        )

    # 7. Opt-out suppression must be active on the source.
    if record.contains_pii and not policy.honours_opt_out:
        return AdmitDecision(
            admit=False,
            reason="policy:opt-out-suppression-not-active",
        )

    # 8. PII admitted only with scrubbing.
    obligations: list[str] = []
    if record.contains_pii and policy.requires_pii_scrub:
        obligations.append("transform:pii-scrub")

    obligations.append(f"attest:gdpr-basis={policy.gdpr_basis.value}")
    if record.contains_special_category:
        obligations.append(
            f"attest:gdpr-art9={policy.special_category_derogation.value}"
        )
    if record.jurisdiction_hint == "US-CA":
        obligations.append(
            f"attest:ccpa-category={policy.ccpa_category.value}"
        )

    return AdmitDecision(
        admit=True,
        reason="admit:pass",
        obligations=tuple(obligations),
    )


SOURCE_POLICIES: Mapping[str, SourcePolicy] = {
    "internal:tickets": SourcePolicy(
        source_id="internal:tickets",
        gdpr_basis=LawfulBasisGDPR.LEGITIMATE_INTERESTS,
        special_category_derogation=SpecialCategoryDerogation.NOT_APPLICABLE,
        ccpa_category=CCPACategory.COVERED_BUSINESS_PURPOSE,
        special_category_data_allowed=False,
        requires_pii_scrub=True,
        honours_opt_out=True,
        vendor_attested=False,
    ),
    "web:crawl": SourcePolicy(
        source_id="web:crawl",
        gdpr_basis=LawfulBasisGDPR.LEGITIMATE_INTERESTS,
        special_category_derogation=SpecialCategoryDerogation.NOT_APPLICABLE,
        ccpa_category=CCPACategory.NOT_APPLICABLE,
        special_category_data_allowed=False,
        requires_pii_scrub=True,
        honours_opt_out=True,
        vendor_attested=False,
    ),
    "vendor:conversations": SourcePolicy(
        source_id="vendor:conversations",
        gdpr_basis=LawfulBasisGDPR.CONSENT,
        special_category_derogation=SpecialCategoryDerogation.NOT_APPLICABLE,
        ccpa_category=CCPACategory.COVERED_BUSINESS_PURPOSE,
        special_category_data_allowed=False,
        requires_pii_scrub=True,
        honours_opt_out=True,
        vendor_attested=True,
    ),
    "synthetic:self": SourcePolicy(
        source_id="synthetic:self",
        gdpr_basis=LawfulBasisGDPR.NOT_APPLICABLE,
        special_category_derogation=SpecialCategoryDerogation.NOT_APPLICABLE,
        ccpa_category=CCPACategory.NOT_APPLICABLE,
        special_category_data_allowed=False,
        requires_pii_scrub=False,
        honours_opt_out=True,
        vendor_attested=False,
    ),
}
```

### 2.3 Worked tests (property-level, no framework needed)

```python
# admit_tests.py
"""Illustrative tests — these are docstring-level assertions a reviewer can
read, not a test-framework harness. Wire them into pytest if desired."""

from dataclasses import replace

from admit import (
    AdmitDecision, CCPACategory, LawfulBasisGDPR, Record, SOURCE_POLICIES,
    SpecialCategoryDerogation, admit,
)


def _record(**kwargs) -> Record:
    defaults = dict(
        record_id="r1",
        source_id="internal:tickets",
        contains_pii=False,
        contains_special_category=False,
        opt_out_flag=False,
        erasure_requested=False,
        jurisdiction_hint="EU",
    )
    defaults.update(kwargs)
    return Record(**defaults)


def test_erasure_always_wins():
    r = _record(contains_pii=True, erasure_requested=True)
    p = SOURCE_POLICIES["internal:tickets"]
    assert admit(r, p).admit is False
    assert admit(r, p).reason == "erasure-request:standing"


def test_opt_out_rejects():
    r = _record(opt_out_flag=True)
    p = SOURCE_POLICIES["web:crawl"]
    assert admit(r, p).admit is False
    assert admit(r, p).reason.startswith("opt-out:")


def test_pii_requires_scrub_obligation():
    r = _record(contains_pii=True)
    p = SOURCE_POLICIES["internal:tickets"]
    d = admit(r, p)
    assert d.admit is True
    assert "transform:pii-scrub" in d.obligations


def test_special_category_requires_derogation():
    r = _record(contains_pii=True, contains_special_category=True)
    p = SOURCE_POLICIES["internal:tickets"]
    assert admit(r, p).admit is False


def test_vendor_without_attestation_rejected():
    r = _record(source_id="vendor:conversations", contains_pii=True)
    p = SOURCE_POLICIES["vendor:conversations"]
    p_unattested = replace(p, vendor_attested=False)
    assert admit(r, p_unattested).admit is False


def test_us_ca_pii_requires_category():
    r = _record(contains_pii=True, jurisdiction_hint="US-CA")
    p = SOURCE_POLICIES["web:crawl"]  # ccpa_category = NOT_APPLICABLE
    assert admit(r, p).admit is False
    assert admit(r, p).reason == "ccpa:pii-without-category"
```

### 2.4 Operational notes

- The admit function is called at *ingest*, not just at release — because an
  opt-out that arrives between ingest and release should also take effect at
  release (see the erasure-sweep hook in Exercise 01's Gap list).
- A rebuild of the training corpus must re-run `admit` against the current
  policies and erasure list. The function is idempotent by design.
- The function does not interpret the statute. It expresses the memo.

## 3. Validation steps

1. **The memo precedes the code.** A reviewer asks for the memo first; the
   code without the memo is not the deliverable.
2. **Every source has a named GDPR Article 6 basis** — even if it is
   `NOT_APPLICABLE` for synthetic data, that choice is defended.
3. **Special-category data is explicitly addressed** — the memo must state
   whether it is allowed per source.
4. **The code's branches match the memo's rows.** Walk the admit function
   top-to-bottom while reading the memo; every reject reason in the code
   corresponds to a constraint in the memo.
5. **Erasure requests are an absolute veto in code.** This is the GDPR
   Article 17 muscle; mis-ranking its branch is a severe failure.
6. **Opt-out honouring is a precondition for admission of PII**, not a
   nice-to-have.
7. **CCPA/CPRA is handled when any US-CA subject is plausibly represented.**
   "We only operate in the EU" must be defensible before dropping CCPA.
8. **Tests exercise each reject reason.** A function with an untested
   rejection branch is a function that will silently drift.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 18/24 and loses zero points on the rows marked
**critical**.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | Lawful-basis memo names the Article 6(1) basis for every source                                                 | Yes       |
| 2  | Legitimate-interests choice is accompanied by a documented balancing test                                       | Yes       |
| 3  | Special-category (Article 9) handling is explicit per source                                                    | Yes       |
| 4  | CCPA/CPRA posture is addressed per source (notice at collection; sale/share; ADMT opt-out)                      | Yes       |
| 5  | `admit` function implements an absolute erasure veto                                                            | Yes       |
| 6  | Opt-out suppression is checked before PII admission                                                             | Yes       |
| 7  | Vendor-sourced data is rejected absent a signed lawful-basis attestation                                        | Yes       |
| 8  | Code is dependency-free, pure, and auditable on a single screen per source                                      | No        |
| 9  | Tests cover each distinct reject reason                                                                         | Yes       |
| 10 | Admit obligations flow downstream (`transform:pii-scrub`, `attest:...`) and are enforced in the pipeline        | No        |
| 11 | Function is idempotent on re-run against updated erasure / opt-out lists                                        | Yes       |
| 12 | Open follow-up notes are used anywhere the memo awaits legal sign-off                                           | No        |

## 5. Common mistakes

- **Legitimate-interests-for-everything.** LI is a legitimate basis, but it
  requires a balancing test. A memo that claims LI for every source without
  writing the balancing test for each has not done the work.
- **Treating consent as always available.** Consent is often inappropriate
  for crawled data (no subject-level consent channel). Consent that cannot
  be withdrawn is not consent.
- **Ignoring Article 9.** Special-category data (health, sexual orientation,
  union membership, biometrics, race / ethnicity) requires a derogation.
  Silence is not an answer.
- **Treating CCPA as "GDPR-lite".** CCPA / CPRA has its own structure
  (notice at collection, categories of personal information, sale/share,
  profiling / ADMT opt-outs); it is not GDPR with a different name.
- **Collapsing opt-out and erasure.** Opt-out blocks future use; erasure
  removes existing use. The function treats them differently.
- **Letting vendor warranties float.** Vendor data without a signed
  attestation gives you no lawful-basis chain; reject it until the
  attestation is on file.
- **Skipping the test for each reject reason.** Tests are the audit trail
  that proves the function matches the memo; skipping them makes the memo
  decorative.
- **Trying to make the admit function "interpret" the law.** The function
  expresses the organisation's documented policy. Statutory interpretation
  belongs in the memo and with counsel.

## 6. References

- Regulation (EU) 2016/679 (GDPR) — Articles 6 (lawfulness of processing),
  7 (conditions for consent), 9 (special categories), 14 (information when
  data not obtained from the subject), 17 (right to erasure), and 22
  (automated individual decision-making). Official text via EUR-Lex.
- California Consumer Privacy Act, as amended by the CPRA — notice at
  collection, categories of personal information, "sale" and "sharing"
  definitions, and ADMT provisions. California Civil Code §§ 1798.100 et
  seq.; implementing regulations issued by the California Privacy
  Protection Agency.
- Regulation (EU) 2024/1689 (the AI Act) — Article 10(5) permits
  processing of special categories of personal data for bias detection
  under strict safeguards; relevant when the admit function must permit
  SCD for a bias-audit split (see Exercise 03).
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (For cross-walking lawful-basis obligations to MANAGE 2.2 ongoing
  risk-mitigation and MAP 4.1 data appropriateness.)
- NIST AI 600-1 — *Generative AI Profile* —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
  (Data-privacy risk anchor.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (Privacy-violation and data-leak risks map to the admit function's
  reject branches.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Data-exfiltration techniques are the adversarial lens on records that
  slip through.)
- VeriSwarm Trust Center — <https://veriswarm.ai/trust>
  (Practitioner reference: example of how a vendor presents a lawful-basis
  attestation. Use only as a format example, not as a source of policy.)
- Local anchors: Exercise 01 (NIST AI RMF map — MANAGE 2.2 anchor),
  Exercise 02 (ISO walk — data-privacy control rows), Exercise 03 (AI Act
  Article 10 and Article 10(5)).
