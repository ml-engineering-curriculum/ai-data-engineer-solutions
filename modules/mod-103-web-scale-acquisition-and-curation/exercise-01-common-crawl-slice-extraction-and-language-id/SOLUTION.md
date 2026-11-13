# Exercise 01 — Common Crawl Slice Extraction and Language-ID (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner stands up a reproducible,
small-scale version of a CCNet-style acquire → extract → language-ID pipeline
against one Common Crawl snapshot. The solution is a working skeleton — real
output volumes depend on the snapshot the learner picks and are not reproducible
out-of-band here — but the structure, file layout, and manifest shape are
authoritative.

A good solution produces three things:

1. A **selection predicate** expressed as SQL against the Common Crawl columnar
   index, pinned to one snapshot.
2. A **per-record manifest** that, together with the code, makes the slice
   byte-exact-reproducible: anyone with S3 read access can regenerate it.
3. An **acquire-stage opt-out log** keyed by host, with the robots.txt evidence
   SHA-256 attached to every URL.

The exercise exists to make the physical steps real: you have to actually read
the index, actually range-fetch WARC bytes, actually run `lid.176`, and actually
emit a manifest. Later exercises (02, 03, 04) layer quality / library / scale
concerns on top of this skeleton.

The threat framing from the governance modules carries forward: an un-opted-out,
un-provenanced scrape is OWASP ML Top 10 **ML02 Data Poisoning Attack** territory,
and mapping to NIST AI RMF `MAP 4.1` / `MEASURE 2.10` is how the acquire-stage
check justifies itself to auditors.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-01/
├── README.md
├── Makefile
├── requirements.txt
├── requirements.lock
├── selection_predicate.sql
├── code/
│   ├── 01_query_index.py
│   ├── 02_acquire_check.py
│   ├── 03_range_fetch_and_extract.py
│   ├── 04_language_id.py
│   ├── 05_emit_manifest.py
│   └── common.py
├── manifest/
│   ├── slice_manifest.json
│   ├── per_record_manifest.parquet
│   └── acquire_evidence/
│       └── {host_hash}.robotstxt
└── output/
    └── slice-00000.jsonl.gz
```

### 2.2 `selection_predicate.sql` — pin the slice

Pick one snapshot. The example uses `CC-MAIN-2024-30` (any snapshot that is at
least two weeks old is fine; the choice is pinned by the manifest).

```sql
-- selection_predicate.sql
-- Columnar CC index: s3://commoncrawl/cc-index/table/cc-main/warc/
-- Snapshot pinned for reproducibility.
WITH filtered AS (
  SELECT
    url,
    url_host_name,
    url_host_tld,
    warc_filename,
    warc_record_offset,
    warc_record_length,
    content_mime_type,
    content_languages,
    fetch_status
  FROM read_parquet(
    's3://commoncrawl/cc-index/table/cc-main/warc/crawl=CC-MAIN-2024-30/subset=warc/*.parquet'
  )
  WHERE fetch_status = 200
    AND content_mime_type = 'text/html'
    AND url_host_tld IN ('com', 'net', 'org', 'edu', 'gov')
    AND (content_languages IS NULL OR content_languages LIKE 'eng%')
)
SELECT *
FROM filtered
USING SAMPLE 150000 ROWS (reservoir, 20240730);  -- deterministic seed
```

Two things matter here. One: the seed in `USING SAMPLE` is the date of the run
and is persisted in the manifest — without it the slice is non-reproducible.
Two: the headroom (150k selected, ~100k target) is intentional, because the
acquire-stage filter and language-ID will each drop a fraction.

### 2.3 `code/01_query_index.py` — materialise the URL list

```python
# code/01_query_index.py
import duckdb
from pathlib import Path

SNAPSHOT = "CC-MAIN-2024-30"
OUT = Path("manifest/url_candidates.parquet")

def main() -> None:
    con = duckdb.connect()
    con.execute("INSTALL httpfs; LOAD httpfs;")
    con.execute("SET s3_region='us-east-1'; SET s3_use_ssl=true;")
    # Common Crawl's bucket is public; unsigned reads are enabled by default
    # when no AWS creds are configured. Verify against current CC docs.
    with open("selection_predicate.sql") as f:
        sql = f.read()
    con.execute(f"COPY ({sql}) TO '{OUT}' (FORMAT PARQUET)")
    n = con.execute(f"SELECT COUNT(*) FROM read_parquet('{OUT}')").fetchone()[0]
    print(f"[01] wrote {n} candidate URLs for {SNAPSHOT}")

if __name__ == "__main__":
    main()
```

### 2.4 `code/02_acquire_check.py` — the Chapter-06 opt-out check

One robots.txt fetch per host, cached by SHA-256, opt-out evaluated against the
AI-use bot list. The bot list evolves; verify against the current operator
pages (OpenAI `GPTBot`, Google `Google-Extended`, Common Crawl `CCBot`,
Anthropic `ClaudeBot` / `anthropic-ai`, Perplexity `PerplexityBot`, Apple
`Applebot-Extended`) at the time of the run.

```python
# code/02_acquire_check.py
import hashlib
import json
import time
from pathlib import Path
from urllib.parse import urlsplit

import httpx
import polars as pl
from reppy.robots import Robots  # verify library name at install time

AI_BOTS = [
    "GPTBot", "Google-Extended", "CCBot",
    "ClaudeBot", "anthropic-ai",
    "PerplexityBot", "Applebot-Extended",
]
EVIDENCE_DIR = Path("manifest/acquire_evidence")
EVIDENCE_DIR.mkdir(parents=True, exist_ok=True)

def host_hash(host: str) -> str:
    return hashlib.sha256(host.lower().encode()).hexdigest()[:16]

def fetch_robots(host: str) -> tuple[bytes, str, int]:
    url = f"https://{host}/robots.txt"
    try:
        r = httpx.get(url, timeout=10.0, follow_redirects=True,
                      headers={"User-Agent": "cc-slice-extractor/0.1 (+contact: you@example.org)"})
        return r.content, url, r.status_code
    except Exception:
        return b"", url, 0

def decide(body: bytes, target_url: str) -> dict:
    # Convention: fetch failure or non-200 => treat as "no robots.txt",
    # which under RFC 9309 is permissive. Document that choice here so
    # reviewers can see it.
    if not body:
        return {"allowed": True, "reason": "no-robots-or-fetch-failed"}
    try:
        robots = Robots.parse(target_url, body.decode("utf-8", errors="replace"))
    except Exception:
        return {"allowed": True, "reason": "robots-parse-failed"}
    blocked_by = [
        agent for agent in AI_BOTS
        if not robots.allowed(target_url, agent)
    ]
    return {
        "allowed": len(blocked_by) == 0,
        "reason": f"blocked-by:{','.join(blocked_by)}" if blocked_by else "allowed-by-robots",
        "blocked_by": blocked_by,
    }

def main() -> None:
    cands = pl.read_parquet("manifest/url_candidates.parquet")
    hosts = cands["url_host_name"].unique().to_list()
    print(f"[02] {len(hosts)} unique hosts to robots-check")

    host_evidence: dict[str, dict] = {}
    for i, host in enumerate(hosts):
        body, src_url, status = fetch_robots(host)
        h = host_hash(host)
        (EVIDENCE_DIR / f"{h}.robotstxt").write_bytes(body or b"")
        host_evidence[host] = {
            "host_hash": h,
            "evidence_sha256": hashlib.sha256(body).hexdigest() if body else "",
            "fetched_at": int(time.time()),
            "source_url": src_url,
            "http_status": status,
        }
        if i and i % 500 == 0:
            print(f"[02] robots fetched: {i}/{len(hosts)}")
        # polite: 1 request/sec/host; adjust if you have a shared pool.
        time.sleep(0.05)

    decisions = []
    for row in cands.iter_rows(named=True):
        host = row["url_host_name"]
        ev = host_evidence[host]
        body = (EVIDENCE_DIR / f"{ev['host_hash']}.robotstxt").read_bytes()
        d = decide(body, row["url"])
        decisions.append({
            "url": row["url"],
            "host": host,
            "warc_filename": row["warc_filename"],
            "warc_record_offset": row["warc_record_offset"],
            "warc_record_length": row["warc_record_length"],
            "acquire_allowed": d["allowed"],
            "acquire_reason": d["reason"],
            "evidence_sha256": ev["evidence_sha256"],
            "evidence_fetched_at": ev["fetched_at"],
        })
    pl.DataFrame(decisions).write_parquet("manifest/acquire_decisions.parquet")
    kept = sum(1 for d in decisions if d["acquire_allowed"])
    print(f"[02] kept {kept}/{len(decisions)} after acquire-stage check")

if __name__ == "__main__":
    main()
```

Design notes worth noting:

- The decision record carries the evidence SHA-256 and the fetch timestamp,
  which is what makes the slice **independently reverifiable**: a reviewer
  re-fetches `https://{host}/robots.txt`, hashes it, and compares.
- Fetch failure defaults to **allow** (RFC 9309 — no robots.txt means no
  restriction). This choice must be written in the README — it is a policy
  decision, not a bug. A stricter default (fail-closed) is also defensible but
  trades reproducibility for conservatism; pick one and document it.
- The polite rate limit is per-host; a shared-pool reviewer will want a
  stricter cap.

### 2.5 `code/03_range_fetch_and_extract.py` — range-GET the WARCs

```python
# code/03_range_fetch_and_extract.py
import gzip
import hashlib
import io
import json
from pathlib import Path

import httpx
import polars as pl
from bs4 import BeautifulSoup
from warcio.archiveiterator import ArchiveIterator

OUT = Path("manifest/extracted_docs.parquet")
CC_BASE = "https://data.commoncrawl.org"

def warc_url(warc_filename: str) -> str:
    return f"{CC_BASE}/{warc_filename}"

def fetch_record(client: httpx.Client, url: str, offset: int, length: int) -> bytes:
    headers = {"Range": f"bytes={offset}-{offset + length - 1}"}
    r = client.get(url, headers=headers, timeout=30.0)
    r.raise_for_status()
    return r.content

def extract(html: bytes) -> str:
    # Chapter 03 extractor benchmark lives in Exercise 02; here we use a
    # deliberately simple BeautifulSoup pass so the language-ID stage has
    # text to score. Do NOT treat this as the production extractor.
    soup = BeautifulSoup(html, "lxml")
    for tag in soup(["script", "style", "noscript"]):
        tag.decompose()
    return " ".join(soup.get_text(separator=" ").split())

def main() -> None:
    dec = pl.read_parquet("manifest/acquire_decisions.parquet")
    kept = dec.filter(pl.col("acquire_allowed"))
    print(f"[03] range-fetching {len(kept)} WARC records")

    rows = []
    with httpx.Client(http2=True) as client:
        for i, row in enumerate(kept.iter_rows(named=True)):
            try:
                blob = fetch_record(
                    client,
                    warc_url(row["warc_filename"]),
                    row["warc_record_offset"],
                    row["warc_record_length"],
                )
                it = ArchiveIterator(io.BytesIO(blob))
                rec = next(it)
                body = rec.content_stream().read()
                obs_sha = hashlib.sha256(body).hexdigest()
                text = extract(body)
                warc_record_id = rec.rec_headers.get_header("WARC-Record-ID")
            except Exception as e:
                # Record the failure in the manifest rather than dropping silently;
                # mod-104 data-quality cares about miss rates.
                rows.append({**row, "observed_sha256": "", "text": "",
                             "warc_record_id": "", "extract_error": str(e)})
                continue
            rows.append({**row, "observed_sha256": obs_sha, "text": text,
                         "warc_record_id": warc_record_id, "extract_error": ""})
            if i and i % 1000 == 0:
                print(f"[03] fetched {i}/{len(kept)}")
    pl.DataFrame(rows).write_parquet(OUT)

if __name__ == "__main__":
    main()
```

### 2.6 `code/04_language_id.py` — FastText `lid.176`

```python
# code/04_language_id.py
import gzip
import hashlib
import json
from pathlib import Path

import fasttext
import polars as pl

LID_MODEL = Path("models/lid.176.bin")
EXPECTED_LID_SHA = "<fill in after download>"  # record in README
THRESHOLD = 0.5
TARGET_LANG = "en"
OUT_SHARD = Path("output/slice-00000.jsonl.gz")
OUT_MANIFEST = Path("manifest/per_record_manifest.parquet")

def main() -> None:
    assert LID_MODEL.exists(), "download lid.176.bin first"
    observed_sha = hashlib.sha256(LID_MODEL.read_bytes()).hexdigest()
    if EXPECTED_LID_SHA and observed_sha != EXPECTED_LID_SHA:
        raise SystemExit(f"lid.176.bin SHA mismatch: {observed_sha}")
    model = fasttext.load_model(str(LID_MODEL))

    docs = pl.read_parquet("manifest/extracted_docs.parquet")
    kept, total_lang_rows = 0, []
    OUT_SHARD.parent.mkdir(parents=True, exist_ok=True)
    with gzip.open(OUT_SHARD, "wt", encoding="utf-8") as fh:
        for row in docs.iter_rows(named=True):
            text = (row.get("text") or "")[:10_000]  # fasttext expects single-line
            if not text:
                total_lang_rows.append({**row, "language": "", "language_score": 0.0})
                continue
            labels, scores = model.predict(text.replace("\n", " "), k=1)
            lang = labels[0].replace("__label__", "")
            score = float(scores[0])
            total_lang_rows.append({**row, "language": lang, "language_score": score})
            if lang == TARGET_LANG and score >= THRESHOLD:
                kept += 1
                fh.write(json.dumps({
                    "text": row["text"],
                    "url": row["url"],
                    "warc_record_id": row["warc_record_id"],
                }, ensure_ascii=False) + "\n")

    pl.DataFrame(total_lang_rows).write_parquet(OUT_MANIFEST)
    print(f"[04] kept {kept} English docs at threshold {THRESHOLD}")

if __name__ == "__main__":
    main()
```

The `lid.176.bin` SHA is pinned in the README. FastText language-ID (`lid.176`)
is distributed by the FastText project; the download URL is in the exercise
prerequisites.

### 2.7 `code/05_emit_manifest.py` — the Chapter-07 slice manifest

```python
# code/05_emit_manifest.py
import datetime as dt
import hashlib
import json
from pathlib import Path

import polars as pl

OUT = Path("manifest/slice_manifest.json")

def sha256_file(p: Path) -> str:
    return hashlib.sha256(p.read_bytes()).hexdigest()

def main() -> None:
    per_record = pl.read_parquet("manifest/per_record_manifest.parquet")
    kept = per_record.filter(
        (pl.col("language") == "en") & (pl.col("language_score") >= 0.5)
    )
    manifest = {
        "slice_name": "slice-cc-en-small-v1",
        "slice_version": "1.0.0",
        "snapshot_id": "CC-MAIN-2024-30",
        "source_selection": {
            "predicate_sha256": sha256_file(Path("selection_predicate.sql")),
            "sample_rows": 150000,
            "sample_seed": 20240730,
        },
        "acquire_stage": {
            "policy": "block if any of [GPTBot, Google-Extended, CCBot, ClaudeBot, anthropic-ai, PerplexityBot, Applebot-Extended] is disallowed",
            "evidence_dir": "manifest/acquire_evidence/",
            "fail_open_on_missing_robots_txt": True,
        },
        "content_stage": {
            "language_id": {
                "model": "fasttext/lid.176",
                "model_sha256": sha256_file(Path("models/lid.176.bin")),
                "threshold": 0.5,
                "target_language": "en",
            },
            "extractor": {
                "name": "beautifulsoup4+lxml",
                "note": "stand-in for Exercise 02; not a production extractor",
            },
        },
        "counts": {
            "candidates": int(pl.read_parquet("manifest/url_candidates.parquet").height),
            "after_acquire_check": int(per_record.height),
            "after_language_id": int(kept.height),
        },
        "shards": [
            {
                "path": "output/slice-00000.jsonl.gz",
                "sha256": sha256_file(Path("output/slice-00000.jsonl.gz")),
                "records": int(kept.height),
            }
        ],
        "built_at": dt.datetime.utcnow().replace(microsecond=0).isoformat() + "Z",
        "builder": {"user": "<fill>", "git_sha": "<fill via git rev-parse HEAD>"},
    }
    OUT.write_text(json.dumps(manifest, indent=2, sort_keys=True) + "\n")
    print(f"[05] wrote {OUT}")

if __name__ == "__main__":
    main()
```

### 2.8 `Makefile`

```makefile
.PHONY: all reproduce clean lid-model
all: reproduce

lid-model:
	@mkdir -p models
	@test -f models/lid.176.bin || \
	  curl -fL -o models/lid.176.bin \
	    https://dl.fbaipublicfiles.com/fasttext/supervised-models/lid.176.bin
	@sha256sum models/lid.176.bin > models/lid.176.bin.sha256

reproduce: lid-model
	python code/01_query_index.py
	python code/02_acquire_check.py
	python code/03_range_fetch_and_extract.py
	python code/04_language_id.py
	python code/05_emit_manifest.py

clean:
	rm -rf manifest/url_candidates.parquet manifest/acquire_decisions.parquet \
	       manifest/extracted_docs.parquet manifest/per_record_manifest.parquet \
	       manifest/slice_manifest.json output/*.jsonl.gz
```

### 2.9 `README.md` (excerpt)

Record the following pinned facts; without them the slice is not reproducible:

- Snapshot ID (e.g. `CC-MAIN-2024-30`).
- SHA-256 of `lid.176.bin` (observed at download).
- `pip freeze > requirements.lock` output.
- Git SHA of this repository.
- The default on missing / unparseable `robots.txt` (fail-open here, following
  RFC 9309's "no robots.txt = no restriction" reading).

## 3. Validation steps

1. **The predicate is pinned.** `SHA256(selection_predicate.sql)` appears in the
   slice manifest, and the file does not contain `NOW()` / `CURRENT_DATE` /
   unseeded `RANDOM()`.
2. **Count envelope.** Final English document count is 80k–120k. If the count
   is far outside that range, either the snapshot is unusual or an operator is
   misconfigured — the README should say which.
3. **`evidence_sha256` is reverifiable.** Pick one host at random, re-fetch
   `https://{host}/robots.txt`, hash it, and confirm a match. One mismatch is
   not fatal (robots can change) but every host should have *some* valid
   hash-at-fetch-time recorded.
4. **Per-record manifest covers every candidate.** `len(per_record_manifest)` ==
   `len(url_candidates)`. A dropped record must still appear in the manifest
   with `acquire_allowed=false` or `language != en` recorded — silent drops are
   the failure mode mod-104 cares about.
5. **The extractor is marked as a stand-in.** The manifest's `extractor.note`
   makes it unambiguous that this is not Exercise 02's benchmark result. If a
   reader could read the manifest and think the extraction quality was proven,
   the manifest is lying.
6. **Reproducibility check.** `make clean && make reproduce` on a fresh clone
   reproduces a per-record manifest with identical `observed_sha256` values.
   (Note: the acquire-stage check and shard counts will fluctuate if robots.txt
   or the snapshot change; that's a fact about the world, not a bug — it's why
   the manifest records the timestamp.)
7. **No invented metrics.** If `requirements.lock`, the snapshot, or the
   `lid.176.bin` SHA was not actually captured, the README must say so with
   `<!-- unverified: ... -->` — never guess.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 = defensible). A
passing solution scores ≥ 18/26 and loses zero points on rows marked
**critical**.

| # | Dimension                                                                               | Critical? |
|---|-----------------------------------------------------------------------------------------|-----------|
| 1 | Snapshot ID, seed, and predicate SHA are pinned in the manifest                        | Yes       |
| 2 | Acquire-stage decision exists for every URL, with evidence SHA-256 + timestamp         | Yes       |
| 3 | At least one `evidence_sha256` is independently reverifiable at review time            | Yes       |
| 4 | Fail-open vs fail-closed default for missing / unparseable `robots.txt` is documented  | Yes       |
| 5 | Per-record manifest contains every candidate, not just survivors (no silent drops)     | Yes       |
| 6 | `lid.176.bin` SHA-256 is pinned in README and manifest; model path is not hard-coded   | Yes       |
| 7 | Extractor used is identified AND flagged as a stand-in for Exercise 02                 | No        |
| 8 | `make reproduce` rebuilds the slice from the pinned predicate + code                   | Yes       |
| 9 | Language threshold (0.5) is documented and used consistently                            | No        |
| 10 | `requirements.lock` captures package versions                                           | No        |
| 11 | Unverifiable claims use `<!-- unverified: ... -->` instead of guesses              | Yes       |
| 12 | Governance anchor cited (OWASP ML Top 10 / NIST AI RMF / MITRE ATLAS)                  | No        |
| 13 | README names the `User-Agent` used for robots.txt fetches                              | No        |

## 5. Common mistakes

- **Treating the WET files as "already extracted".** Common Crawl's WET format
  is a baseline, not a quality extractor; mod-103's whole point is that the
  WET pass is one of the weakest options. Range-fetch WARC and extract
  yourself. (Chapter 03 covers this in depth; Exercise 02 benchmarks it.)
- **No seed in the sample.** `USING SAMPLE N ROWS` without a seed produces a
  different slice every run. The manifest records the seed.
- **One robots.txt fetch per URL instead of per host.** This is a self-DoS and
  a reputation risk. Group by host first.
- **Silent drops of fetch failures.** Every candidate URL must still appear in
  the per-record manifest with a reason; otherwise mod-104's audit cannot
  compute miss rates.
- **Hard-coded `User-Agent: Mozilla`.** The acquire-stage check is a policy
  statement; impersonating a browser to fetch robots.txt is incoherent with
  the policy and will mislead the host operator. Use a project-identifying UA
  with a contact.
- **Confusing the snapshot's `content_languages` field with a real language-ID
  result.** The CC field is cheap and often wrong (it comes from HTTP /
  `<html lang>` hints). You still have to run `lid.176` to produce a defensible
  language label.
- **Confusing the exercise with the full CCNet.** The KenLM perplexity stage,
  the MinHash dedup stage, and the decontamination stage are explicitly out of
  scope here — they live in Exercise 03 and mod-105 / mod-106.
- **Writing the manifest with a wall-clock timestamp but no git SHA.** The
  timestamp drifts; the git SHA + predicate SHA + model SHA are what actually
  pin the build.

## 6. References

- **Common Crawl columnar index documentation** — <https://commoncrawl.org/get-started>
  and the index Parquet layout under
  `s3://commoncrawl/cc-index/table/cc-main/warc/`.
- **FastText `lid.176` language-ID model** —
  <https://fasttext.cc/docs/en/language-identification.html>. Download URL and
  model card are on that page.
- **RFC 9309 (Robots Exclusion Protocol)** —
  <https://www.rfc-editor.org/rfc/rfc9309.html>. The "no robots.txt = no
  restriction" default comes from §2.5.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02
  (Data Poisoning Attack) anchors why an un-audited scrape is not neutral.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. Browse the current matrix for
  the technique IDs under "Poison Training Data" at the time of the run.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MAP 4.1` grounds
  the acquire-stage check as a governance obligation, not a nice-to-have.
- **Local anchors:** Chapter 06 (crawling ethics in code) for the acquire-stage
  policy; Chapter 07 for the slice manifest schema; Exercise 02 for the
  extractor benchmark that justifies swapping out BeautifulSoup for a real
  extractor in production.
