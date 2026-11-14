# Exercise 02 — ISO/IEC 25012 and 5259-2 Characteristic Coverage Check (Solution)

## 1. Solution overview

This is a **design** exercise. The learner takes the quality bar they wrote
as a GX suite in Exercise 01 (or any equivalent executable or prose quality
bar) and maps it, characteristic by characteristic, against:

- **ISO/IEC 25012:2008** — Data Quality Model (15 quality characteristics
  partitioned into *inherent*, *system-dependent*, and the overlap).
- **ISO/IEC 5259-2:2024** — Data quality for analytics and ML: data quality
  measures (training/validation/test and inference data).

Deliverable: a **coverage matrix** — one row per characteristic, with
columns for current coverage (covered / partial / uncovered / N/A), the
evidence (which Expectation, metric, or document provides coverage), and
the gap statement if coverage is partial or missing. The matrix must also
state which 25012 characteristics are *out of scope* for this slice and
why, so a reviewer can distinguish "forgotten" from "deliberately
unscored".

Why this exercise matters: GX (or any validation tool) only enforces
dimensions you remembered to encode. The published standards exist so you
can audit your own quality bar against a comprehensive reference set, and
pick up the characteristics you forgot — or decide, in writing, that they
don't apply.

The two standards do not have a 1:1 mapping. 5259-2 is explicitly ML-aware
and introduces characteristics (and quality-measure families) that 25012
does not. The solution handles both: map 25012 first because it is more
granular on general data quality, then overlay 5259-2's ML-specific
characteristics.

Terminology caveat: the ISO/IEC 5259-2:2024 title, publication year, and
the exact name of each ML-specific characteristic family must be
re-verified against the current standard text (or an ISO-authoritative
summary) before publishing. The mapping below uses the conventional names
in the public preview; swap in the authoritative wording from the standard
when available.

Governance framing: this matrix is what an auditor reads to see whether
your quality controls are *complete* against a recognized framework. OWASP
ML Top 10 **ML02** and MITRE ATLAS' poison-training-data techniques put
pressure on *accuracy*, *credibility*, *currentness*, and *traceability*
specifically; NIST AI RMF `MEASURE 2.3` ("AI system performance or
assurance criteria are measured qualitatively or quantitatively") expects
the mapping to *exist*, not a specific result.

## 2. Implementation — the coverage matrix

### 2.1 ISO/IEC 25012 characteristics (15, grouped)

**Inherent** (properties of the data itself):

| # | Characteristic   | Covered? | Evidence (Exercise 01 suite)                                              | Gap / Note |
|---|------------------|----------|---------------------------------------------------------------------------|------------|
| 1 | Accuracy         | Partial  | `language == 'en'` + `language_score ≥ 0.5` catches wrong-language        | Does not check factual accuracy of text; out of scope at corpus scale |
| 2 | Completeness     | Covered  | `expect_column_values_to_not_be_null(mostly=1.0)` on all required columns | — |
| 3 | Consistency      | Partial  | Type Expectations enforce schema consistency                              | No cross-field consistency check (e.g. url host vs warc filename) |
| 4 | Credibility      | Partial  | `url` regex + `evidence_sha256` from mod-103 acquire stage                | No source-reputation signal; see Dodge audit in Exercise 03 |
| 5 | Currentness      | Partial  | Snapshot ID pinned in `slice_pin.yaml`                                    | No staleness Expectation (slice may be months old) |

**System-dependent** (properties of the data-in-system):

| #  | Characteristic   | Covered? | Evidence                                                                 | Gap / Note |
|----|------------------|----------|--------------------------------------------------------------------------|------------|
| 6  | Availability     | Partial  | Shard path + SHA in `slice_pin.yaml`; Makefile re-fetches                | No SLO; no uptime measure for the host |
| 7  | Portability      | Covered  | JSONL.gz + parquet manifest; no bespoke binary format                     | — |
| 8  | Recoverability   | Uncovered| —                                                                        | Delegated to the dataset-versioning module (mod-109) |

**Overlap** (both inherent and system-dependent):

| #  | Characteristic     | Covered? | Evidence                                                                     | Gap / Note |
|----|--------------------|----------|------------------------------------------------------------------------------|------------|
| 9  | Accessibility      | N/A      | —                                                                            | Internal corpus; no end-user accessibility surface |
| 10 | Compliance         | Partial  | mod-103 acquire-stage records the robots.txt opt-out evidence                | Licensing/provenance compliance lives in mod-102 |
| 11 | Confidentiality    | Uncovered| —                                                                            | PII scan is mod-110; call out the scope boundary |
| 12 | Efficiency         | Partial  | JSONL.gz + parquet manifest                                                  | No column-level efficiency check (e.g. compression ratio bounds) |
| 13 | Precision          | Covered  | `language_score` has defined numeric precision; Expectation bounds it in [0.5, 1.0] | — |
| 14 | Traceability       | Covered  | Every record has `warc_record_id`, `url`, host evidence SHA, snapshot id    | — |
| 15 | Understandability  | Partial  | `meta.rationale` per Expectation; slice README                               | No data dictionary; add one with units + semantics |

### 2.2 ISO/IEC 5259-2 ML-specific overlays

ISO/IEC 5259-2 extends the 25012 characteristic set with measures that
matter for *training* and *inference* data specifically. Even where
nomenclature overlaps, the scoring is different (e.g. "accuracy" in a
training-data sense asks about label fidelity, not record-level correctness).

Terminology caveat: each row below pairs a 5259-2 characteristic family
with an Exercise 01 evidence pointer. Validate the exact family names and
the 5259-2 subclause numbers against the standard text; rename to match if
the published terminology differs.

| Characteristic family (5259-2)       | Covered? | Evidence (Exercise 01 + sibling modules)                                   | Gap / Note |
|--------------------------------------|----------|-----------------------------------------------------------------------------|------------|
| Representativeness                   | Uncovered| —                                                                           | Audited in Exercise 04 (subpopulation / distribution); call out here |
| Balance                              | Uncovered| —                                                                           | Same; Exercise 04 |
| Diversity (source / content)         | Partial  | URL host regex, language share envelope                                     | No per-domain diversity target; Dodge-style audit in Exercise 03 |
| Relevance                            | Partial  | Language filter + source predicate                                          | No task-relevance check (no downstream task defined here) |
| Correctness of annotation / labels   | N/A      | —                                                                           | Slice is unlabeled pretraining data |
| Freshness / currency                 | Partial  | Snapshot ID                                                                 | Same staleness gap as 25012 §5 |
| Confidentiality / PII risk           | Uncovered| —                                                                           | mod-110 |
| Lineage / provenance                 | Covered  | mod-103 slice manifest + acquire evidence + Exercise 01 `slice_pin.yaml`    | — |
| Reproducibility                      | Covered  | Pinned predicate SHA, model SHA, snapshot ID, `make reproduce`              | — |
| Bias (statistical / sampling)        | Uncovered| —                                                                           | Exercise 04 |

### 2.3 Decision rationale — why each coverage verdict

For each characteristic the matrix must answer three questions:

1. **Is it in scope for this slice?** Confidentiality is in scope for a
   user-content corpus; Accessibility is not for an internal corpus.
2. **If in scope, is it covered?** Covered = an executable Expectation,
   metric, or document directly measures it. Partial = one axis of the
   characteristic is measured but others are not. Uncovered = no
   measurement.
3. **If partial or uncovered, where is the gap closed?** Either name
   another module (mod-109 for recoverability, mod-110 for
   confidentiality) or write the characteristic down as a known deferred
   control with a date to revisit.

**N/A is a legitimate verdict**, but it must be justified in one sentence.
An unexamined N/A is scored the same as "Uncovered".

### 2.4 Example: writing up "Accuracy" (25012 §1)

> **Characteristic:** Accuracy (inherent).
>
> **Scope:** In scope. The slice feeds pretraining; mislabelled language
> means wrong-script text slips into English-only shards and distorts
> downstream perplexity filtering.
>
> **Coverage:** Partial. Exercise 01's suite enforces `language == 'en'`
> and `language_score ≥ 0.5`, which measures *syntactic* accuracy of the
> language label. It does **not** measure factual accuracy of the text
> itself, and no commodity tool scales to factual accuracy at corpus size.
>
> **Gap:** Factual accuracy is out of scope by policy at the corpus-level
> and is pushed to the downstream task-level eval (not this module).
> Document the policy in the slice README so a reader does not read "97%
> language accuracy" as "97% factually correct".

A full matrix is 25 rows — 15 ISO/IEC 25012 characteristics plus the
~10 5259-2 overlays — each written to this template.

## 3. Validation steps

1. **Every 25012 characteristic has a row.** Count = 15, no silent omissions.
2. **Every 5259-2 characteristic family has a row.** Count matches the
   published list. Re-verify the count against the 5259-2 table of
   contents before publishing, since the public preview and the
   as-published standard can differ.
3. **Every row has an Evidence cell.** "Uncovered" with no gap statement is
   a review failure. A gap statement that only says "TODO" is also a failure.
4. **Each "Covered" cell points at a specific artefact.** A line number, a
   JSON path (`gx/expectations/....json → expectations[7]`), or a module
   reference (`mod-109 Exercise 02, §3`). "The suite checks this" with no
   locator is insufficient.
5. **Each "N/A" cell carries a one-sentence justification.** An
   unjustified N/A is a false negative on coverage.
6. **Scope boundaries are explicit.** Confidentiality (→ mod-110), dedup
   (→ mod-105), contamination (→ mod-106), representativeness
   (→ Exercise 04) are called out, not quietly dropped.
7. **The matrix is versioned with the slice.** Bumping the slice version
   requires re-running the matrix; a mismatch in version tags is a stale
   artefact.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 20/28 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                                  | Critical? |
|---|--------------------------------------------------------------------------------------------|-----------|
| 1 | All 15 ISO/IEC 25012 characteristics appear                                                | Yes       |
| 2 | All ISO/IEC 5259-2 ML-specific characteristic families appear                              | Yes       |
| 3 | Every row has Evidence or an explicit gap statement                                        | Yes       |
| 4 | Each Covered cell points at a concrete artefact (file + section, not prose)                | Yes       |
| 5 | Each N/A cell has a one-sentence justification                                             | Yes       |
| 6 | Scope-boundary characteristics explicitly delegate to a sibling module                     | Yes       |
| 7 | At least one characteristic has a worked rationale paragraph (not just a table cell)       | Yes       |
| 8 | Standard version is stated (25012:2008 and 5259-2:2024) with date                          | No        |
| 9 | Matrix is tied to a slice name + version, not left anonymous                               | Yes       |
| 10 | Unverified or paraphrased standard wording is flagged explicitly (prose caveat or `<!-- unverified: ... -->`) | Yes       |
| 11 | Governance anchor (OWASP ML Top 10 / NIST AI RMF) is cited and specific                    | No        |
| 12 | At least one "Partial" row names the axis that is NOT measured, not just "more work"       | Yes       |
| 13 | Matrix is checked into the repo alongside the suite, not emailed around                    | No        |
| 14 | Each row is editable in isolation (one characteristic per row; no double-booked cells)     | No        |

## 5. Common mistakes

- **Treating "we have a GX suite" as full coverage.** A suite only covers
  the characteristics whose Expectations you wrote. The matrix forces the
  honesty that those are a subset.
- **Paraphrasing the standard wording.** "Accuracy" in ISO 25012 is a
  defined term. If you rewrite it, you have invented a near-neighbour
  characteristic and your matrix no longer maps to the standard. Cite the
  standard clause number and quote on hover; paraphrase only in the
  rationale.
- **Marking confidentiality / PII "Covered" because the slice is "mostly
  English news".** Confidentiality means *a measurement*. If you don't
  have a PII scan, mark Uncovered and delegate to mod-110.
- **One-row-per-Expectation instead of one-row-per-characteristic.** The
  matrix is indexed by *characteristic*, not by Expectation; otherwise
  characteristics with zero Expectations disappear.
- **Collapsing 25012 and 5259-2 into a single table with no overlay.** The
  standards overlap but are not identical; some 5259-2 families (balance,
  representativeness) have no 25012 analogue, and conflating them hides
  gaps.
- **No slice version tag.** The quality bar is specific to a slice version;
  an undated matrix drifts out of sync the first time the slice is
  rebuilt.
- **Scoring characteristics the organization does not actually commit to.**
  "Recoverability: Covered — we trust the cloud provider" is a policy
  claim without evidence. Either demonstrate the recovery drill or mark
  Partial/Uncovered.

## 6. References

- **ISO/IEC 25012:2008 — Data Quality Model.** The 15-characteristic
  taxonomy this matrix is anchored on. The ISO catalogue page is the
  authoritative text; abstracts are freely available. Cite the ISO
  catalogue URL at write time rather than relying on this template.
- **ISO/IEC 5259-2:2024 — Data quality for analytics and ML: Data quality
  measures.** The ML-specific overlay in §2.2 comes from this standard.
  Verify characteristic family names against the standard text before
  publishing; abstracts are freely available, but the full standard is
  paywalled.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02
  (Data Poisoning Attack) is why characteristics like Credibility,
  Traceability, and Currentness get first-class rows.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. Browse the current matrix
  for technique IDs under "Poison Training Data" and "Supply Chain
  Compromise: ML Artifacts"; those are the characteristics the matrix is
  partly defending against.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MEASURE 2.3`
  grounds the expectation that measurement coverage *exists*.
- **Local anchors:** Exercise 01 (the suite this matrix audits);
  Exercise 03 (Dodge audit — fills parts of the "Credibility" and
  "Diversity" rows); Exercise 04 (Subpopulation + distribution — fills the
  "Representativeness" and "Balance" rows); mod-109 (versioning); mod-110
  (PII); mod-105 (dedup); mod-106 (contamination).
