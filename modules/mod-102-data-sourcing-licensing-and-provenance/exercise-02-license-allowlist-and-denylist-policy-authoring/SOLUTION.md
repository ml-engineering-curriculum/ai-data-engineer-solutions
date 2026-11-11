# Exercise 02 — Licence Allowlist and Denylist Policy Authoring (Solution)

## 1. Solution overview

This exercise produces a **written, enforceable licence policy** that an
acquisition pipeline can be configured against. The deliverable is not a list of
favourite licences — it is a document that:

1. Names the training objective and deployment context the policy serves (the
   same objective from Exercise 01).
2. Classifies licences into four buckets — **Allow**, **Conditional**, **Review
   required**, **Deny** — using SPDX identifiers as the canonical keys.
3. Specifies, for every Conditional / Review-required licence, the *operational
   obligation* that must be met before the data lands in the training corpus
   (attribution file, ShareAlike flag, no-commercial-use flag, non-derivative
   marker, etc.).
4. Specifies escalation and exception handling, so the pipeline knows when to
   defer to humans.
5. Is pinned to a review date and an owner.

A policy without owners, review cadence, and exception handling is a checklist,
not a policy, and will decay silently.

## 2. Implementation — worked answer

### 2.1 Front matter (policy header)

```yaml
# data-licence-policy.yaml
policy_name: "training-data-licence-policy"
version: "1.0.0"
effective_date: "YYYY-MM-DD"     # set at approval time
review_cadence: "quarterly"
next_review: "YYYY-MM-DD"
owner: "<data-governance-lead>"
scope:
  training_objective: "Internal Python-packaging assistant (see mod-102 ex-01)"
  deployment_context: "internal, ≤500 employees, not customer-facing"
  model_class: "fine-tune of open-weights base; weights not redistributed"
canonical_licence_vocabulary: "SPDX"      # https://spdx.org/licenses/
tie_breaker_rule: "if the file carries conflicting licence signals, the
  strictest applicable term controls until Review is cleared"
```

### 2.2 Bucket definitions

| Bucket            | Pipeline behaviour                                                           |
|-------------------|------------------------------------------------------------------------------|
| **Allow**         | Admitted automatically; attribution/propagation obligations still enforced.  |
| **Conditional**   | Admitted only if the per-licence obligation list below is satisfied.         |
| **Review required** | Routed to a human reviewer; blocks until a signed exception is attached.   |
| **Deny**          | Rejected at ingest; audit log captures the SPDX ID and source URL.           |

### 2.3 Worked allowlist (SPDX identifiers)

Each licence below is listed by its SPDX short identifier. The SPDX licence list
is the canonical registry; identifiers must be looked up at
<https://spdx.org/licenses/> at policy-authoring time to confirm spelling and
version pins.

**Allow (permissive, low obligation):**

| SPDX ID        | Why allowed                                                                                                           | Obligations enforced                                      |
|----------------|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| `CC0-1.0`      | Public-domain dedication; no attribution required.                                                                    | None.                                                     |
| `MIT`          | Permissive; permits broad use including derivative works.                                                             | Preserve copyright + licence notice in the data card.     |
| `BSD-2-Clause` | Permissive; equivalent to MIT in practice for text data.                                                              | Preserve copyright + licence notice.                       |
| `BSD-3-Clause` | Permissive + no-endorsement clause.                                                                                   | Preserve copyright + licence notice.                       |
| `Apache-2.0`   | Permissive + explicit patent grant; well-suited to corporate use.                                                     | Preserve NOTICE file; record licence in data card.        |
| `CC-BY-4.0`    | Permissive; requires attribution.                                                                                     | Emit attribution file keyed by source URL.                |

**Conditional (permissive with a propagation obligation):**

| SPDX ID            | Condition before admission                                                                                                                                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CC-BY-SA-4.0`     | Admit only if downstream outputs can carry a ShareAlike flag. If the model weights or outputs will be redistributed, legal must confirm ShareAlike propagation treatment for ML-trained artifacts before admission.               |
| `CC-BY-SA-3.0`     | Same as 4.0, plus: do not mix 3.0-SA and 4.0-SA content in the same partition unless legal approves (version-compatibility question).                                                                                               |
| `MPL-2.0`          | File-level copyleft; admissible for text data if the per-file notice is preserved in the attribution file.                                                                                                                         |
| `OFL-1.1`          | Open-Font Licence; admissible for *font metadata* but not for font binaries used as training input without a specific legal sign-off.                                                                                             |

**Review required (legal sign-off before admission):**

| SPDX ID               | Why review required                                                                                                              |
|-----------------------|----------------------------------------------------------------------------------------------------------------------------------|
| `GPL-2.0-only`        | Strong copyleft; ML-training interaction is unsettled.                                                                           |
| `GPL-2.0-or-later`    | Same as above.                                                                                                                   |
| `GPL-3.0-only`        | Strong copyleft; ML-training interaction is unsettled.                                                                           |
| `GPL-3.0-or-later`    | Same as above.                                                                                                                   |
| `LGPL-2.1-only`       | Lesser-copyleft; typically applied to linkable libraries, not corpora — review before treating as training text.                 |
| `AGPL-3.0-only`       | Network-use triggers source distribution; the model-serving boundary is unsettled.                                               |
| `AGPL-3.0-or-later`   | Same as AGPL-3.0-only.                                                                                                           |
| `EPL-2.0`             | File-level copyleft with secondary-licence option; needs a decision on how the secondary-licence choice is recorded.             |
| `CC-BY-NC-4.0`        | Non-commercial use only; admissible only if the deployment is non-commercial *and* legal confirms NC scope.                       |
| `CC-BY-NC-SA-4.0`     | Non-commercial + ShareAlike; same as NC plus propagation question.                                                               |

**Deny (no admission; log and move on):**

| SPDX ID                  | Why denied                                                                                   |
|--------------------------|----------------------------------------------------------------------------------------------|
| `CC-BY-ND-4.0`           | No-derivatives; training a model is almost certainly a derivative. Default deny.             |
| `CC-BY-NC-ND-4.0`        | No-derivatives + NC; default deny.                                                           |
| `SSPL-1.0`               | Server-side-public-licence; incompatible with internal-only deployment assumptions.          |
| `Commons-Clause`         | Not SPDX-recognised but widely encountered; forbids commercial sale; default deny. |
| `BUSL-1.1`               | Business Source Licence; use restrictions until change-date; default deny for training.      |
| `Elastic-2.0`            | Use restrictions on resale as a service; default deny for training without legal sign-off.   |
| `NOASSERTION`            | SPDX marker meaning the source did not declare a licence. Default deny (see §2.5).           |
| `NONE`                   | No licence at all. Default deny.                                                             |

**Non-SPDX / sui generis categories (always Deny unless an exception is signed):**

- Content with **no declared licence at all** → Deny. "Posted publicly" is not a
  licence grant.
- Content whose **terms of service** forbid machine-reading or training — route
  as Deny regardless of any machine-readable licence string.
- Content under a **custom "research use only"** licence — Deny for production
  training; route to Review for research prototypes.
- Content whose author has asserted `noai` / `noimageai` / equivalent (see
  Exercise 03) — Deny even if the file licence is permissive. The opt-out
  signal beats the licence signal.

### 2.4 Operational obligations table

Every ✓ in the allowlist costs the pipeline a side-effect. This table is what
makes the policy *enforceable* rather than aspirational.

| Obligation                       | Trigger licences               | Pipeline implementation                                                                   |
|----------------------------------|--------------------------------|-------------------------------------------------------------------------------------------|
| Attribution file emission        | `CC-BY-*`, `Apache-2.0`, …     | Per-record (source URL, author, licence SPDX) written to `attribution.jsonl` in the dataset artifact. |
| NOTICE file preservation         | `Apache-2.0`                   | Upstream NOTICE text copied verbatim into the data card.                                  |
| ShareAlike propagation flag      | `CC-BY-SA-*`                   | Dataset metadata flags `share_alike: true`; blocks downstream export without legal sign-off. |
| Non-commercial-use flag          | `CC-BY-NC-*`                   | Dataset metadata flags `non_commercial: true`; blocks model release channels that are commercial. |
| Per-file licence preservation    | `MPL-2.0`, `EPL-2.0`           | Per-file licence recorded at the row level, not just the dataset level.                   |
| Opt-out re-check                 | all                            | Re-fetch robots.txt / ai.txt / llms.txt / HTTP TDM Reservation before every full retrain. |

### 2.5 Precedence when signals disagree

In order, strictest first:

1. Explicit author opt-out signal (see Exercise 03) → **Deny**.
2. Terms-of-service prohibition on training → **Deny**.
3. Site-level machine-readable licence (e.g. `ai.txt`, `LICENSE`) → that licence
   controls.
4. File-level licence header → controls within that file only.
5. Default: **Deny** (SPDX `NOASSERTION`).

A learner who writes "default allow when unclear" has failed this section.

### 2.6 Exception handling

- Any Review-required licence requires a written exception that names the
  source, the SPDX ID, the deployment scope the exception covers, the expiry
  date, and the approving role (not person).
- Exceptions are recorded in-tree under `exceptions/<YYYY-MM-DD>-<slug>.md` so
  that `git log` is the audit trail.
- An exception never promotes a Deny licence; it only moves a Review-required
  licence to Conditional for the named deployment.

### 2.7 Review cadence

- Full policy review: quarterly.
- Emergency review triggered by: (a) a new widely-used licence appearing in the
  pipeline's ingest logs, (b) a court ruling that changes ShareAlike / NC / ND
  treatment for ML training, (c) a regulator issuing new guidance.
- Each review bumps `version:` under semver (**MAJOR** for bucket changes,
  **MINOR** for added SPDX IDs, **PATCH** for wording-only edits).

## 3. Validation steps

1. **Dry-run against a sample manifest.** Take the last 1000 ingested records,
   look up each record's declared licence, and confirm the policy sends every
   record to exactly one bucket (no ambiguity, no "it depends").
2. **Attribution file renders.** For every Allow / Conditional bucket row, the
   pipeline emits an attribution entry that the data card points to. Grep for
   zero empty-licence rows in the output.
3. **Opt-out precedence works.** Seed the sample with one record from a domain
   that publishes a `noai` directive. Confirm the record is **denied** even
   though its SPDX licence is permissive.
4. **Policy file lints.** `spdx-lookup` (or any SPDX-aware linter) accepts every
   identifier in the Allow / Conditional / Review / Deny tables.
5. **Exception trail is complete.** `git log exceptions/` shows every active
   exception with an expiry date in the future.
6. **Owner & review date are not empty.** A policy without owner + next-review
   is unassigned debt.

## 4. Rubric or review checklist

Score each row 0–2. Passing solution ≥ 20/26 and zero critical misses.

| # | Dimension                                                                                           | Critical? |
|---|-----------------------------------------------------------------------------------------------------|-----------|
| 1 | Policy header names scope (objective, deployment, model class), owner, cadence, next-review date    | Yes       |
| 2 | Four buckets exist: Allow / Conditional / Review / Deny                                              | Yes       |
| 3 | Canonical licence vocabulary is SPDX; every entry is an SPDX short identifier or explicitly marked non-SPDX | Yes |
| 4 | Every Conditional licence has a written obligation; Review-required licences have a written reason   | Yes       |
| 5 | Operational obligations table maps each licence family to a concrete pipeline side-effect           | Yes       |
| 6 | NOASSERTION / NONE / terms-of-service-prohibits-training default to Deny                            | Yes       |
| 7 | Opt-out signals override licence signals in the precedence rules                                     | Yes       |
| 8 | Exceptions have an expiry date, named approving role, and in-tree audit trail                       | No        |
| 9 | Review cadence + emergency-review triggers defined                                                   | No        |
| 10 | Open follow-up notes are used where an SPDX identifier status was not freshly verified             | No        |
| 11 | ShareAlike and Non-Commercial obligations propagate to a *dataset-level* flag, not just a note       | No        |
| 12 | Policy does not adopt any licence by name without an SPDX identifier (ambiguity blocker)             | Yes       |
| 13 | A sample dry-run against real ingest logs is included                                               | No        |

## 5. Common mistakes

- **Allowing GPL-family text content "because it's open source".** The
  copyleft-propagation question for ML training is unsettled; Review-required is
  the conservative default, not Allow.
- **Treating `CC-BY-ND` as permissive because it's Creative Commons.** The ND
  clause (no derivatives) is the whole point; training is a derivative. Deny.
- **Letting an undeclared licence default to Allow.** `NOASSERTION` means "we
  haven't checked" — the policy must treat that as Deny.
- **No attribution file.** `CC-BY-*` is in the allowlist only if the pipeline
  actually produces attribution; if it doesn't, those SPDX IDs belong in Review.
- **Confusing the ToS with the content licence.** A site can publish CC-BY text
  and still forbid scraping for training in its ToS. The ToS wins. See
  Exercise 03 for the signal precedence.
- **No ownership.** A policy with no named owner role and no review date decays
  into a dead file.
- **Collapsing BY and BY-SA.** They have different downstream obligations;
  merging them into a single row loses the ShareAlike propagation flag that the
  pipeline is supposed to set.
- **Treating "internal deployment" as "no licence obligation".** Internal use
  still has to honour attribution and ShareAlike terms; those travel with the
  data, not with the deployment boundary.

## 6. References

- SPDX Licence List — <https://spdx.org/licenses/>
  (Canonical vocabulary for licence identifiers; always look up the current
  entry before pinning one.)
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (GOVERN 1.4 and GOVERN 5.1 anchor the policy-ownership and documentation
  obligations this policy codifies.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (ML02 Data Poisoning Attack — unvetted-licence scrapes are a common entry
  point; the licence policy is a defence-in-depth control.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Browse for current technique IDs under the data-supply-chain tactic family
  before citing in exception memos.)
- Local anchors: Exercise 01 for the objective + deployment scoping, Exercise 03
  for opt-out signal precedence, mod-109 for how dataset-level flags such as
  `share_alike` and `non_commercial` propagate through lineage artifacts.
