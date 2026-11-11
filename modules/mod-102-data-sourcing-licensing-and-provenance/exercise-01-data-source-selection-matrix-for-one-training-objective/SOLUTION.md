# Exercise 01 — Data-Source Selection Matrix for One Training Objective (Solution)

## 1. Solution overview

This is a **design** exercise. The learner picks a single training objective, enumerates
the candidate data sources that could plausibly feed it, and defends the final
selection against a fixed rubric before any acquisition happens.

A good solution produces three artifacts:

1. A one-paragraph **training-objective statement** that is narrow enough to score
   candidate sources against (not "make a better LLM", but "fine-tune a 7B chat model
   to answer Python packaging questions in English").
2. A **selection matrix** whose rows are candidate sources and whose columns are the
   decision criteria (licence, consent/opt-out posture, provenance strength,
   representativeness/coverage, data-quality signal, privacy/PII risk, security
   risk, cost, operability).
3. A **selection memo** that names the chosen sources, justifies rejections, and
   maps each criterion to an upstream governance obligation (NIST AI RMF function,
   OWASP ML Top 10 risk, or MITRE ATLAS technique).

The point of the exercise is not to memorise a scoring formula; it is to show that
the learner can refuse a source on a *governance* ground, not only on quality.

## 2. Implementation — worked answer

### 2.1 Training-objective statement (worked example)

> **Objective.** Supervised fine-tune a 7B open-weights base model to answer
> English-language questions about Python packaging (pip, PyPI, wheel, pyproject)
> for internal developer support. Expected deployment: an internal chat tool used
> by ~200 engineers. Not customer-facing. No code generation beyond ≤30-line
> snippets. No PII in prompts. Target: v1 release within one quarter.

Why this shape matters:

- The **population** is defined (English-language Python packaging questions), so
  representativeness can be judged.
- The **deployment context** is defined (internal, bounded audience), which caps
  the risk tier under the EU AI Act framing covered in mod-101.
- The **output class** is bounded (short snippets, no PII), which lets the opt-out
  and PII columns be scored meaningfully.

A training objective that cannot answer "who uses the output, for what, with what
blast radius?" is not yet ready for a selection matrix.

### 2.2 Candidate data sources

For the worked objective, six plausible candidates:

| # | Candidate source                                                       | Short description                                              |
|---|------------------------------------------------------------------------|----------------------------------------------------------------|
| A | Internal support-ticket transcripts (last 24 months)                   | First-party, ~40k Q&A pairs, mixed quality                     |
| B | Public Python packaging docs (packaging.python.org, pip, PyPA guides)  | First-party upstream docs, permissively licenced               |
| C | Stack Overflow Q&A tagged `python`, `pip`, `pyproject`, `packaging`    | Third-party, mixed licences, user-contributed                  |
| D | GitHub issues and PR discussions from packaging-adjacent repos         | Third-party, per-repo licensing, high noise                    |
| E | A broad web-scrape of Python tutorials from random blogs               | Third-party, unknown licensing, unknown consent                |
| F | Synthetic Q&A generated from (A)+(B) with a commercial frontier model  | Derivative; licence depends on the generator's terms-of-use    |

### 2.3 Selection matrix

Scoring key: **✓ accept** / **~ accept with mitigation** / **✗ reject**. Each cell
is defended in the rationale that follows.

| Criterion                        | A (tickets) | B (PyPA docs) | C (Stack Overflow) | D (GH issues) | E (random blogs) | F (synthetic) |
|----------------------------------|-------------|---------------|--------------------|---------------|------------------|---------------|
| Licence clarity                  | ✓           | ✓             | ~                  | ~             | ✗                | ~             |
| Consent / opt-out posture        | ~           | ✓             | ~                  | ~             | ✗                | ✓             |
| Provenance strength              | ✓           | ✓             | ✓                  | ✓             | ✗                | ✓             |
| Representativeness vs objective  | ✓           | ✓             | ✓                  | ~             | ~                | ~             |
| Quality signal                   | ~           | ✓             | ~                  | ✗             | ✗                | ~             |
| PII / privacy risk               | ~           | ✓             | ✓                  | ✓             | ✓                | ✓             |
| Security risk (poisoning)        | ✓           | ✓             | ~                  | ~             | ✗                | ~             |
| Cost / operability               | ✓           | ✓             | ~                  | ~             | ✓                | ~             |
| **Decision**                     | **✓**       | **✓**         | **~**              | **✗**         | **✗**            | **~**         |

### 2.4 Rationale — per source

**A. Internal support tickets (accept, with mitigations).**
First-party data. Licence and consent are governed by the internal employment /
tool-use agreement — those must be re-read before use (the tool's privacy notice
must state that aggregated content can be used for internal model training; if it
does not, this column drops to ✗). Mitigations: PII-redact before training;
exclude tickets that quote customer logs. Maps to NIST AI RMF **MAP 4.1**
(contextual appropriateness of data use).

**B. Public Python packaging docs (accept).**
The PyPA documentation set is published under permissive terms (the individual
projects declare their own licences; see each repository's `LICENSE` file). High
provenance strength (one upstream org per sub-project). Representative of the
target population by construction. No PII expected. Low poisoning risk because
changes are reviewed in public on GitHub.

**C. Stack Overflow Q&A (accept with mitigation).**
Content is contributed by the community under a Creative Commons ShareAlike
licence per Stack Exchange's long-standing terms of service.
Mitigations: attribution file must be emitted (ShareAlike implies downstream
licence propagation); honour per-post deletions by refreshing periodically;
drop posts whose authors have exercised an opt-out. Poisoning risk exists via
answer-manipulation; mitigate by requiring a minimum score and age threshold.

**D. GitHub issues / PR discussions (reject).**
Per-comment licensing is ambiguous unless the repository's `CONTRIBUTING` file or
the GitHub Terms of Service grant is explicitly relied on. Signal-to-noise is
poor (bug reports, off-topic chatter). Retaining this source would force a
per-repository licence audit that the quarter's budget cannot afford. Revisit
after a licence-classification pipeline is built.

**E. Random blog scrape (reject).**
No licence signal, no reliable opt-out posture (very few personal blogs publish
`ai.txt` or honour `noai`), weak provenance, high poisoning / spam risk. There is
no mitigation inside the project's budget that lowers this below "reject".
Blocking rejection — this one is where the rubric is most likely to catch a
learner that defaulted to "scrape it all".

**F. Synthetic data from a commercial frontier model (accept with mitigation).**
The licence question is about the **generator's output terms**, not about the
synthetic corpus itself. Mitigations: freeze the generator version; store the
prompts; preserve the generator's commercial-use terms in the data card so the
downstream deployment-rights question is answerable. Add an explicit audit for
memorised third-party content leaking through the generator (see mod-108).

### 2.5 Selection memo (worked example)

> **Selected sources, v1:** A (tickets, PII-scrubbed), B (PyPA docs), C (Stack
> Overflow, filtered + attribution file).
> **Deferred:** F (synthetic) — re-evaluate once mod-108 synthetic-pipeline
> controls are in place.
> **Rejected:** D (GitHub discussions), E (random blogs).
> **Primary governance anchors:** NIST AI RMF `MAP 4.1`, `GOVERN 5.1`, `MANAGE 2.2`.
> OWASP ML Top 10 relevance: **ML02 Data Poisoning Attack** (why E is rejected).
> MITRE ATLAS relevance: the poisoning rationale maps to the ATLAS "Poison
> Training Data" tactic/technique family; the exact technique ID should be read
> off the ATLAS matrix at acquisition time.

## 3. Validation steps

1. **Objective is testable.** Can you name a held-out question the model must
   answer, and a question it must refuse? If not, the objective is still too
   broad to score sources against.
2. **Every candidate has a non-empty rationale in every column.** A blank cell
   means the learner hasn't actually checked that criterion.
3. **At least one source is rejected on a governance-only ground** (licence or
   consent), not only on quality. Selection matrices that reject only on quality
   have not exercised the governance muscle this module is about.
4. **The memo names the standard.** Each accept / reject decision links to at
   least one anchor in NIST AI RMF, OWASP ML Top 10, or MITRE ATLAS — not vibes.
5. **PII column is evaluated even for first-party data.** A common failure is to
   write "✓, first-party" for internal data without having checked whether the
   privacy notice covers model training (mod-101, Exercise 04 territory).
6. **Deferred ≠ rejected.** The memo distinguishes sources that are blocked by
   this quarter's budget from sources that are blocked on principle. The two
   require different follow-ups.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 16/20 and loses zero points on the rows marked
**critical**.

| # | Dimension                                                                                        | Critical? |
|---|--------------------------------------------------------------------------------------------------|-----------|
| 1 | Training-objective statement is narrow enough to score against (population, deployment, blast)   | Yes       |
| 2 | At least four candidate sources considered, including at least one that must be rejected        | Yes       |
| 3 | Matrix has distinct columns for licence, consent/opt-out, provenance, and PII (not merged)       | Yes       |
| 4 | Every ✓/~/✗ in the matrix has a one-sentence rationale beneath it                                | Yes       |
| 5 | At least one rejection is on a governance-only ground, not quality                               | Yes       |
| 6 | Mitigations are concrete (what, by when, verifiable) for every ~ cell                             | No        |
| 7 | Selection memo links decisions to NIST AI RMF, OWASP ML Top 10, or MITRE ATLAS                   | Yes       |
| 8 | Synthetic / generator-sourced data treats licence as a function of the generator's terms         | No        |
| 9 | Open follow-up notes are used anywhere a licence or ToS has not been freshly checked              | No        |
| 10 | "Deferred" and "rejected" are not confused                                                       | No        |

## 5. Common mistakes

- **Treating "we already have it" as a selection reason.** Internal data is
  first-party but is not automatically consented for model training. Re-read the
  privacy notice before the ✓ goes on the sheet.
- **Merging licence and consent into one column.** They fail independently — a
  CC-BY-licenced blog can still host a `noai` directive. See Exercise 03.
- **Rejecting sources only on quality.** Quality-only matrices skip the exercise.
- **Promising a specific SPDX ID without checking the repository.** Licence
  strings drift; verify at acquisition time and leave an open follow-up note
  until you have.
- **Scoring synthetic data as "no licence question".** The generator's terms are
  the licence question; mod-108 covers this in depth.
- **No deferred row.** Real selection work produces a "revisit next quarter" pile
  — if your memo doesn't, you probably over-accepted or over-rejected.
- **Collapsing MITRE ATLAS and OWASP ML Top 10 into one citation.** They answer
  different questions (attacker techniques vs. ranked risk categories); keep
  them separate so reviewers can tell which lens caught the issue.

## 6. References

- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (functions GOVERN / MAP / MEASURE / MANAGE; see MAP 4.1 for data-source
  appropriateness.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (ML02 Data Poisoning Attack is the primary anchor for rejecting unverified
  scrapes.)
- MITRE ATLAS —
  <https://atlas.mitre.org/>
  (Browse the matrix for the current technique IDs under the training-data
  poisoning tactic before citing a specific ID in a memo.)
- Local anchor: `mod-101` governance module for lawful-basis and risk-tier
  framing; `mod-108` for synthetic-data generator terms.
