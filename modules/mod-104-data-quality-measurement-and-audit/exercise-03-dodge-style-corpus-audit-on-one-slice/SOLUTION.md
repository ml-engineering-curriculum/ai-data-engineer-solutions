# Exercise 03 — Dodge-Style Corpus Audit on One Slice (Solution)

## 1. Solution overview

This is an **implementation + analysis** exercise. The learner runs a
Dodge-style corpus audit (Dodge et al., *Documenting Large Webtext
Corpora: A Case Study on the Colossal Clean Crawled Corpus*, EMNLP 2021)
against the slice built in mod-103 Exercise 01. The point is not to
reproduce the paper's exact numbers — those are specific to C4 — but to
reproduce its **methodology**, so the learner leaves with concrete
answers to the same questions Dodge et al. asked of C4, now asked of
*their* slice.

A good solution produces:

1. A **source composition** report — top-N URL hosts, PLDs (pay-level
   domains), and TLDs by byte / token / document count, with a long-tail
   coverage curve.
2. A **temporal composition** report — the earliest and latest content
   dates the slice spans (from HTTP `Last-Modified` / sitemap /
   WARC-Date / embedded date signals), binned by month.
3. A **filter-impact** report — for each operator in the mod-103 pipeline
   (acquire-stage check, language ID, extractor), how many documents were
   dropped and from which hosts.
4. A **content-flag** pass — rough counts of documents matching
   lists-of-known-filtered patterns (profanity / blocklist, machine-
   generated text markers, excessive boilerplate), using **published**
   lists only. Record list SHA-256s in the audit manifest.
5. An **audit report** (~1,000 words) that reads like Dodge et al.'s
   §4–§5 but about this slice, with citations to the paper, the operator
   configs, and the mod-103 manifest.

Framing: the paper's thesis is that very large web corpora are *not
documented* enough to know what they contain — and that this is a
governance, not just a curiosity, problem. OWASP ML Top 10 **ML02**
calls out the same gap from the attack side; MITRE ATLAS "Poison
Training Data" techniques exploit precisely the blind spots an
undocumented corpus creates; NIST AI RMF `MEASURE 2.11` ("fairness and
bias … are examined and documented") is the same obligation on the
compliance side.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── slice_pin.yaml                 # same shape as Exercise 01
├── requirements.lock
├── Makefile
├── blocklists/                     # published lists; SHAs pinned in manifest
│   ├── ldnoobw.en.txt              # List of Dirty, Naughty, Obscene and Otherwise Bad Words
│   └── machine_gen_markers.txt     # e.g. "lorem ipsum", templated footers
├── code/
│   ├── 01_source_composition.py
│   ├── 02_temporal_composition.py
│   ├── 03_filter_impact.py
│   ├── 04_content_flags.py
│   ├── 05_render_report.py
│   └── common.py
├── audit/
│   ├── source_top_hosts.parquet
│   ├── source_top_plds.parquet
│   ├── temporal_hist.parquet
│   ├── filter_impact.parquet
│   ├── content_flag_counts.parquet
│   └── audit_manifest.json
└── report/
    └── corpus_audit.md
```

### 2.2 `code/01_source_composition.py` — who is in the corpus?

```python
# code/01_source_composition.py
# Group slice docs by host / PLD / TLD; rank by both byte and doc counts.
# A doc-count-only ranking misses that one newswire can dominate the bytes.
from __future__ import annotations

from pathlib import Path

import polars as pl
import tldextract

IN = Path("../mod-103/manifest/per_record_manifest.parquet")
OUT_DIR = Path("audit")


def pld(host: str) -> str:
    # tldextract computes the pay-level domain (effective 2nd-level), e.g.
    # news.bbc.co.uk → bbc.co.uk; used in Dodge et al. for host grouping.
    ex = tldextract.extract(host or "")
    return ".".join(p for p in [ex.domain, ex.suffix] if p) or host or ""


def main() -> None:
    df = pl.read_parquet(IN).filter(pl.col("language") == "en")
    df = df.with_columns([
        pl.col("url").map_elements(lambda u: u.split("/")[2] if u else "",
                                   return_dtype=pl.Utf8).alias("host"),
    ])
    df = df.with_columns([
        pl.col("host").map_elements(pld, return_dtype=pl.Utf8).alias("pld"),
        pl.col("text").str.len_bytes().alias("doc_bytes"),
    ])
    OUT_DIR.mkdir(parents=True, exist_ok=True)

    hosts = (df.group_by("host")
             .agg([pl.len().alias("docs"), pl.sum("doc_bytes").alias("bytes")])
             .sort("bytes", descending=True))
    plds = (df.group_by("pld")
            .agg([pl.len().alias("docs"), pl.sum("doc_bytes").alias("bytes")])
            .sort("bytes", descending=True))
    tlds = (df.with_columns(pl.col("host").str.split(".").list.last().alias("tld"))
            .group_by("tld")
            .agg([pl.len().alias("docs"), pl.sum("doc_bytes").alias("bytes")])
            .sort("bytes", descending=True))

    hosts.write_parquet(OUT_DIR / "source_top_hosts.parquet")
    plds.write_parquet(OUT_DIR / "source_top_plds.parquet")
    tlds.write_parquet(OUT_DIR / "source_top_tlds.parquet")

    # Coverage: how much of the corpus is in the top K PLDs?
    total_bytes = plds["bytes"].sum()
    cum = plds.with_columns(
        (pl.col("bytes").cum_sum() / total_bytes).alias("cum_share")
    )
    cum.write_parquet(OUT_DIR / "pld_cumulative_coverage.parquet")

    print(f"[01] {len(plds)} distinct PLDs; top-10 share = "
          f"{float(cum.head(10)['cum_share'].tail(1)[0]):.3f}")


if __name__ == "__main__":
    main()
```

The top-K coverage curve is the Dodge et al. figure that matters most:
if the top 10 PLDs cover > 50% of bytes, a reviewer should (a) know this
and (b) check those 10 PLDs' licensing and opt-out status by hand.

### 2.3 `code/02_temporal_composition.py` — when was it written?

```python
# code/02_temporal_composition.py
# Temporal composition from WARC-Date; HTTP Last-Modified is a plus if present.
# NB: WARC-Date is *fetch* time, not *authoring* time; be explicit in the
# report that these are lower bounds on content age.
from __future__ import annotations

from pathlib import Path

import polars as pl

IN = Path("../mod-103/manifest/per_record_manifest.parquet")
OUT = Path("audit/temporal_hist.parquet")


def main() -> None:
    df = pl.read_parquet(IN).filter(pl.col("language") == "en")
    # If upstream didn't carry WARC-Date forward, re-extract from WARC headers
    # in mod-103 Exercise 01 and backfill before running this stage.
    if "warc_date" not in df.columns:
        raise SystemExit(
            "per_record_manifest.parquet missing warc_date; "
            "backfill in mod-103 before running this audit."
        )
    df = df.with_columns([
        pl.col("warc_date").str.to_datetime().dt.truncate("1mo").alias("month"),
    ])
    hist = (df.group_by("month").agg(pl.len().alias("docs")).sort("month"))
    hist.write_parquet(OUT)
    span = (hist["month"].min(), hist["month"].max())
    print(f"[02] temporal span: {span[0]} → {span[1]}")


if __name__ == "__main__":
    main()
```

### 2.4 `code/03_filter_impact.py` — what did the pipeline drop?

```python
# code/03_filter_impact.py
# For each filter stage in mod-103 (acquire-stage, language-ID, extractor),
# compute drop counts and top-N hosts hit. This is the "what was quietly
# removed" view that Dodge et al. §4 shows for C4.
from __future__ import annotations

from pathlib import Path

import polars as pl

IN = Path("../mod-103/manifest/per_record_manifest.parquet")
OUT = Path("audit/filter_impact.parquet")


def main() -> None:
    df = pl.read_parquet(IN)
    # mod-103's per-record manifest keeps *every* candidate, including drops.
    stages = [
        ("acquire_stage", pl.col("acquire_allowed") == False),
        ("extractor_failed", pl.col("extract_error") != ""),
        ("language_filter",
         (pl.col("language") != "en") | (pl.col("language_score") < 0.5)),
    ]
    rows = []
    for name, cond in stages:
        drops = df.filter(cond)
        top = (drops.with_columns(
                   pl.col("url").map_elements(lambda u: u.split("/")[2] if u else "",
                                              return_dtype=pl.Utf8).alias("host"))
                   .group_by("host").agg(pl.len().alias("docs"))
                   .sort("docs", descending=True).head(25))
        for r in top.iter_rows(named=True):
            rows.append({"stage": name, "host": r["host"], "docs": r["docs"]})
        print(f"[03] {name}: dropped {len(drops)} docs")
    pl.DataFrame(rows).write_parquet(OUT)


if __name__ == "__main__":
    main()
```

### 2.5 `code/04_content_flags.py` — rough content signals

```python
# code/04_content_flags.py
# Count occurrences of pattern families from *published* blocklists.
# This is NOT a content-moderation call; it's a composition signal.
# "0.3% of docs match ldnoobw" is a corpus-shape claim, not a safety claim.
from __future__ import annotations

import hashlib
import json
import re
from pathlib import Path

import polars as pl

IN = Path("../mod-103/manifest/per_record_manifest.parquet")
LDNOOBW = Path("blocklists/ldnoobw.en.txt")
MG = Path("blocklists/machine_gen_markers.txt")
OUT = Path("audit/content_flag_counts.parquet")


def sha256(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()


def compile_or(words: list[str]) -> re.Pattern[str]:
    # word-boundary \b alternation; case-insensitive
    pat = r"\b(" + "|".join(re.escape(w.strip()) for w in words if w.strip()) + r")\b"
    return re.compile(pat, re.IGNORECASE)


def main() -> None:
    bad = compile_or(LDNOOBW.read_text().splitlines())
    mg = compile_or(MG.read_text().splitlines())

    df = pl.read_parquet(IN).filter(pl.col("language") == "en")
    df = df.with_columns([
        pl.col("text").map_elements(lambda t: bool(bad.search(t or "")),
                                    return_dtype=pl.Boolean).alias("has_ldnoobw"),
        pl.col("text").map_elements(lambda t: bool(mg.search(t or "")),
                                    return_dtype=pl.Boolean).alias("has_machine_gen_marker"),
    ])
    counts = df.select([
        pl.len().alias("docs"),
        pl.sum("has_ldnoobw").alias("docs_with_ldnoobw_match"),
        pl.sum("has_machine_gen_marker").alias("docs_with_machine_gen_marker"),
    ])
    counts.write_parquet(OUT)
    print({
        "ldnoobw_sha256": sha256(LDNOOBW),
        "mg_sha256": sha256(MG),
        **counts.row(0, named=True),
    })


if __name__ == "__main__":
    main()
```

### 2.6 `code/05_render_report.py` — the audit write-up

Pulls the parquet summaries into a Markdown report. The template mirrors
Dodge et al.'s structure: slice description, source composition, temporal
composition, filter impact, content flags, limitations.

```python
# code/05_render_report.py
from __future__ import annotations
import datetime as dt, hashlib, json
from pathlib import Path
import polars as pl

OUT = Path("report/corpus_audit.md")


def pct(numer: int, denom: int) -> str:
    return "n/a" if not denom else f"{100.0 * numer / denom:.2f}%"


def main() -> None:
    hosts = pl.read_parquet("audit/source_top_hosts.parquet")
    plds = pl.read_parquet("audit/source_top_plds.parquet")
    tlds = pl.read_parquet("audit/source_top_tlds.parquet")
    cum  = pl.read_parquet("audit/pld_cumulative_coverage.parquet")
    hist = pl.read_parquet("audit/temporal_hist.parquet")
    fi   = pl.read_parquet("audit/filter_impact.parquet")
    cf   = pl.read_parquet("audit/content_flag_counts.parquet")
    total_docs = int(cf.row(0, named=True)["docs"])

    parts = [
        "# Corpus audit — slice-cc-en-small-v1\n",
        f"Built {dt.datetime.utcnow().isoformat(timespec='seconds')}Z.\n",
        "## Source composition\n",
        f"- {len(plds)} distinct PLDs across {len(hosts)} hosts.\n",
        f"- Top 10 PLDs cover {float(cum.head(10)['cum_share'].tail(1)[0]):.1%} of bytes.\n",
        "\n### Top 25 PLDs (by bytes)\n",
        plds.head(25).to_pandas().to_markdown(index=False),
        "\n\n### TLD distribution\n",
        tlds.to_pandas().to_markdown(index=False),
        "\n\n## Temporal composition\n",
        f"- Span: {hist['month'].min()} → {hist['month'].max()}\n",
        f"- NB: `warc_date` is *fetch* time, a lower bound on document age.\n",
        "\n## Filter impact\n",
        fi.to_pandas().to_markdown(index=False),
        "\n\n## Content flags\n",
        f"- ldnoobw matches: {pct(cf.row(0, named=True)['docs_with_ldnoobw_match'], total_docs)}\n",
        f"- machine-gen markers: {pct(cf.row(0, named=True)['docs_with_machine_gen_marker'], total_docs)}\n",
        "\n## Limitations\n",
        "- Content flags are counts of *pattern matches*, not safety judgments.\n",
        "- Temporal data is fetch-time, not authoring-time.\n",
        "- No demographic or geographic audit; see Exercise 04.\n",
    ]
    OUT.parent.mkdir(parents=True, exist_ok=True)
    OUT.write_text("\n".join(parts))
    print(f"[05] wrote {OUT}")


if __name__ == "__main__":
    main()
```

### 2.7 `audit/audit_manifest.json`

The `05_render_report.py` step (or a sibling emitter) should also drop a
JSON manifest with: slice name + version + SHA, blocklist SHAs, dates,
software versions, git SHA. Without this, the audit's numbers cannot be
traced to the inputs that produced them.

### 2.8 `Makefile`

```makefile
.PHONY: all audit clean
all: audit

audit:
	python code/01_source_composition.py
	python code/02_temporal_composition.py
	python code/03_filter_impact.py
	python code/04_content_flags.py
	python code/05_render_report.py

clean:
	rm -rf audit/*.parquet audit/audit_manifest.json report/corpus_audit.md
```

### 2.9 Where the limits are honest

The audit will not tell you:

- Whether specific content is copyrighted. (License / provenance is
  mod-102.)
- Whether specific content is personally identifying. (mod-110.)
- Whether specific content is machine-generated beyond pattern-matching.
  (mod-108.)
- Representativeness of the slice to any real-world population — the
  audit tells you *what's there*, not *what should be there*. Exercise 04
  of this module handles that.

Writing those limits into the report is non-optional: Dodge et al.'s
contribution is precisely the honesty about what an audit does and does
not establish.

## 3. Validation steps

1. **Numbers reproduce from code.** Running `make clean && make audit` on
   a fixed slice produces byte-identical `*.parquet` files (modulo file
   metadata). If any number in `report/corpus_audit.md` isn't derived
   from one of those parquets, it's a fabricated figure.
2. **Every blocklist has a SHA.** `blocklists/*.txt` paths appear in
   `audit/audit_manifest.json` with their SHA-256. Without this, the
   content-flag numbers aren't reproducible when the lists update.
3. **Top-10 PLD share matches independently.** A reviewer can re-sum the
   top 10 rows of `source_top_plds.parquet` and get the same share as
   the report. A shift means a stale report.
4. **Filter impact adds up.** `sum(dropped) + len(final_slice) ==
   len(per_record_manifest)`; a mismatch means a drop stage was invisible
   to the audit.
5. **No invented figures.** Dodge et al.'s C4 numbers are *not* the same
   as this slice's numbers. If the report contains numbers that don't
   appear in any parquet, remove them — a Dodge-*style* audit reports on
   *this* slice, not on C4.
6. **Limitations section is present.** A missing limitations section is
   an auto-fail on the review checklist.
7. **Temporal field is actually from the WARC header.** A hallucinated
   date field is worse than no date; if the manifest didn't carry
   `warc_date`, backfill in mod-103 instead of fabricating here.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
A passing solution scores ≥ 20/28 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                                | Critical? |
|---|------------------------------------------------------------------------------------------|-----------|
| 1 | Audit runs against the mod-103 slice (same version), not a toy dataset                   | Yes       |
| 2 | Source composition reports hosts, PLDs, TLDs, and a top-K coverage curve                 | Yes       |
| 3 | Temporal composition reports fetch time AND explicitly flags it as a lower bound         | Yes       |
| 4 | Filter-impact stage breaks out each pipeline stage (acquire / extract / language)        | Yes       |
| 5 | Content flags cite published blocklists with pinned SHAs                                 | Yes       |
| 6 | Content flags are reported as composition signals, not safety judgments                  | Yes       |
| 7 | Limitations section lists what the audit does NOT establish                              | Yes       |
| 8 | Every number in the report traces to a parquet artefact                                  | Yes       |
| 9 | Audit manifest captures slice SHA + blocklist SHAs + git SHA + dates                     | Yes       |
| 10 | Report is reviewable (table of contents / section headings) not a code dump              | No        |
| 11 | Governance anchor cited (OWASP ML Top 10 / NIST AI RMF `MEASURE 2.11` / MITRE ATLAS)    | No        |
| 12 | Unverified or paraphrased figures are flagged explicitly (prose caveat or `<!-- unverified: ... -->`) | Yes       |
| 13 | Report explicitly cites Dodge et al. 2021 (paper, not a blog paraphrase)                 | Yes       |
| 14 | Known-missing audits (PII, license, dedup, representativeness) delegate to siblings      | No        |

## 5. Common mistakes

- **Reproducing Dodge et al.'s C4 figures.** The paper is a case study on
  C4; your slice is not C4. Numbers are not transferable. Reuse
  *methodology*, not *results*.
- **Doc-count ranking without byte ranking.** A single newswire can
  produce 10× the bytes of 100 blogs; a doc-count top-10 can hide the
  slice's actual byte concentration.
- **Using a one-off scraped host list for "content flags".** Flags must
  come from published lists so the audit is reviewable; lists must be
  SHA-pinned so the counts are reproducible.
- **Reporting content flags as "safety".** "0.3% of documents matched a
  dirty-word list" says nothing about whether those documents are unsafe
  to use. State this in the report.
- **Dropping silent-drop stages.** If mod-103's pipeline fails extraction
  on 0.5% of candidates and your audit doesn't show that 0.5%, your audit
  is lying about pipeline impact.
- **Pretending the audit covers PII / license / representativeness.** It
  doesn't; those are explicit scope boundaries. Call them out.
- **No audit manifest.** Without a manifest the audit's own
  reproducibility is zero — which is exactly the gap the Dodge et al.
  paper is complaining about.

## 6. References

- **Dodge, Sap, Marasović, Agnew, Ilharco, Groeneveld, Mitchell, Gardner.
  2021. *Documenting Large Webtext Corpora: A Case Study on the Colossal
  Clean Crawled Corpus*.** EMNLP 2021. arXiv:2104.08758. This exercise's
  methodology (source composition, filter impact, content flags, temporal
  breakdown) follows the paper's structure. Confirm author order and the
  exact arXiv ID / EMNLP proceedings page at write time rather than
  trusting this template.
- **Common Crawl columnar index documentation** —
  <https://commoncrawl.org/get-started>. For the WARC header fields
  (`WARC-Date`, `WARC-Target-URI`) the temporal + source stages lean on.
- **List of Dirty, Naughty, Obscene and Otherwise Bad Words (LDNOOBW)**
  — a community-maintained multilingual word list. Pin a specific
  revision SHA in `audit_manifest.json`. Cite the current upstream
  repository URL at write time; do not trust a stale fork.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  ML02 (Data Poisoning Attack) is the attack side of the "we don't know
  what's in the corpus" problem.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. "Poison Training Data" /
  "Supply Chain Compromise" techniques are the concrete attacker patterns
  an undocumented corpus enables.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MEASURE 2.11`
  (fairness and bias examined and documented) is the compliance side.
- **Local anchors:** mod-103 Exercise 01 (the slice this audits);
  Exercise 02 of this module (ISO coverage matrix — this audit fills the
  Credibility / Diversity / Currentness rows); Exercise 04 of this module
  (representativeness, distribution — scope boundary with this audit).
