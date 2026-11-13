# Exercise 02 — trafilatura vs resiliparse Extraction Benchmark (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds a small, honest
benchmark that replaces "trafilatura is the Python one, resiliparse is the
Cython one" with numbers measured on a reference set they control, then writes
a decision memo that cites those numbers.

A good solution has three parts:

1. A **reference set** of ~200 HTML pages drawn from at least three sources
   (Exercise-01 slice, a public labelled corpus, and hand-picked stress
   pages), with reference extractions produced by a documented method.
2. A **benchmark harness** that runs both extractors on every page and records
   per-page metrics (ROUGE-L, character-level F1, boilerplate ratio,
   throughput, robustness) plus the hardware / version context.
3. A one-page **decision memo** that recommends one extractor (or a hybrid
   router), cites the numbers that drove the recommendation, and names an
   explicit re-evaluation trigger.

The exercise is deliberately *small*. An industrial extractor benchmark would
cover thousands of pages, multiple labellers, and inter-annotator agreement.
The learner's job is to produce a defensible first-cut comparison that is
honest about its scope.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-02/
├── README.md
├── requirements.txt
├── requirements.lock
├── reference/
│   ├── slice/                     # 100 pages from Exercise 01
│   │   ├── page_001.html
│   │   ├── page_001.reference.txt
│   │   └── page_001.meta.json     # TLD, page_type, origin
│   ├── labelled/                  # 50 pages from a public labelled corpus
│   └── stress/                    # 50 hand-picked edge cases
├── code/
│   ├── 01_build_reference.py
│   ├── 02_run_extractors.py
│   ├── 03_compute_metrics.py
│   ├── 04_summarise.py
│   └── common.py
├── results/
│   ├── per_page.parquet
│   ├── summary.md
│   └── env.json
└── decision_memo.md
```

### 2.2 Reference-set methodology

Three sources, documented in `README.md`:

- **100 pages from the Exercise-01 slice.** Sampled across TLDs
  (`.com`/`.net`/`.org`/`.edu`/`.gov`) with 20 pages per TLD. These are the
  pages whose extraction the pipeline actually cares about.
- **50 pages from a public labelled corpus.** ScrapingHub's
  `article-extraction-benchmark` (<https://github.com/scrapinghub/article-extraction-benchmark>)
  is the pragmatic choice because it ships ground-truth extractions.
  Alternatives (CleanEval, Dragnet's corpus) may or may not still be
  reachable; any inability to download must be marked
  `<!-- unverified: ... -->` rather than silently dropped.
- **50 stress pages, hand-picked.** Target page types: forum thread (`#forum`),
  product page (`#product`), long-scroll news with infinite-load widgets,
  paywalled / consent-wall pages, deliberately malformed HTML, one non-English
  page. These are the pages that produce the *interesting* findings.

Reference-extraction policy (written into `README.md`):

- Labelled corpus → use the corpus's reference text directly.
- Slice pages → run Mozilla's **Readability.js** inside a headless Chromium
  and treat the result as reference. This introduces a bias toward
  Readability-like extractors (trafilatura's output has been reported to
  track Readability more closely than resiliparse's does on article pages);
  the decision memo must name this bias.
- Stress pages → hand-label. Even 50 hand-labels is enough to catch gross
  failure modes; the memo must not overclaim more precision than that.

### 2.3 `code/02_run_extractors.py` — the harness

```python
# code/02_run_extractors.py
import json
import resource
import time
from pathlib import Path
from typing import Callable

import polars as pl
import trafilatura
from resiliparse.extract.html2text import extract_plain_text as resiliparse_extract
from resiliparse.parse.html import HTMLTree

REF_ROOT = Path("reference")
OUT = Path("results/per_page.parquet")

def _rss_kb() -> int:
    return resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

def run_trafilatura(html: str) -> str:
    # Pin the configuration explicitly; defaults have drifted historically.
    return trafilatura.extract(
        html,
        favor_precision=True,       # reduce false-positive text
        include_comments=False,
        include_tables=True,
        deduplicate=True,
        output_format="txt",
    ) or ""

def run_resiliparse(html: str) -> str:
    # resiliparse exposes a raw-tree path; use the high-level helper for parity.
    tree = HTMLTree.parse(html)
    return resiliparse_extract(tree, main_content=True, preserve_formatting=False) or ""

EXTRACTORS: dict[str, Callable[[str], str]] = {
    "trafilatura": run_trafilatura,
    "resiliparse": run_resiliparse,
}

def _iter_pages():
    for part in ("slice", "labelled", "stress"):
        for html_path in sorted((REF_ROOT / part).glob("*.html")):
            ref_path = html_path.with_suffix(".reference.txt")
            meta_path = html_path.with_suffix(".meta.json")
            yield {
                "page_id": html_path.stem,
                "source": part,
                "html_path": str(html_path),
                "ref_text": ref_path.read_text(encoding="utf-8", errors="replace"),
                "meta": json.loads(meta_path.read_text()),
            }

def main() -> None:
    rows = []
    for page in _iter_pages():
        html = Path(page["html_path"]).read_text(encoding="utf-8", errors="replace")
        for name, fn in EXTRACTORS.items():
            t0 = time.perf_counter()
            rss0 = _rss_kb()
            raised, text = "", ""
            try:
                text = fn(html)
            except Exception as e:
                raised = f"{type(e).__name__}: {e}"
            t1 = time.perf_counter()
            rows.append({
                "page_id": page["page_id"],
                "source": page["source"],
                "tld": page["meta"].get("tld"),
                "page_type": page["meta"].get("page_type"),
                "extractor": name,
                "extract_ms": (t1 - t0) * 1000.0,
                "peak_rss_kb_delta": _rss_kb() - rss0,
                "raised": raised,
                "empty": len(text.strip()) == 0,
                "fragmentary": 0 < len(text.strip()) < 200,
                "chars_out": len(text),
                "output_text": text,
                "ref_text": page["ref_text"],
            })
    pl.DataFrame(rows).write_parquet(OUT)

if __name__ == "__main__":
    main()
```

Two decisions worth naming:

- **`favor_precision=True` on trafilatura, `main_content=True` on resiliparse.**
  Both choices are the closest "give me the article body, drop everything
  else" configurations in each library. Running the ablation is a stretch
  goal; the baseline must pick one per extractor and document it.
- **Peak RSS is a delta per page, not absolute.** Absolute RSS is dominated by
  the Python interpreter; the delta reflects extractor-attributable memory.

### 2.4 `code/03_compute_metrics.py` — per-page metrics

```python
# code/03_compute_metrics.py
from pathlib import Path

import polars as pl
from rouge_score import rouge_scorer

PER_PAGE = Path("results/per_page.parquet")
OUT = Path("results/per_page_scored.parquet")

def char_f1(pred: str, ref: str) -> float:
    if not pred or not ref:
        return 0.0
    pred_c = list(pred)
    ref_c = list(ref)
    # Multiset intersection: character-level precision/recall.
    from collections import Counter
    pc, rc = Counter(pred_c), Counter(ref_c)
    overlap = sum((pc & rc).values())
    precision = overlap / max(1, len(pred_c))
    recall = overlap / max(1, len(ref_c))
    if precision + recall == 0:
        return 0.0
    return 2 * precision * recall / (precision + recall)

def boilerplate_ratio(pred: str, ref: str) -> float:
    """Lines in pred that are not in ref, over total pred lines.

    Interpretation: what fraction of the extractor's output is
    content that the reference considered boilerplate / cruft?
    Lower is better. A value of 0.0 means every line was in the reference;
    1.0 means nothing in the output matched the reference at all.
    """
    pred_lines = {ln.strip() for ln in pred.splitlines() if ln.strip()}
    ref_lines = {ln.strip() for ln in ref.splitlines() if ln.strip()}
    if not pred_lines:
        return 0.0
    non_ref = pred_lines - ref_lines
    return len(non_ref) / len(pred_lines)

def main() -> None:
    df = pl.read_parquet(PER_PAGE)
    scorer = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=True)
    rouge_l = []
    f1 = []
    bp = []
    for row in df.iter_rows(named=True):
        pred = row["output_text"] or ""
        ref = row["ref_text"] or ""
        score = scorer.score(ref, pred)["rougeL"].fmeasure if pred and ref else 0.0
        rouge_l.append(score)
        f1.append(char_f1(pred, ref))
        bp.append(boilerplate_ratio(pred, ref))
    df = df.with_columns([
        pl.Series("rouge_l", rouge_l),
        pl.Series("char_f1", f1),
        pl.Series("boilerplate_ratio", bp),
        pl.Series("robust",
                  [not r and not e for r, e in zip(df["raised"], df["empty"])]),
    ])
    df.write_parquet(OUT)

if __name__ == "__main__":
    main()
```

### 2.5 `code/04_summarise.py` — aggregates + per-type breakdown

```python
# code/04_summarise.py
import json
import platform
import subprocess
from pathlib import Path

import polars as pl

SCORED = Path("results/per_page_scored.parquet")
SUMMARY = Path("results/summary.md")
ENV = Path("results/env.json")

def _env() -> dict:
    def pip_ver(pkg: str) -> str:
        out = subprocess.run(
            ["python", "-m", "pip", "show", pkg], capture_output=True, text=True
        ).stdout
        for ln in out.splitlines():
            if ln.lower().startswith("version:"):
                return ln.split(":", 1)[1].strip()
        return "unknown"
    return {
        "python": platform.python_version(),
        "platform": platform.platform(),
        "cpu_count": platform.processor() or "unknown",
        "trafilatura": pip_ver("trafilatura"),
        "resiliparse": pip_ver("resiliparse"),
        "rouge_score": pip_ver("rouge-score"),
    }

def main() -> None:
    env = _env()
    ENV.write_text(json.dumps(env, indent=2))
    df = pl.read_parquet(SCORED)

    agg = (
        df.group_by("extractor")
          .agg([
              pl.col("rouge_l").mean().alias("rougeL_mean"),
              pl.col("rouge_l").median().alias("rougeL_p50"),
              pl.col("rouge_l").quantile(0.1).alias("rougeL_p10"),
              pl.col("char_f1").mean().alias("charF1_mean"),
              pl.col("boilerplate_ratio").mean().alias("bp_ratio_mean"),
              pl.col("extract_ms").mean().alias("ms_mean"),
              pl.col("extract_ms").median().alias("ms_p50"),
              (1000.0 / pl.col("extract_ms").mean()).alias("pages_per_sec_per_core"),
              pl.col("robust").mean().alias("robust_frac"),
          ])
          .sort("extractor")
    )
    by_type = (
        df.group_by(["extractor", "page_type"])
          .agg([
              pl.col("rouge_l").mean().alias("rougeL"),
              pl.col("char_f1").mean().alias("charF1"),
              pl.col("boilerplate_ratio").mean().alias("bp_ratio"),
              pl.col("robust").mean().alias("robust_frac"),
          ])
          .sort(["extractor", "page_type"])
    )
    SUMMARY.write_text(
        "# Benchmark summary\n\n"
        f"## Environment\n\n```json\n{json.dumps(env, indent=2)}\n```\n\n"
        "## Aggregate\n\n" + agg.to_pandas().to_markdown(index=False) + "\n\n"
        "## Per-page-type breakdown\n\n" + by_type.to_pandas().to_markdown(index=False) + "\n"
    )

if __name__ == "__main__":
    main()
```

### 2.6 Decision memo — worked example

> **To:** pretraining pipeline owner, `cc-en-500tb` corpus
>
> **From:** data-eng
>
> **Subject:** Main-content extractor selection — 6-month pin
>
> **Recommendation:** **resiliparse** as the default extractor, with
> **trafilatura** routed to on pages classified as "news/article" by a cheap
> length+density heuristic.
>
> **Why — quality:** On the 200-page reference set, trafilatura leads on
> ROUGE-L on `news/article` pages by a meaningful margin, but the two
> extractors are within noise on `product` and `forum` pages, and trafilatura
> is penalised on `malformed` pages where it bails out more often. The
> boilerplate ratio is ~1.5–2× lower on resiliparse for product pages,
> because trafilatura's precision mode strips legitimate short lines in those
> templates.
>
> **Why — throughput:** resiliparse is substantially faster per core on our
> reference set; the exact factor is hardware-dependent and is reported in
> `results/summary.md` with the pinned versions and CPU model.
>
> **Why — robustness:** resiliparse's robust fraction (returned non-empty
> non-fragmentary text) is higher on the stress set (malformed HTML,
> consent-wall pages). The pipeline runs on 128 cores with no supervision;
> fewer raised exceptions matters more than a small ROUGE-L delta.
>
> **Why — licensing:** resiliparse is Apache-2.0; trafilatura is GPLv3 (verify
> at read time). The pretraining pipeline statically links neither — licence
> is a non-blocker here — but the corporate preference for permissive
> licences on batch pipeline components breaks ties toward resiliparse.
>
> **Hybrid router:** a 15-line heuristic (DOM text density + `<article>`
> presence) picks trafilatura on news/article pages at near-trafilatura
> quality and near-resiliparse throughput; implementation is a stretch goal.
>
> **Re-evaluation trigger:** the slice composition shifts — specifically if
> the share of pages classified as `news/article` exceeds 40% in the corpus
> we are building — OR either library ships a major version. We will re-run
> this benchmark on a fresh 200-page sample at that point.
>
> **Scope caveats:** 200 pages is a first-cut. Reference extractions on the
> 100 slice pages came from Readability.js in headless Chromium; this biases
> ROUGE-L in favour of extractors that happen to agree with Readability.
> Metrics should be treated as directional, not authoritative — mod-104
> covers the quality-measurement machinery that would turn this into a
> production evaluation.

The numeric deltas are deliberately described qualitatively ("meaningful
margin", "~1.5–2×") in this written solution because the specific numbers
depend on the slice and hardware; the learner's memo must cite the actual
numbers from `results/summary.md`.

## 3. Validation steps

1. **Reference set composition matches the acceptance criteria.** ≥200 pages,
   ≥4 page types, ≥5 TLDs. Count the `.meta.json` files, not the HTML files.
2. **Reference methodology is written down.** The README has a paragraph per
   reference source saying how its ground truth was produced, and the
   decision memo acknowledges the resulting bias.
3. **Metrics are per-page before they are aggregated.** `per_page.parquet`
   has one row per `(page_id, extractor)` with every metric — not just
   aggregates. Aggregation that cannot be re-derived from per-page rows is
   not auditable.
4. **Quality metrics report mean, median, and 10th percentile.** The 10th
   percentile is where extractor failures hide; an extractor with a great
   mean and a terrible p10 is a trap.
5. **Throughput is hardware-qualified.** `results/env.json` captures CPU,
   OS, Python version, and extractor versions. "5× slower" without that
   context is not an acceptable finding.
6. **The decision memo references specific numbers, not adjectives.** Every
   recommendation sentence is traceable to a row in `summary.md`.
7. **Versions are pinned.** `requirements.lock` captures `trafilatura==x.y.z`
   and `resiliparse==a.b.c`. The benchmark is invalid when either drifts.
8. **The license caveat is present.** Trafilatura's licence (GPL at time of
   writing, verify current) vs resiliparse's Apache-2.0 appears in the memo
   even when it is not the deciding factor.
9. **Unverifiable dependencies are flagged.** If CleanEval or a labelled
   corpus could not be reached, it appears with
   `<!-- unverified: ... -->` instead of a silently smaller reference.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible).
Pass at ≥18/26 with zero zeros on **critical** rows.

| # | Dimension                                                                        | Critical? |
|---|----------------------------------------------------------------------------------|-----------|
| 1 | Reference set ≥200 pages, ≥4 page types, ≥5 TLDs, composition documented        | Yes       |
| 2 | Reference-extraction methodology is explicit and names its bias                  | Yes       |
| 3 | Both extractors have pinned configs documented in the harness code               | Yes       |
| 4 | Per-page results exist as a single tidy table (`per_page.parquet`)              | Yes       |
| 5 | ROUGE-L, char-F1, boilerplate ratio, throughput, robustness all computed        | Yes       |
| 6 | Aggregates reported with mean + median + 10th percentile                         | Yes       |
| 7 | Per-page-type breakdown present; each type has ≥10 pages                         | No        |
| 8 | Throughput measurements include CPU model, OS, Python version                    | Yes       |
| 9 | Library versions pinned in `requirements.lock`                                    | Yes       |
| 10 | Decision memo recommends one extractor (or a hybrid) and cites numbers           | Yes       |
| 11 | Decision memo includes an explicit re-evaluation trigger                          | Yes       |
| 12 | Licence posture (GPL vs Apache-2.0) is noted even when it does not decide        | No        |
| 13 | Unavailable dependencies are flagged `<!-- unverified: ... -->`              | Yes       |

## 5. Common mistakes

- **Reporting only aggregates.** A single "mean ROUGE-L" number hides
  everything useful. The per-page-type breakdown is where the actual
  finding lives.
- **Comparing a precision-tuned trafilatura against a recall-tuned
  resiliparse.** The matched configuration is the honest one; the ablation
  belongs in a stretch section.
- **Measuring throughput on cold caches with a tiny sample.** 200 pages is
  enough for a direction but not a precise number; the memo must not claim
  more precision than the sample supports.
- **Treating Readability.js output as ground truth without flagging the
  bias.** The bias is real; flagging it is enough.
- **Pinning the "winner" with no re-evaluation trigger.** Extractors improve;
  the slice drifts. A pin with no trigger silently rots.
- **Forgetting the robustness column.** On 128 cores with no supervision, a
  2% robust-fraction gap matters more than a 2% ROUGE-L gap. The memo must
  consider operational cost, not just quality.
- **Not recording the HTML path.** If the per-page table does not carry
  `html_path`, you cannot go back and look at the pages the extractor
  failed on — and you will need to.
- **Claiming the licence does not matter.** It matters at pin time even if
  it is not decisive today; the memo is what a future reviewer reads when
  they want to challenge the pin.

## 6. References

- **trafilatura** — Barbaresi, "Trafilatura: A Web Scraping Library and
  Command-Line Tool for Text Discovery and Extraction" (ACL 2021 Demo),
  <https://aclanthology.org/2021.acl-demo.15/>. Project:
  <https://github.com/adbar/trafilatura>.
- **resiliparse** — ChatNoir / Webis project documentation,
  <https://resiliparse.chatnoir.eu/>. Source:
  <https://github.com/chatnoir-eu/chatnoir-resiliparse>.
- **ScrapingHub article-extraction benchmark** —
  <https://github.com/scrapinghub/article-extraction-benchmark>.
- **Mozilla Readability** — <https://github.com/mozilla/readability>. Used as
  a reference extractor for unlabelled slice pages; its bias toward article
  pages is the main caveat.
- **rouge-score (Python port)** —
  <https://pypi.org/project/rouge-score/>. ROUGE-L implementation used in
  the harness.
- **Local anchors:** Chapter 03 for the extraction landscape; mod-104 for
  the data-quality measurement framework that turns this benchmark into a
  production evaluation.
