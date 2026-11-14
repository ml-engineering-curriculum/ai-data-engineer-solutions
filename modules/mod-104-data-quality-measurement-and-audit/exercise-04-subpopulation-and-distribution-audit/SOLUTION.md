# Exercise 04 — Subpopulation and Distribution Audit (Solution)

## 1. Solution overview

This is an **implementation + analysis** exercise. The learner audits the
slice from mod-103 for **subpopulation coverage** and **distribution
shape** across axes they define (source TLD, PLD, language, document
length, topical cluster, time bucket, country-of-origin where defensible).
It is the complement to Exercise 03 (Dodge-style audit): Exercise 03 asks
*what's in there*, Exercise 04 asks *is the mix what we intended, and how
does it compare to a reference population*.

The deliverable is:

1. A **subpopulation table** — one row per subpopulation with (observed
   share, expected share, support count, representation ratio, 95% CI).
2. A **distribution comparison** — for each continuous axis (doc length,
   language score, time), the empirical CDF of the slice overlaid on a
   reference distribution, with a Kolmogorov–Smirnov statistic and
   Earth Mover's (Wasserstein) distance.
3. A **drift report** vs a prior slice version (if one exists) using the
   same subpopulation table shape, so release-over-release changes are
   visible.
4. A **limits-of-inference** section: this slice does not represent
   speakers / regions / ideologies 1:1; the audit tells you about the
   *corpus*, not the *world*.

The axes are the learner's choice. The design discipline is:

- Every axis must have a **reference distribution** the comparison is
  against (another slice, a published population reference, or a
  policy-set uniform target). "I compared against my gut" is not a
  comparison.
- Every subpopulation must have a **minimum support** (e.g. ≥ 500
  documents) below which the audit reports "insufficient support"
  instead of a ratio; small-N ratios are noise.
- Every finding of under- or over-representation must say what the
  **actionable consequence** is: re-weight at training, bump the
  acquire-stage seed list, document as known-limitation, or accept.

Governance framing: "representativeness" and "bias" are 5259-2
characteristics that Exercise 02's coverage matrix flags as uncovered.
This exercise is where they get covered. NIST AI RMF `MEASURE 2.11`
(fairness and bias examined and documented) and OWASP ML Top 10 **ML01**
(Input Manipulation) / **ML02** (Data Poisoning) are the governance
anchors — a lopsided subpopulation distribution is both a fairness
concern and an attack surface.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-04/
├── README.md
├── slice_pin.yaml
├── requirements.lock
├── Makefile
├── axes.yaml                  # the subpopulation axes under audit
├── reference/
│   └── reference_shares.csv   # expected shares with sources cited
├── code/
│   ├── 01_derive_axes.py
│   ├── 02_subpop_table.py
│   ├── 03_distribution_tests.py
│   ├── 04_drift_vs_prior.py
│   └── 05_render_report.py
├── audit/
│   ├── subpop_table.parquet
│   ├── distribution_tests.parquet
│   ├── drift.parquet
│   └── audit_manifest.json
└── report/
    └── subpop_distribution_audit.md
```

### 2.2 `axes.yaml` — pick the axes under audit

```yaml
# axes.yaml
# One block per axis. 'reference_source' is required; "none" is not an acceptable value.
axes:
  - name: tld
    kind: categorical
    derive: "host.rsplit('.', 1)[-1]"
    reference_source: "reference_shares.csv#tld"
    min_support: 500

  - name: pld_top100
    kind: categorical
    derive: "tldextract.extract(host).registered_domain"
    bucket: "top-100 by bytes; rest into 'other'"
    reference_source: "none-target-is-dispersion"   # policy axis, not a reference
    min_support: 200

  - name: language_score_binned
    kind: continuous_binned
    derive: "language_score rounded to 0.05"
    reference_source: "reference_shares.csv#language_score"
    min_support: 500

  - name: doc_length_tokens
    kind: continuous
    derive: "whitespace_tokens(text)"
    reference_source: "prior_slice:v0.9.0"
    min_support: 200

  - name: content_year
    kind: continuous_binned
    derive: "warc_date -> YYYY (warn: fetch time, lower bound)"
    reference_source: "prior_slice:v0.9.0"
    min_support: 200
```

Two things worth noting:

- `pld_top100` has `reference_source: none-target-is-dispersion`. That's a
  legitimate design — the policy is "no PLD exceeds X% of bytes", not
  "match a reference". The audit code checks dispersion instead of a
  comparison. Mark it in the YAML so the audit report can render the
  right view.
- `content_year` is explicitly a lower bound; the report must render that
  caveat next to the bins.

### 2.3 `code/02_subpop_table.py` — observed vs expected

```python
# code/02_subpop_table.py
# Build the subpop table with representation ratios and Wilson CIs.
# representation_ratio = observed_share / expected_share
# A ratio of 1.0 means on-target; < 0.8 or > 1.25 is the usual actionable band.
from __future__ import annotations

import math
from pathlib import Path

import polars as pl
import yaml

AXES = Path("axes.yaml")
IN = Path("../mod-103/manifest/per_record_manifest.parquet")
REF = Path("reference/reference_shares.csv")
OUT = Path("audit/subpop_table.parquet")


def wilson_ci(k: int, n: int, z: float = 1.96) -> tuple[float, float]:
    if n == 0:
        return (0.0, 0.0)
    p = k / n
    denom = 1 + (z * z) / n
    center = p + (z * z) / (2 * n)
    margin = z * math.sqrt((p * (1 - p) / n) + (z * z) / (4 * n * n))
    return ((center - margin) / denom, (center + margin) / denom)


def derive_values(df: pl.DataFrame, axis: dict) -> pl.DataFrame:
    # Real derivation lives in code/01_derive_axes.py; here we assume the
    # per-record manifest has already been extended with `axis.name` columns.
    return df


def main() -> None:
    cfg = yaml.safe_load(AXES.read_text())
    df = pl.read_parquet(IN).filter(pl.col("language") == "en")
    ref = pl.read_csv(REF)

    rows = []
    n_total = len(df)
    for axis in cfg["axes"]:
        name = axis["name"]
        min_support = int(axis["min_support"])
        counts = (df.group_by(name).agg(pl.len().alias("support")))
        for row in counts.iter_rows(named=True):
            value = row[name]
            k = int(row["support"])
            obs_share = k / n_total if n_total else 0.0
            lo, hi = wilson_ci(k, n_total)

            exp_share = None
            if axis.get("reference_source", "").startswith("reference_shares.csv"):
                key = axis["reference_source"].split("#", 1)[1]
                match = ref.filter(
                    (pl.col("axis") == key) & (pl.col("value").cast(pl.Utf8) == str(value))
                )
                if len(match):
                    exp_share = float(match[0, "expected_share"])

            rep_ratio = None
            if exp_share and exp_share > 0:
                rep_ratio = obs_share / exp_share

            rows.append({
                "axis": name,
                "value": str(value),
                "support": k,
                "observed_share": obs_share,
                "expected_share": exp_share,
                "representation_ratio": rep_ratio,
                "ci_low": lo,
                "ci_high": hi,
                "sufficient_support": k >= min_support,
            })
    pl.DataFrame(rows).write_parquet(OUT)
    print(f"[02] wrote {OUT} ({len(rows)} subpop rows)")


if __name__ == "__main__":
    main()
```

### 2.4 `code/03_distribution_tests.py` — KS + Wasserstein

```python
# code/03_distribution_tests.py
# For continuous axes, compare slice distribution to reference distribution
# via Kolmogorov-Smirnov statistic and Wasserstein (Earth Mover's) distance.
# These are complementary: KS is sensitive to any CDF deviation; Wasserstein
# captures *how much* mass has moved.
from __future__ import annotations

from pathlib import Path

import numpy as np
import polars as pl
from scipy import stats

IN = Path("../mod-103/manifest/per_record_manifest.parquet")
REF_SAMPLES = Path("reference/continuous_samples.parquet")  # shape: (axis, value)
OUT = Path("audit/distribution_tests.parquet")


def main() -> None:
    cur = pl.read_parquet(IN).filter(pl.col("language") == "en")
    cur = cur.with_columns([
        pl.col("text").str.split(" ").list.len().alias("doc_length_tokens"),
    ])
    ref = pl.read_parquet(REF_SAMPLES)

    rows = []
    for axis in ["doc_length_tokens", "language_score"]:
        a = cur[axis].drop_nulls().to_numpy()
        b = ref.filter(pl.col("axis") == axis)["value"].to_numpy()
        if len(a) < 500 or len(b) < 500:
            rows.append({"axis": axis, "ks_stat": None, "ks_p": None,
                         "wasserstein": None, "n_slice": int(len(a)),
                         "n_ref": int(len(b)), "note": "insufficient support"})
            continue
        ks = stats.ks_2samp(a, b, alternative="two-sided", method="auto")
        emd = stats.wasserstein_distance(a, b)
        rows.append({
            "axis": axis,
            "ks_stat": float(ks.statistic),
            "ks_p": float(ks.pvalue),
            "wasserstein": float(emd),
            "n_slice": int(len(a)),
            "n_ref": int(len(b)),
            "note": "",
        })
    pl.DataFrame(rows).write_parquet(OUT)


if __name__ == "__main__":
    main()
```

Two caveats the report must repeat:

- At large N the KS test is essentially always "significant"; the
  p-value is uninformative. Report the KS *statistic* (distance between
  CDFs) and let the reviewer judge magnitude.
- Wasserstein distance has units. `wasserstein_distance = 12.4` is
  meaningless without the axis unit. Report with units
  ("tokens", "score points").

### 2.5 `code/04_drift_vs_prior.py` — release-over-release

```python
# code/04_drift_vs_prior.py
# Compare this slice's subpop_table to a prior slice's; flag any subpop whose
# representation ratio or share moved by more than 20% relative.
from __future__ import annotations

from pathlib import Path

import polars as pl

CUR = Path("audit/subpop_table.parquet")
PRIOR = Path("reference/prior_slice_subpop_table.parquet")
OUT = Path("audit/drift.parquet")


def main() -> None:
    if not PRIOR.exists():
        OUT.write_bytes(pl.DataFrame({
            "axis": [], "value": [], "delta_share": [], "flagged": []
        }).to_arrow().serialize())
        print("[04] no prior slice; skipping drift")
        return
    cur = pl.read_parquet(CUR)
    pri = pl.read_parquet(PRIOR)
    joined = cur.join(pri, on=["axis", "value"], how="outer", suffix="_prior")
    joined = joined.with_columns([
        (pl.col("observed_share") - pl.col("observed_share_prior")).alias("delta_share"),
    ])
    joined = joined.with_columns([
        (pl.col("delta_share").abs() > 0.02).alias("flagged"),
    ])
    joined.write_parquet(OUT)


if __name__ == "__main__":
    main()
```

### 2.6 `reference/reference_shares.csv` — honesty about the reference

```csv
axis,value,expected_share,source,note
tld,com,0.52,"<citation of reference distribution at run time>","adjust at build time"
tld,org,0.08,"<citation>",""
tld,net,0.06,"<citation>",""
tld,edu,0.02,"<citation>",""
tld,gov,0.01,"<citation>",""
tld,other,0.31,"<citation>","catch-all"
language_score,0.50,0.02,"prior_slice:v0.9.0",""
language_score,0.55,0.03,"prior_slice:v0.9.0",""
...
```

**The reference must be cited.** A reference without a citation is worse
than no reference: it passes judgement without accountability. Prefer, in
order of preference:

1. A prior slice (tag + git SHA + manifest SHA).
2. A published population distribution (with citation).
3. A policy target distribution set by the owning team (with decision
   record link).

If none of these exist for an axis, mark the axis "policy axis — target
is dispersion" (like `pld_top100` above) and audit dispersion instead.

### 2.7 `code/05_render_report.py` — report structure

Render a Markdown report with these sections:

1. **Scope.** Which axes, which slice version, which reference per axis.
2. **Subpopulation table.** The parquet rendered with `representation
   ratio` highlighted when outside [0.8, 1.25] and `support ≥ min_support`.
3. **Distribution tests.** KS statistic + Wasserstein distance per
   continuous axis, with units, with "at this N, KS p-values are not
   informative" called out.
4. **Drift vs prior.** Flagged subpopulations only, with magnitude.
5. **Actionable findings.** One paragraph per over/under-represented
   subpopulation: *what will we do about it* — re-weight, re-seed,
   accept with annotation, accept unchanged.
6. **Limits of inference.** The audit tells you about the *corpus*. It
   does not tell you about speakers / regions / groups except as mediated
   by the axis derivation. A TLD is not a country; a language score is
   not a language-skill signal. State this.

### 2.8 `Makefile`

```makefile
.PHONY: all audit clean
all: audit
audit:
	python code/01_derive_axes.py
	python code/02_subpop_table.py
	python code/03_distribution_tests.py
	python code/04_drift_vs_prior.py
	python code/05_render_report.py
clean:
	rm -rf audit/*.parquet report/subpop_distribution_audit.md
```

## 3. Validation steps

1. **Every axis has a cited reference** (or is marked a policy axis).
   Grep `reference_shares.csv` for a blank `source` column — any blank
   fails review.
2. **Every subpop row has a `support` count** and the audit respects
   `min_support`: rows below support must render as "insufficient
   support", not as a low-N ratio.
3. **Representation ratios reconcile with shares.** Pick a random row:
   `observed_share / expected_share` matches `representation_ratio` to
   three decimals.
4. **At the slice's N, KS p-values are NOT the report's headline.**
   Headline must be the KS statistic (or Wasserstein distance). A report
   that says "p < 0.001" without the statistic magnitude is bad science
   writing; fix it.
5. **Drift comparison exists if a prior slice exists.** If no prior slice
   is available, `drift.parquet` is empty and the report says so
   explicitly — not silent.
6. **"Limits of inference" section is present.** Checkable by grep.
7. **Reproducibility.** `make clean && make audit` produces byte-identical
   `subpop_table.parquet`. Any non-determinism in the derivation step is
   a bug.
8. **No invented reference distributions.** Any `expected_share` whose
   source is blank or "see paper / wiki" without a specific citation
   fails this check.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 22/30 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                                      | Critical? |
|---|------------------------------------------------------------------------------------------------|-----------|
| 1 | Every axis has a reference source (slice/publication/policy) with citation                    | Yes       |
| 2 | Subpop table includes support + observed share + expected share + ratio + CI                  | Yes       |
| 3 | Minimum support is enforced; small-N rows render as "insufficient support"                    | Yes       |
| 4 | Continuous axes report KS statistic AND Wasserstein distance with units                       | Yes       |
| 5 | Report notes that at large N, KS p-values are uninformative — uses statistic magnitude        | Yes       |
| 6 | Drift vs prior slice is computed OR explicitly marked "no prior"                              | Yes       |
| 7 | Every flagged subpop has an actionable disposition (re-weight / re-seed / accept)             | Yes       |
| 8 | Limits-of-inference section is present and names TLD-vs-country / score-vs-skill distinctions | Yes       |
| 9 | Audit manifest captures slice SHA, reference SHAs, axis YAML, git SHA, date                   | Yes       |
| 10 | Governance anchor (NIST AI RMF `MEASURE 2.11` / OWASP ML Top 10) cited                       | No        |
| 11 | Unverified reference shares are flagged explicitly (prose caveat or `<!-- unverified: ... -->`), not guessed | Yes       |
| 12 | Report distinguishes "policy axes" (target is dispersion) from reference axes                  | No        |
| 13 | `Makefile` reproduces the audit from clean clone                                              | No        |
| 14 | Report is plain Markdown, not embedded images from a notebook                                 | No        |
| 15 | At least one dispersion check (e.g. top-N PLD share) complements the ratio view               | No        |

## 5. Common mistakes

- **Comparing to no reference.** A subpop audit without an expected
  distribution is just a histogram. The whole point is the comparison.
- **Using the slice itself as its own reference.** "The top TLD is .com
  at 52% and the expected is 52%" — because you set expected from the
  slice. Reference must be external to the slice under audit.
- **p-values as headline at N = 10⁵.** KS is going to be significant.
  Report the *statistic*; p-values at this scale are decorative.
- **Reading TLD / PLD as country / demographic.** `.co.uk` sites are not
  the UK; sites in English are not English speakers. Keep the axis name
  honest to the *signal*, not the *inference*.
- **Reporting representation ratios without CIs.** A 1.3 ratio with
  support = 60 is noise; with support = 60,000 it's a real finding.
  Readers can't tell without the CI.
- **Treating 'representation ratio = 1.0' as 'done'.** Dispersion, long
  tail, and interaction of axes matter too — a perfectly on-target TLD
  distribution can still have 90% of bytes in 5 hosts.
- **No "limits of inference" section.** This is the single most common
  misread of audit numbers in the wild. Write it explicitly; refer the
  reader to it from the report's intro.
- **No actionable dispositions.** If every finding is "flagged", the
  audit is noise. Each flag must resolve to a decision (even if the
  decision is "accept and document as known limitation").

## 6. References

- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MEASURE 2.11`
  is the compliance hook for this audit.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML01 (Input Manipulation) and ML02 (Data Poisoning) both key off
  lopsided subpopulation coverage.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. "Supply Chain
  Compromise: ML Artifacts" surfaces when one dominant source skews a
  corpus.
- **ISO/IEC 5259-2:2024.** The "representativeness" and "balance"
  characteristics audited here come from this standard; the Exercise 02
  matrix flagged them uncovered.
- **Kolmogorov–Smirnov test (two-sample):** `scipy.stats.ks_2samp` —
  <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html>.
- **Wasserstein distance:** `scipy.stats.wasserstein_distance` —
  <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.wasserstein_distance.html>.
- **Wilson score interval:** standard reference, Agresti & Coull 1998.
  Cite the Agresti & Coull 1998 paper precisely in the audit report if the
  write-up attributes the CI formula.
- **Local anchors:** mod-103 Exercise 01 (the slice this audits);
  Exercise 02 of this module (the coverage matrix this fills in);
  Exercise 03 of this module (Dodge audit — complementary, not
  overlapping).
