# Exercise 05 — Regression Fixture Authoring for One Pipeline (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner picks one concrete
pipeline, pins a small hand-curated set of input/expected pairs as
golden fixtures, authors a pytest suite that re-runs the pipeline
against every fixture, and authors a second suite of invariant
assertions that do not depend on exact output bytes. The fixture pack,
the two test suites, and a short policy document together form a
tripwire: any future change to the pipeline is detected deterministically
as either a golden diff or an invariant violation.

Running example throughout: **`corpus-clean`**, a text-cleaning
pipeline that reads a JSONL shard, applies Unicode NFC normalisation,
strips boilerplate, deduplicates by near-identical hash, and emits a
Parquet table with one row per surviving document. The fixture pack
below is deliberately concrete to `corpus-clean`; the structure
generalises.

Deliverables (taken verbatim from the exercise contract):

- `fixtures/inputs/` — pinned input slices across happy_path,
  boundary, unicode, historical, adversarial. Each file has a sidecar
  `.sha256`.
- `fixtures/expected/` — expected-output Parquet per input. Each file
  has a sidecar `.sha256`.
- `fixtures/catalog.yaml` — the index: every pair's purpose,
  invariants, owner, `last_touched_commit`, `added_in_ticket`.
- `fixtures/quarantine.yaml` — flaky fixtures with expiry.
- `tests/test_golden_outputs.py` — runs the pipeline and asserts
  logical equality (canonical-sorted) against the golden.
- `tests/test_invariants.py` — property assertions: schema conformance,
  row-count bounds, monotonic timestamps, join-key uniqueness,
  null-rate ceilings. Great Expectations + `hypothesis` back the
  heavier checks.
- `scripts/regen_goldens.py` — regeneration gate: non-empty
  `--justification`, named `--reviewer`, refuses a dirty tree, writes
  a per-fixture CSV diff into `CHANGELOG_GOLDENS.md`.
- `docs/FIXTURE_POLICY.md` — when to add, when to regen, size budget,
  quarantine, dead-fixture detection, ownership.

Governance framing. A regression-fixture suite is an operational
tripwire under a data pipeline. The primary OWASP ML Top 10 anchor is
**ML02:2023 Data Poisoning Attack** — a silent upstream change (an
operator loosens a dedupe threshold, a library upgrade changes the
Unicode normaliser) is a self-inflicted poisoning vector that only
surfaces downstream; a golden suite catches it at the earliest
deterministic point. The MITRE ATLAS anchor is the **Impact tactic**
at the pillar level; the specific AML.T* identifier that most cleanly
covers "silent upstream pipeline change corrupts training data" is a
reviewer-owned follow-up and should be pinned once ATLAS adds the
matching technique.
The NIST AI RMF anchors are the **MEASURE** pillar (validity and
reliability evaluation — what the invariant suite operationalises)
and the **MANAGE** pillar (if the suite goes red in CI, the pipeline
is held out of production — the deactivation posture). Framework URLs
are in §6.

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-05/
├── README.md
├── requirements.lock
├── Makefile
├── pyproject.toml
├── src/corpus_clean/
│   ├── pipeline.py                  # pipeline under test; version-pinned
│   └── canonical.py                 # canonical-sort helper (pipeline+tests)
├── fixtures/
│   ├── catalog.yaml                 # index of truth; see §2.2
│   ├── quarantine.yaml              # flaky fixtures w/ expiry
│   ├── inputs/
│   │   ├── happy_path/{001_minimal_en,002_multisource_en}.jsonl(+.sha256)
│   │   ├── boundary/{010_empty_text,011_max_text_bytes,
│   │   │              012_null_optional_fields}.jsonl(+.sha256)
│   │   ├── unicode/{020_nfc_vs_nfd,021_emoji_zwj,
│   │   │             022_rtl_arabic_hebrew}.jsonl(+.sha256)
│   │   ├── historical/{030_ticket_DQ-401_dedupe_stable,
│   │   │                031_ticket_DQ-507_tz_edge}.jsonl(+.sha256)
│   │   └── adversarial/{040_prompt_injection_shaped,
│   │                     041_malformed_json_trailing,
│   │                     042_tz_boundary_dst}.jsonl(+.sha256)
│   └── expected/                    # parallel tree of .parquet (+.sha256)
├── tests/
│   ├── conftest.py
│   ├── test_golden_outputs.py
│   ├── test_invariants.py
│   └── test_catalog_integrity.py
├── scripts/
│   ├── regen_goldens.py
│   └── seal_fixture.py              # recompute sidecar .sha256 after edits
├── docs/
│   ├── FIXTURE_POLICY.md
│   └── CHANGELOG_GOLDENS.md         # written by regen_goldens.py
└── ci/
    └── dead_fixture_check.py        # warn-only CI job; see §2.9
```

### 2.2 `fixtures/catalog.yaml`

```yaml
# fixtures/catalog.yaml
# One entry per fixture pair. The catalog is the index of truth.
# Tests enumerate fixtures from here, not from filesystem globs.
# Enforced by tests/test_catalog_integrity.py.

pipeline:
  name: corpus-clean
  version: 1.4.2
  entrypoint: corpus_clean.pipeline:run
  canonical_sort_keys: [doc_id, source]

defaults:
  owner: data-quality@team.example
  invariants: [schema_conformance, row_count_bounds,
               unique_join_keys, null_rate_ceilings]

fixtures:
  # ---- happy_path ---------------------------------------------------------
  - id: happy_path/001_minimal_en
    purpose: >
      Three English docs, no duplicates, no boilerplate. Pins baseline
      output schema and canonical doc_id ordering.
    input:    fixtures/inputs/happy_path/001_minimal_en.jsonl
    expected: fixtures/expected/happy_path/001_minimal_en.parquet
    invariants:
      - schema_conformance
      - row_count_bounds: { min: 3, max: 3 }
      - unique_join_keys: [doc_id]
      - monotonic_timestamps: ingested_at
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-100

  - id: happy_path/002_multisource_en
    purpose: Three sources interleaved; fixes (doc_id, source) sort.
    input:    fixtures/inputs/happy_path/002_multisource_en.jsonl
    expected: fixtures/expected/happy_path/002_multisource_en.parquet
    invariants:
      - schema_conformance
      - unique_join_keys: [doc_id, source]
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-100

  # ---- boundary -----------------------------------------------------------
  - id: boundary/010_empty_text
    purpose: Empty `text` field. Must drop, not crash.
    input:    fixtures/inputs/boundary/010_empty_text.jsonl
    expected: fixtures/expected/boundary/010_empty_text.parquet
    invariants:
      - schema_conformance
      - row_count_bounds: { min: 0, max: 0 }
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-212

  - id: boundary/011_max_text_bytes
    purpose: >
      One doc at the `max_text_bytes` ceiling (512 KiB). Pins truncation:
      chunk, do not drop; chunk indices are monotonic.
    input:    fixtures/inputs/boundary/011_max_text_bytes.jsonl
    expected: fixtures/expected/boundary/011_max_text_bytes.parquet
    invariants:
      - schema_conformance
      - row_count_bounds: { min: 1, max: 4 }
      - monotonic_timestamps: ingested_at
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-212

  - id: boundary/012_null_optional_fields
    purpose: Null in every nullable column. Pins null-rate ceilings.
    input:    fixtures/inputs/boundary/012_null_optional_fields.jsonl
    expected: fixtures/expected/boundary/012_null_optional_fields.parquet
    invariants:
      - schema_conformance
      - null_rate_ceilings: { language: 1.0, author: 1.0, section: 1.0 }
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-212

  # ---- unicode ------------------------------------------------------------
  - id: unicode/020_nfc_vs_nfd
    purpose: NFC and NFD encoding of same visible string must dedupe to 1.
    input:    fixtures/inputs/unicode/020_nfc_vs_nfd.jsonl
    expected: fixtures/expected/unicode/020_nfc_vs_nfd.parquet
    invariants:
      - schema_conformance
      - row_count_bounds: { min: 1, max: 1 }
      - unique_join_keys: [doc_id]
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-305

  - id: unicode/021_emoji_zwj
    purpose: ZWJ-joined emoji sequences not split by whitespace collapser.
    input:    fixtures/inputs/unicode/021_emoji_zwj.jsonl
    expected: fixtures/expected/unicode/021_emoji_zwj.parquet
    invariants: [schema_conformance]
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-305

  - id: unicode/022_rtl_arabic_hebrew
    purpose: RTL strings not reordered.
    input:    fixtures/inputs/unicode/022_rtl_arabic_hebrew.jsonl
    expected: fixtures/expected/unicode/022_rtl_arabic_hebrew.parquet
    invariants: [schema_conformance]
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-305

  # ---- historical (one row per past bug) ----------------------------------
  - id: historical/030_ticket_DQ-401_dedupe_stable
    purpose: >
      DQ-401: dedupe winner is deterministic (lex-smallest source,doc_id).
    input:    fixtures/inputs/historical/030_ticket_DQ-401_dedupe_stable.jsonl
    expected: fixtures/expected/historical/030_ticket_DQ-401_dedupe_stable.parquet
    invariants:
      - schema_conformance
      - unique_join_keys: [doc_id]
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-401

  - id: historical/031_ticket_DQ-507_tz_edge
    purpose: >
      DQ-507: `ingested_at` is UTC; +05:30 source is converted.
    input:    fixtures/inputs/historical/031_ticket_DQ-507_tz_edge.jsonl
    expected: fixtures/expected/historical/031_ticket_DQ-507_tz_edge.parquet
    invariants:
      - schema_conformance
      - monotonic_timestamps: ingested_at
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-507

  # ---- adversarial --------------------------------------------------------
  - id: adversarial/040_prompt_injection_shaped
    purpose: Prompt-injection text treated as ordinary text, no leakage.
    input:    fixtures/inputs/adversarial/040_prompt_injection_shaped.jsonl
    expected: fixtures/expected/adversarial/040_prompt_injection_shaped.parquet
    invariants:
      - schema_conformance
      - null_rate_ceilings: { author: 1.0 }
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-612

  - id: adversarial/041_malformed_json_trailing
    purpose: Malformed last record routed to error channel; main has N-1.
    input:    fixtures/inputs/adversarial/041_malformed_json_trailing.jsonl
    expected: fixtures/expected/adversarial/041_malformed_json_trailing.parquet
    invariants:
      - schema_conformance
      - row_count_bounds: { min: 2, max: 2 }
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-612

  - id: adversarial/042_tz_boundary_dst
    purpose: Europe/London DST spring-forward; no ambiguous local time.
    input:    fixtures/inputs/adversarial/042_tz_boundary_dst.jsonl
    expected: fixtures/expected/adversarial/042_tz_boundary_dst.parquet
    invariants:
      - schema_conformance
      - monotonic_timestamps: ingested_at
    last_touched_commit: a70cd5a
    added_in_ticket: DQ-612
```

Why each field matters at review:

- **`pipeline.version` pinned in the catalog.** Fixture pack is valid
  against one version; a library bump that moves `corpus_clean.__version__`
  without a golden regen is a review-blocking event.
- **`canonical_sort_keys` pinned globally, not per fixture.** Every
  fixture must sort the same way or logical equality becomes unreliable.
- **`last_touched_commit` enables dead-fixture detection** via §2.9's CI
  job.
- **`added_in_ticket` recovers the "why"** six months later.

### 2.3 Fixture file conventions

A fixture pair is three files on disk:

- `fixtures/inputs/<category>/<nnn_slug>.jsonl` — the input shard.
- `fixtures/inputs/<category>/<nnn_slug>.jsonl.sha256` — sidecar
  hash, one line, `<64-hex>  <filename>`, the exact format `sha256sum`
  emits.
- `fixtures/expected/<category>/<nnn_slug>.parquet` — the golden
  output.
- `fixtures/expected/<category>/<nnn_slug>.parquet.sha256` — sidecar.

Rules the fixture pack obeys, enforced by `tests/test_catalog_integrity.py`
in §2.6:

- Every `fixtures/inputs/**/*.jsonl` has a sibling `.sha256`.
- Every input has exactly one matching entry in `catalog.yaml`.
- Every catalog entry references a file that exists on disk.
- No fixture file exceeds the per-fixture budget (16 KiB input,
  64 KiB expected output, documented in `docs/FIXTURE_POLICY.md`).
- The aggregate size of `fixtures/` does not exceed the pack budget
  (2 MiB).

Example of a hand-curated input (`happy_path/001_minimal_en.jsonl`):

```json
{"doc_id": "d-0001", "source": "wiki",  "language": "en", "text": "The cat sat on the mat.", "ingested_at": "2025-01-02T10:00:00Z"}
{"doc_id": "d-0002", "source": "news",  "language": "en", "text": "Tokyo opened a new line.", "ingested_at": "2025-01-02T10:00:01Z"}
{"doc_id": "d-0003", "source": "forum", "language": "en", "text": "I made soup.",           "ingested_at": "2025-01-02T10:00:02Z"}
```

### 2.4 `src/corpus_clean/canonical.py`

```python
# src/corpus_clean/canonical.py
# Canonical-sort helper. Used by pipeline AND test harness; keeping this
# in one module is what makes logical equality honest.
from __future__ import annotations
import pyarrow as pa, pyarrow.compute as pc


def canonical_sort(table: pa.Table, keys: list[str]) -> pa.Table:
    """Sort `table` by `keys` ascending. Canonical form = pinned column
    order (from the pipeline's OUTPUT_SCHEMA) + ascending sort on `keys`
    + pinned writer options (see test_golden_outputs.PARQUET_WRITE)."""
    missing = [k for k in keys if k not in table.column_names]
    if missing:
        raise ValueError(f"canonical_sort: missing keys {missing}")
    idx = pc.sort_indices(table, sort_keys=[(k, "ascending") for k in keys])
    return table.take(idx)
```

### 2.5 `tests/test_golden_outputs.py`

```python
# tests/test_golden_outputs.py
# Runs corpus-clean on each input fixture; asserts logical equality
# (canonical-sorted) against the golden parquet.
from __future__ import annotations

import hashlib
from pathlib import Path

import pyarrow.parquet as pq
import pytest
import yaml

from corpus_clean.canonical import canonical_sort
from corpus_clean.pipeline import run as run_pipeline

REPO_ROOT = Path(__file__).resolve().parents[1]
CATALOG = yaml.safe_load((REPO_ROOT / "fixtures/catalog.yaml").read_text())
SORT_KEYS: list[str] = CATALOG["pipeline"]["canonical_sort_keys"]

# Pinned Parquet writer options. Writer-option drift IS a regression.
PARQUET_WRITE = dict(compression="zstd", compression_level=3,
                     use_dictionary=False, write_statistics=False,
                     row_group_size=1024, version="2.6")


def _sha256(p):   return hashlib.sha256(p.read_bytes()).hexdigest()
def _sidecar(p):  return (p.with_suffix(p.suffix + ".sha256")
                            .read_text().split()[0].strip())


def _quarantined() -> set[str]:
    qf = REPO_ROOT / "fixtures/quarantine.yaml"
    if not qf.exists():
        return set()
    return {e["id"] for e in (yaml.safe_load(qf.read_text()) or {})
                               .get("entries", [])}


QUARANTINED = _quarantined()


@pytest.mark.parametrize(
    "entry",
    [e for e in CATALOG["fixtures"] if e["id"] not in QUARANTINED],
    ids=lambda e: e["id"],
)
def test_golden_output(entry, tmp_path):
    inp = REPO_ROOT / entry["input"]
    exp = REPO_ROOT / entry["expected"]

    # 1) sidecar integrity on both input and expected.
    assert _sha256(inp) == _sidecar(inp), (
        f"input sidecar drift for {entry['id']}; re-seal or fix input"
    )
    assert _sha256(exp) == _sidecar(exp), (
        f"expected sidecar drift for {entry['id']}; regen required"
    )

    # 2) run pipeline, canonical-sort both tables.
    out = tmp_path / "out.parquet"
    run_pipeline(input_path=inp, output_path=out)
    actual   = canonical_sort(pq.read_table(out), SORT_KEYS)
    expected = canonical_sort(pq.read_table(exp), SORT_KEYS)

    # 3) schema strictly, then rows cell-by-cell.
    assert actual.schema.equals(expected.schema, check_metadata=False), (
        f"schema drift for {entry['id']}"
    )
    assert actual.equals(expected), (
        f"row diff for {entry['id']}. Accept via "
        f"scripts/regen_goldens.py --justification '<ticket>'."
    )
```

Design notes defensible at review:

- **Byte-identity vs canonical-sort.** Raw Parquet byte equality flakes
  on row order, dictionary encoding, and row-group sizing. The suite
  canonical-sorts and compares at the logical `pa.Table.equals` level
  with writer options pinned. One honest test beats two dishonest ones.
- **Sidecars checked before running the pipeline.** A silent edit to
  input or golden must fail loudly, not pass "mysteriously".
- **Quarantine is a set, not an `xfail`.** A fixture either runs or it
  does not; conditional xfail masks regressions.
- **Catalog, not filesystem, drives parametrisation.** A stray file
  under `fixtures/inputs/` with no catalog entry fails
  `test_catalog_integrity.py`, not silently.

### 2.6 `tests/test_catalog_integrity.py`

```python
# tests/test_catalog_integrity.py
# The catalog is the index of truth; the filesystem must agree with it.
from __future__ import annotations

from datetime import date
from pathlib import Path

import yaml

REPO_ROOT = Path(__file__).resolve().parents[1]
FIXTURE_ROOT = REPO_ROOT / "fixtures"
CATALOG = yaml.safe_load((FIXTURE_ROOT / "catalog.yaml").read_text())
PER_INPUT_MAX, PER_EXPECTED_MAX, PACK_MAX = 16 * 1024, 64 * 1024, 2 * 1024**2


def _declared():
    return {REPO_ROOT / e[k] for e in CATALOG["fixtures"]
            for k in ("input", "expected")}


def test_every_input_has_sidecar():
    for p in (FIXTURE_ROOT / "inputs").rglob("*.jsonl"):
        assert p.with_suffix(".jsonl.sha256").exists(), f"missing: {p}"


def test_every_expected_has_sidecar():
    for p in (FIXTURE_ROOT / "expected").rglob("*.parquet"):
        assert p.with_suffix(".parquet.sha256").exists(), f"missing: {p}"


def test_no_filesystem_orphans():
    seen = set((FIXTURE_ROOT / "inputs").rglob("*.jsonl")) \
         | set((FIXTURE_ROOT / "expected").rglob("*.parquet"))
    orphans = seen - _declared()
    assert not orphans, f"not in catalog.yaml: {sorted(orphans)}"


def test_every_catalog_entry_exists_on_disk():
    for p in _declared():
        assert p.exists(), f"catalog references missing file: {p}"


def test_fixture_size_budget():
    total = 0
    for p in (FIXTURE_ROOT / "inputs").rglob("*.jsonl"):
        assert p.stat().st_size <= PER_INPUT_MAX, f"over budget: {p}"
        total += p.stat().st_size
    for p in (FIXTURE_ROOT / "expected").rglob("*.parquet"):
        assert p.stat().st_size <= PER_EXPECTED_MAX, f"over budget: {p}"
        total += p.stat().st_size
    assert total <= PACK_MAX, f"pack {total}B > budget {PACK_MAX}B"


def test_quarantine_entries_have_future_expiry():
    qf = FIXTURE_ROOT / "quarantine.yaml"
    if not qf.exists():
        return
    for e in (yaml.safe_load(qf.read_text()) or {}).get("entries", []):
        assert "expires_on" in e, f"{e.get('id')} missing expires_on"
        exp = e["expires_on"]
        if isinstance(exp, str):
            exp = date.fromisoformat(exp)
        assert exp >= date.today(), (
            f"quarantine entry {e['id']} expired on {e['expires_on']}"
        )
```

### 2.7 `tests/test_invariants.py`

```python
# tests/test_invariants.py
# Property-style assertions that do not depend on exact output bytes.
# Every fixture declaring an invariant in catalog.yaml must pass it.
from __future__ import annotations

from pathlib import Path

import great_expectations as gx
import pyarrow.parquet as pq
import pytest
import yaml
from hypothesis import given, settings, strategies as st

from corpus_clean.pipeline import run as run_pipeline

REPO_ROOT = Path(__file__).resolve().parents[1]
CATALOG = yaml.safe_load((REPO_ROOT / "fixtures/catalog.yaml").read_text())

# Pinned output schema the pipeline is contractually required to emit.
EXPECTED_SCHEMA = {
    "doc_id":      "string",
    "source":      "string",
    "language":    "string",
    "text":        "string",
    "ingested_at": "timestamp[us, tz=UTC]",
    "chunk_index": "int32",
    "row_hash":    "string",
}


def _invariant(entry, key):
    """Extract an invariant payload from a catalog entry, or None."""
    for i in entry.get("invariants", []):
        if isinstance(i, dict) and key in i:
            return i[key]
    return None


def _run(entry, tmp_path):
    out = tmp_path / "out.parquet"
    run_pipeline(input_path=REPO_ROOT / entry["input"], output_path=out)
    return out


@pytest.mark.parametrize("entry", CATALOG["fixtures"], ids=lambda e: e["id"])
def test_schema_conformance(entry, tmp_path):
    out = _run(entry, tmp_path)
    actual = {f.name: str(f.type) for f in pq.read_schema(out)}
    assert actual == EXPECTED_SCHEMA, f"schema drift on {entry['id']}"


@pytest.mark.parametrize(
    "entry",
    [e for e in CATALOG["fixtures"] if _invariant(e, "row_count_bounds")],
    ids=lambda e: e["id"],
)
def test_row_count_bounds(entry, tmp_path):
    b = _invariant(entry, "row_count_bounds")
    n = pq.read_metadata(_run(entry, tmp_path)).num_rows
    assert b["min"] <= n <= b["max"], (
        f"{entry['id']}: row count {n} not in [{b['min']}, {b['max']}]"
    )


@pytest.mark.parametrize(
    "entry",
    [e for e in CATALOG["fixtures"] if _invariant(e, "monotonic_timestamps")],
    ids=lambda e: e["id"],
)
def test_monotonic_timestamps(entry, tmp_path):
    col = _invariant(entry, "monotonic_timestamps")
    df = pq.read_table(_run(entry, tmp_path)).to_pandas()
    df = df.sort_values(["doc_id", "chunk_index"])
    ts = df[col].tolist()
    assert all(ts[i] <= ts[i + 1] for i in range(len(ts) - 1)), (
        f"{entry['id']}: `{col}` not monotonic within (doc_id, chunk_index)"
    )


@pytest.mark.parametrize(
    "entry",
    [e for e in CATALOG["fixtures"] if _invariant(e, "unique_join_keys")],
    ids=lambda e: e["id"],
)
def test_unique_join_keys(entry, tmp_path):
    keys = _invariant(entry, "unique_join_keys")
    df = pq.read_table(_run(entry, tmp_path)).to_pandas()
    assert not df.duplicated(subset=keys).any(), (
        f"{entry['id']}: duplicate keys on {keys}"
    )


@pytest.mark.parametrize(
    "entry",
    [e for e in CATALOG["fixtures"] if _invariant(e, "null_rate_ceilings")],
    ids=lambda e: e["id"],
)
def test_null_rate_ceilings(entry, tmp_path):
    ceilings = _invariant(entry, "null_rate_ceilings")
    df = pq.read_table(_run(entry, tmp_path)).to_pandas()
    if len(df) == 0:
        return
    for col, max_rate in ceilings.items():
        rate = df[col].isna().mean()
        assert rate <= max_rate, (
            f"{entry['id']}: null-rate on `{col}` = {rate:.3f} > {max_rate}"
        )


# ---- Great Expectations: a documented subset of the mod-104 suite --------

@pytest.mark.parametrize("entry", CATALOG["fixtures"], ids=lambda e: e["id"])
def test_great_expectations_suite(entry, tmp_path):
    df = pq.read_table(_run(entry, tmp_path)).to_pandas()
    if len(df) == 0:
        return
    gdf = gx.from_pandas(df)
    gdf.expect_table_columns_to_match_ordered_list(list(EXPECTED_SCHEMA))
    gdf.expect_column_values_to_not_be_null("doc_id")
    gdf.expect_column_values_to_not_be_null("source")
    gdf.expect_column_values_to_not_be_null("text")
    gdf.expect_column_values_to_match_regex("row_hash", r"^[0-9a-f]{64}$")
    result = gdf.validate()
    assert result["success"], f"{entry['id']}: GE suite failed: {result}"


# ---- Hypothesis: generative idempotence property -------------------------

@st.composite
def _synthetic_doc(draw):
    return {
        "doc_id":      f"d-{draw(st.integers(0, 10_000)):05d}",
        "source":      draw(st.sampled_from(["wiki", "news", "forum"])),
        "language":    "en",
        "text":        draw(st.text(min_size=1, max_size=256)),
        "ingested_at": "2025-01-02T10:00:00Z",
    }


@given(docs=st.lists(_synthetic_doc(), min_size=1, max_size=32))
@settings(max_examples=50, deadline=None)
def test_idempotent_on_canonical_input(docs, tmp_path):
    """Second pipeline run on its own output must equal the first."""
    import json
    (tmp_path / "first.jsonl").write_text(
        "\n".join(json.dumps(d) for d in docs) + "\n"
    )
    run_pipeline(input_path=tmp_path / "first.jsonl",
                 output_path=tmp_path / "first.parquet")
    df = pq.read_table(tmp_path / "first.parquet").to_pandas()
    with (tmp_path / "second.jsonl").open("w") as f:
        for _, row in df.iterrows():
            f.write(json.dumps({
                "doc_id": row["doc_id"], "source": row["source"],
                "language": row["language"], "text": row["text"],
                "ingested_at": row["ingested_at"].isoformat()
                                .replace("+00:00", "Z"),
            }) + "\n")
    run_pipeline(input_path=tmp_path / "second.jsonl",
                 output_path=tmp_path / "second.parquet")
    assert pq.read_table(tmp_path / "first.parquet").equals(
        pq.read_table(tmp_path / "second.parquet")
    )
```

Design notes defensible at review:

- **Two layers.** Golden equality pins exact behaviour; invariants pin
  the shape. A golden regen is an exercise of judgment; an invariant
  failure is always a bug.
- **Invariants declared per-fixture in the catalog, not hard-coded.**
  Adding a new invariant is a catalog edit plus (if new) a test fn.
- **Great Expectations overlap with mod-104 on purpose** — the fixture
  pack asserts a documented subset of the production suite.
- **Hypothesis scoped to idempotence.** A narrow property ("second run
  equals first") is cheap to defend and catches the entire class of
  non-deterministic cleaning regressions.

### 2.8 `scripts/regen_goldens.py`

```python
# scripts/regen_goldens.py
# Regenerate fixtures/expected/ when pipeline behaviour INTENTIONALLY
# changes. Refuses dirty tree. Requires --justification and --reviewer.
# Writes a CSV-level unified diff of old vs. new into the changelog.
from __future__ import annotations

import argparse, difflib, hashlib, subprocess, sys
from datetime import datetime, timezone
from pathlib import Path

import pyarrow.parquet as pq
import yaml

REPO_ROOT = Path(__file__).resolve().parents[1]
CATALOG_PATH   = REPO_ROOT / "fixtures/catalog.yaml"
CHANGELOG_PATH = REPO_ROOT / "docs/CHANGELOG_GOLDENS.md"


def _sha256(p):   return hashlib.sha256(p.read_bytes()).hexdigest()
def _seal(p):     p.with_suffix(p.suffix + ".sha256").write_text(
                      f"{_sha256(p)}  {p.name}\n")
def _git_clean(): return subprocess.check_output(
    ["git", "status", "--porcelain"], cwd=REPO_ROOT, text=True).strip() == ""
def _git_head():  return subprocess.check_output(
    ["git", "rev-parse", "HEAD"], cwd=REPO_ROOT, text=True).strip()


def _render_diff(old: Path, new: Path) -> str:
    """CSV-level unified diff of two parquet files (reviewable in PR UI)."""
    def rows(p):
        return pq.read_table(p).to_pandas().to_csv(index=False).splitlines()
    return "\n".join(difflib.unified_diff(
        rows(old), rows(new),
        fromfile=f"{old.name} (old)", tofile=f"{new.name} (new)", n=3,
    ))


def main() -> None:
    ap = argparse.ArgumentParser(
        description="Regenerate golden fixtures. "
                    "NEVER run silently — see docs/FIXTURE_POLICY.md.")
    ap.add_argument("--justification", required=True,
        help="Non-empty; must reference a ticket id (e.g. 'DQ-812: ...').")
    ap.add_argument("--reviewer", required=True,
        help="Email of the second reviewer; copied into the changelog.")
    ap.add_argument("--only", action="append", default=None,
        help="Regenerate only this fixture id (repeatable).")
    args = ap.parse_args()

    if not args.justification.strip():
        sys.exit("--justification must be non-empty")
    if "DQ-" not in args.justification and "ticket:" not in args.justification:
        sys.exit("--justification must reference a ticket id")
    if not _git_clean():
        sys.exit("working tree is dirty; commit or stash before regen")

    # Import late so a broken pipeline does not mask the preflight.
    from corpus_clean.pipeline import run as run_pipeline

    catalog = yaml.safe_load(CATALOG_PATH.read_text())
    entries = catalog["fixtures"]
    if args.only:
        missing = set(args.only) - {e["id"] for e in entries}
        if missing:
            sys.exit(f"--only referenced unknown ids: {sorted(missing)}")
        entries = [e for e in entries if e["id"] in args.only]

    now, head = datetime.now(timezone.utc).isoformat(timespec="seconds"), \
                _git_head()
    diffs: list[str] = []
    for e in entries:
        expected = REPO_ROOT / e["expected"]
        new = expected.with_suffix(".parquet.new")
        run_pipeline(input_path=REPO_ROOT / e["input"], output_path=new)
        diff = _render_diff(expected, new) if expected.exists() else "(new)"
        diffs.append(f"### {e['id']}\n\n```diff\n{diff}\n```\n")
        new.replace(expected)
        _seal(expected)
        e["last_touched_commit"] = head

    CATALOG_PATH.write_text(yaml.safe_dump(catalog, sort_keys=False))

    header = (
        f"## {now}\n"
        f"- reviewer: {args.reviewer}\n"
        f"- justification: {args.justification}\n"
        f"- commit: {head}\n"
        f"- fixtures touched: {len(entries)}\n\n"
    )
    existing = CHANGELOG_PATH.read_text() if CHANGELOG_PATH.exists() else ""
    CHANGELOG_PATH.write_text(header + "\n".join(diffs) + "\n" + existing)


if __name__ == "__main__":
    main()
```

Design notes defensible at review:

- **`--justification` is required, non-empty, ticket-referencing.**
  `.strip()` closes `""`; the ticket check closes the "lgtm" shortcut.
- **Dirty tree aborts.** The regen commit must contain nothing but
  goldens + changelog.
- **Changelog prepended, not appended.** Newest first is what
  reviewers read.
- **`last_touched_commit` written back into the catalog** — this is
  what makes the dead-fixture CI check meaningful.
- **CSV-level unified diff.** Parquet is binary; CSV rendering is the
  lowest-friction way to make regeneration reviewable in a PR UI.

### 2.9 `ci/dead_fixture_check.py`

```python
# ci/dead_fixture_check.py
# Warn-only. Flags fixtures whose last_touched_commit is >= N releases
# behind main AND which have no entry in CHANGELOG_GOLDENS.md.
from __future__ import annotations
import subprocess
from pathlib import Path
import yaml

REPO = Path(__file__).resolve().parents[1]
CATALOG = yaml.safe_load((REPO / "fixtures/catalog.yaml").read_text())
CHANGELOG = (REPO / "docs/CHANGELOG_GOLDENS.md")
STALE_RELEASES = 3


def _releases_since(commit):
    try:
        out = subprocess.check_output(
            ["git", "tag", "--contains", commit, "--sort=-creatordate"],
            cwd=REPO, text=True)
    except subprocess.CalledProcessError:
        return 0
    return sum(1 for t in out.splitlines() if t.startswith("v"))


def main():
    changelog = CHANGELOG.read_text() if CHANGELOG.exists() else ""
    stale = [e["id"] for e in CATALOG["fixtures"]
             if e.get("last_touched_commit")
             and _releases_since(e["last_touched_commit"]) >= STALE_RELEASES
             and e["id"] not in changelog]
    if stale:
        print("DEAD FIXTURE WARNING — review or regen:")
        for s in stale:
            print(f"  - {s}")
    return 0  # warn only


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.10 `docs/FIXTURE_POLICY.md`

A one-page policy. Required sections:

1. **When to add.** Every bug fix lands with one new row under
   `fixtures/inputs/historical/` named `<nnn>_ticket_<id>_<slug>.jsonl`.
   The bugfix PR and the fixture PR are the same PR; no exceptions.
2. **When to regenerate.** Never silently. A separate PR generated by
   `scripts/regen_goldens.py`, with non-empty `--justification` citing
   a ticket and a named `--reviewer`. Reviewers verify the diff in
   `CHANGELOG_GOLDENS.md` matches the intended behavioural change.
3. **Size budget.** 16 KiB per input, 64 KiB per expected, 2 MiB
   pack-aggregate. Enforced by `test_catalog_integrity.py`. Fixtures
   travel in every clone; keep the repo honest.
4. **Quarantine.** Flaky fixtures go into `fixtures/quarantine.yaml`
   with an `expires_on` ≤ 14 days ahead and a named owner. Expiry
   fails the integrity test, which forces action.
5. **Rot.** `ci/dead_fixture_check.py` is warn-only; a fixture
   `last_touched_commit` ≥ 3 releases behind `main` and absent from
   `CHANGELOG_GOLDENS.md` is listed for owner review.
6. **Ownership.** Every fixture has an `owner` (per-fixture or
   defaulted). CODEOWNERS routes review of the subdirectory to that
   team.
7. **Prohibited content.** No real user data, credentials, or
   secrets. Incident-derived fixtures are hand-synthesised to
   preserve the shape of the bug, not the data.
8. **Sibling contract.** Invariants in `catalog.yaml` are a
   documented subset of the mod-104 Great Expectations suite;
   mod-104 is authoritative for production.

## 3. Validation steps

1. **Catalog integrity passes.** `pytest tests/test_catalog_integrity.py`
   is green: every filesystem fixture is declared, every catalog
   entry resolves to disk, every pair has sidecars, the pack is
   within budget, no quarantine entry is expired.
2. **Sidecar hashes match disk.** `for f in $(git ls-files 'fixtures/**'
   | grep -v '\.sha256$'); do sha256sum -c "$f.sha256"; done` emits
   `OK` for every fixture. A single `FAILED` is review-blocking; re-seal
   only when the change to the fixture body was intended.
3. **Golden suite passes.** `pytest tests/test_golden_outputs.py`
   succeeds on every non-quarantined catalog entry. Failures are
   either a real regression (fix in this PR) or an intentional
   behaviour change (accept via `regen_goldens.py` in a separate PR).
4. **Invariant suite passes.** `pytest tests/test_invariants.py`
   succeeds on every catalog entry. Invariant failures are always
   bugs — schema drift, null-rate blow-up, or duplicate join keys
   cannot be "accepted" via regeneration.
5. **Hypothesis idempotence holds.** `pytest -k idempotent` runs the
   generative test; 50 synthetic shards all satisfy "second run equals
   first".
6. **Regeneration gate works.** `scripts/regen_goldens.py` with no args,
   or with an empty `--justification`, or with a `--justification` that
   does not reference a ticket, exits non-zero. A well-formed invocation
   (`--justification 'DQ-900: pin new dedupe threshold' --reviewer
   r@team.example`) on a clean tree updates `fixtures/expected/`,
   re-seals sidecars, prepends a `CHANGELOG_GOLDENS.md` entry with the
   per-fixture CSV diff, and bumps `last_touched_commit`.
7. **Dirty tree aborts regen.** `touch dirty.txt &&
   scripts/regen_goldens.py --justification 'DQ-900: x' --reviewer
   r@t.example` exits non-zero before touching any golden.
8. **Quarantine semantics.** A fixture in `quarantine.yaml` is skipped
   by `test_golden_outputs.py`; `test_invariants.py` still runs against
   it (invariants are not quarantinable). An expired `expires_on` fails
   `test_catalog_integrity.py`.
9. **Dead-fixture CI check is warn-only.** `ci/dead_fixture_check.py`
   exits 0 and prints stale ids; owners are paged via CODEOWNERS.
10. **Governance anchors cited** with exact URLs in §6; unverifiable
    claims are called out inline as placeholders with a named owner
    for the follow-up.

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros
on critical rows.

| #  | Dimension                                                                                                           | Critical? |
|----|---------------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `fixtures/catalog.yaml` lists every pair with purpose, invariants, owner, `last_touched_commit`, `added_in_ticket`  | Yes       |
| 2  | Every input `.jsonl` has a sidecar `.sha256` and every expected `.parquet` has a sidecar `.sha256`                  | Yes       |
| 3  | Fixture categories cover happy_path, boundary, unicode, historical, adversarial (each a subdir of `inputs/`)        | Yes       |
| 4  | At least one `historical/` fixture per past bug, named with the ticket id                                           | Yes       |
| 5  | `tests/test_golden_outputs.py` uses canonical sort (not raw byte compare) and pins Parquet writer options           | Yes       |
| 6  | `tests/test_invariants.py` asserts schema conformance, row-count bounds, monotonic timestamps, unique join keys, null-rate ceilings | Yes       |
| 7  | Great Expectations expectations overlap deliberately with the mod-104 suite                                         | No        |
| 8  | A `hypothesis` property test asserts pipeline idempotence on its own output                                         | No        |
| 9  | `scripts/regen_goldens.py` requires `--justification`, enforces ticket id, requires `--reviewer`, refuses dirty tree | Yes       |
| 10 | `regen_goldens.py` writes a human-readable per-fixture diff to `CHANGELOG_GOLDENS.md` and bumps `last_touched_commit` | Yes       |
| 11 | `fixtures/quarantine.yaml` entries require `expires_on`; expiry is enforced by `test_catalog_integrity.py`          | Yes       |
| 12 | `docs/FIXTURE_POLICY.md` documents when to add, when to regen, size budget, quarantine, rot, ownership              | Yes       |
| 13 | Fixture-size budget (per-fixture and pack-aggregate) is enforced by a test, not just documented                     | Yes       |
| 14 | No raw user data, credentials, or secrets in any fixture                                                            | Yes       |
| 15 | Governance anchors (OWASP ML02:2023, MITRE ATLAS Impact pillar, NIST AI RMF MEASURE/MANAGE pillars) cited with URLs | No        |

## 5. Common mistakes

- **Comparing raw Parquet bytes.** Row ordering, dictionary encoding,
  and row-group sizing flip on tiny data or library changes. Byte diff
  is unreviewable noise. Canonical-sort and compare at the Arrow
  logical level with writer options pinned.
- **Using `pytest-regressions` auto-approve.**
  `data_regression.check(force_regen=True)` writes a new golden the
  first time it fails — a silent-accept pathway in CI. Route all
  regeneration through `scripts/regen_goldens.py` with its
  justification gate.
- **Globbing the filesystem for fixtures.** A stray `.jsonl` under
  `fixtures/inputs/` runs as a test under a glob harness and usually
  passes vacuously. The catalog-driven harness makes orphans a test
  failure.
- **Fixture filename without a ticket id.** Six months later nobody
  remembers what `030_special_case.jsonl` protects. Put the ticket id
  in the filename; put the "why" in the catalog entry.
- **Sidecar hashes generated by the test harness.** Self-fulfilling
  hashes catch nothing. Sidecars are committed, hand-sealed by
  `scripts/seal_fixture.py`, checked at the top of every test.
- **Empty `--justification` sneaks past the gate.** `""` or `" "`
  must fail the preflight; `.strip()` and the ticket-id check are
  doing real work.
- **Regenerating goldens in the same PR as a behavioural code
  change.** Reviewers cannot tell the behavioural diff from the
  golden movement. Regeneration is always its own PR from a clean
  tree.
- **Over-sized fixtures.** A 2 MB "representative shard" costs every
  clone forever and is almost never minimal. If 16 KiB does not cover
  the assertion, split it into two smaller fixtures.
- **Invariants drift from the production Great Expectations suite.**
  A fixture pack tighter than production goes red on acceptable
  shards; looser than production misses real regressions. Keep the
  pack a documented subset of mod-104.
- **No historical row per bug.** The `historical/` subdirectory goes
  empty six months later; the pack loses its memory. Enforce
  "bugfix adds one row" in code review.
- **Quarantining with no expiry.** No `expires_on` = permanent
  disable. The integrity test's refusal is what forces the fixer's
  hand.
- **Treating `last_touched_commit` as cosmetic.** A fixture that has
  not moved in a year probably protects behaviour the pipeline no
  longer has. The dead-fixture CI job surfaces it; owner regenerates
  or deletes.

## 6. References

- **OWASP Machine Learning Security Top 10 — ML02:2023 Data Poisoning
  Attack:** <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Silent upstream pipeline changes are a self-poisoning vector; a
  regression-fixture suite is the earliest deterministic tripwire.
- **MITRE ATLAS (pillar reference):** <https://atlas.mitre.org/>.
  The Impact tactic is the pillar-level anchor for silent
  training-data corruption. The specific AML.T* identifier that most
  cleanly covers "silent upstream pipeline change corrupts training
  data" is a reviewer-owned follow-up and should be pinned once
  ATLAS adds the matching technique.
- **NIST AI Risk Management Framework (AI RMF 1.0):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. **MEASURE**
  frames validity and reliability evaluation (what the invariant suite
  operationalises); **MANAGE** frames the deactivation posture when
  the regression suite goes red.
- **Great Expectations:** <https://greatexpectations.io/>. Invariants
  in `test_invariants.py` are authored as GE expectations where the
  equivalent mod-104 expectation exists in the production suite.
- **hypothesis:** <https://hypothesis.readthedocs.io/>. The generative
  companion to the hand-curated pack; the idempotence property catches
  regressions the authors did not imagine.
- **pytest-regressions:** <https://pytest-regressions.readthedocs.io/>.
  The reference library for golden-file testing in pytest. This
  exercise deliberately does not use `data_regression.check()`'s
  auto-approve path; the regen gate here is stricter than the
  library's default.
- **Local anchors (siblings in this module):**
  - `exercise-01` (DQ monitoring contract) — fixtures under
    `boundary/` and `historical/` catch silent SLI-schema changes to
    the contract itself.
  - `exercise-02` (drift detection) — `adversarial/` rows stress the
    drift tools' thresholding; drift unit tests parametrise off them.
  - `exercise-03` (schema-drift detector) — `boundary/` and
    `historical/` include the exact schema-drift scenarios the
    detector in ex03 must flag; the detector's unit tests are seeded
    from these fixtures.
  - `exercise-04` (sub-population drift) — fixtures include known
    subpop-drift distributions as deterministic inputs to the
    detector tests.
  - `mod-104` (Great Expectations suite) — catalog invariants are a
    documented subset of the mod-104 expectations; mod-104 is
    authoritative for production, this fixture pack is authoritative
    for CI tripwires.
  - `mod-109` (dataset versioning) — sidecar `.sha256` files are
    mod-109-pinned identities, interoperable with the dataset-manifest
    hashes used for lineage.
