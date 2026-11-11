# Exercise 05 — Croissant and ML-BOM Metadata Emission (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner emits two machine-readable
artifacts alongside the dataset build:

- A **Croissant** manifest — an MLCommons-stewarded JSON-LD vocabulary built on
  top of `schema.org/Dataset`, designed so dataset consumers can discover,
  download, and load an ML-ready dataset uniformly across catalogues.
- A **CycloneDX ML-BOM** — a CycloneDX bill-of-materials describing the
  "model" / "dataset" components and their licence, hashes, supplier, and
  dependencies. CycloneDX is an OWASP-stewarded specification; the ML-BOM
  profile adds `components` of type `machine-learning-model` and `data`.

The two artifacts are complementary. Croissant is dataset-centric and
loader-friendly; the ML-BOM is supply-chain-centric and governance-friendly.
Both are emitted at corpus freeze, both are checked into the dataset artifact,
and both must validate against the current published schema before the build
is considered done.

## 2. Implementation — worked answer

### 2.1 Pre-flight checks

Before emitting metadata, the pipeline needs the following facts recorded from
the earlier exercises:

| Fact                          | Source                                |
|-------------------------------|---------------------------------------|
| Dataset version               | Lineage / Exercise 04 datasheet       |
| Per-row SPDX licence          | Ingest manifest (Exercise 02)         |
| Opt-out decision log          | Exercise 03                           |
| Content-hash per file         | Lineage (mod-109)                     |
| Top-level distribution URL    | Internal artifact store               |
| Dataset owner (role)          | Exercise 04 datasheet                 |
| Model(s) trained from dataset | Model-card manifest (for the ML-BOM)  |

If any of these are missing, the metadata emitter refuses to run.

### 2.2 Worked Croissant manifest

File: `croissant.json`. Keys below reflect the Croissant vocabulary. Confirm
each key against the current spec before committing.

```json
{
  "@context": {
    "@language": "en",
    "@vocab": "https://schema.org/",
    "cr": "http://mlcommons.org/croissant/",
    "sc": "https://schema.org/"
  },
  "@type": "sc:Dataset",
  "name": "internal-python-packaging-sft",
  "description": "Supervised fine-tuning corpus for an internal Python-packaging assistant. See DATASHEET.md and DATA_STATEMENT.md for the full context.",
  "version": "1.0.0",
  "datePublished": "YYYY-MM-DD",
  "license": "LicenseRef-internal-use-only",
  "citeAs": "Internal, no external citation.",
  "url": "https://artifact-store.internal/datasets/internal-python-packaging-sft/1.0.0",
  "creator": {
    "@type": "sc:Organization",
    "name": "Data Platform — AI Data Team"
  },
  "publisher": {
    "@type": "sc:Organization",
    "name": "Internal"
  },
  "sameAs": [
    "https://artifact-store.internal/datasets/internal-python-packaging-sft"
  ],
  "cr:conformsTo": "http://mlcommons.org/croissant/1.0",

  "distribution": [
    {
      "@type": "cr:FileObject",
      "@id": "train-jsonl",
      "name": "train.jsonl",
      "contentUrl": "train/train.jsonl",
      "encodingFormat": "application/x-ndjson",
      "sha256": "<sha256-of-train-jsonl>",
      "contentSize": "<bytes>"
    },
    {
      "@type": "cr:FileObject",
      "@id": "dev-jsonl",
      "name": "dev.jsonl",
      "contentUrl": "dev/dev.jsonl",
      "encodingFormat": "application/x-ndjson",
      "sha256": "<sha256-of-dev-jsonl>",
      "contentSize": "<bytes>"
    },
    {
      "@type": "cr:FileObject",
      "@id": "held-jsonl",
      "name": "held.jsonl",
      "contentUrl": "held/held.jsonl",
      "encodingFormat": "application/x-ndjson",
      "sha256": "<sha256-of-held-jsonl>",
      "contentSize": "<bytes>"
    },
    {
      "@type": "cr:FileObject",
      "@id": "attribution-jsonl",
      "name": "attribution.jsonl",
      "contentUrl": "attribution.jsonl",
      "encodingFormat": "application/x-ndjson",
      "sha256": "<sha256-of-attribution-jsonl>",
      "description": "Per-row attribution and SPDX licence. Required by Exercise 02 policy."
    },
    {
      "@type": "cr:FileObject",
      "@id": "optout-log-jsonl",
      "name": "opt-out-log.jsonl",
      "contentUrl": "opt-out-log.jsonl",
      "encodingFormat": "application/x-ndjson",
      "sha256": "<sha256-of-optout-log>",
      "description": "Per-URL opt-out verdicts from Exercise 03."
    }
  ],

  "recordSet": [
    {
      "@type": "cr:RecordSet",
      "@id": "sft-records",
      "name": "sft-records",
      "description": "Prompt/answer pairs with per-row provenance.",
      "field": [
        {
          "@type": "cr:Field",
          "@id": "sft-records/prompt",
          "name": "prompt",
          "description": "Free-text user prompt in English.",
          "dataType": "sc:Text",
          "source": {
            "fileObject": {"@id": "train-jsonl"},
            "extract": {"column": "prompt"}
          }
        },
        {
          "@type": "cr:Field",
          "@id": "sft-records/answer",
          "name": "answer",
          "description": "Target answer string.",
          "dataType": "sc:Text",
          "source": {
            "fileObject": {"@id": "train-jsonl"},
            "extract": {"column": "answer"}
          }
        },
        {
          "@type": "cr:Field",
          "@id": "sft-records/source_url",
          "name": "source_url",
          "description": "Upstream URL this row was derived from.",
          "dataType": "sc:URL",
          "source": {
            "fileObject": {"@id": "train-jsonl"},
            "extract": {"column": "source_url"}
          }
        },
        {
          "@type": "cr:Field",
          "@id": "sft-records/spdx",
          "name": "spdx_license",
          "description": "SPDX identifier of the upstream content. See Exercise 02 policy.",
          "dataType": "sc:Text",
          "source": {
            "fileObject": {"@id": "train-jsonl"},
            "extract": {"column": "spdx_license"}
          }
        },
        {
          "@type": "cr:Field",
          "@id": "sft-records/retrieved_at",
          "name": "retrieved_at",
          "description": "ISO-8601 retrieval timestamp.",
          "dataType": "sc:DateTime",
          "source": {
            "fileObject": {"@id": "train-jsonl"},
            "extract": {"column": "retrieved_at"}
          }
        }
      ]
    }
  ]
}
```

Notes on the choices above:

- **`license`** is `LicenseRef-internal-use-only` because the aggregate corpus
  is not redistributable even though individual rows carry permissive SPDX
  IDs. This is the correct framing — the dataset licence and the per-row
  licence are separate facts; the per-row licence lives in the `spdx_license`
  field.
- **`conformsTo`** names the Croissant version this manifest targets. Pin it
  explicitly; a manifest without a version pin will drift.
- **`sha256`** on every `FileObject` is required so the ML-BOM can cross-link
  to the same hash.
- **`attribution.jsonl`** and **`opt-out-log.jsonl`** are declared as first-class
  distribution files, not as "assets". A consumer that downloads the dataset
  cannot be left to re-derive the attribution; the Exercise 02 obligations
  travel with the files.

### 2.3 Worked CycloneDX ML-BOM

File: `bom.cdx.json`. CycloneDX is a JSON (and XML) format maintained by OWASP;
ML-BOM uses the standard CycloneDX envelope with `components[].type` values
for `data` and `machine-learning-model`.

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "serialNumber": "urn:uuid:00000000-0000-0000-0000-000000000000",
  "version": 1,
  "metadata": {
    "timestamp": "YYYY-MM-DDTHH:MM:SSZ",
    "tools": {
      "components": [
        {
          "type": "application",
          "name": "internal-mlbom-emitter",
          "version": "1.0.0"
        }
      ]
    },
    "authors": [
      {"name": "Data Platform — AI Data Team"}
    ],
    "component": {
      "bom-ref": "pkg:internal/dataset/internal-python-packaging-sft@1.0.0",
      "type": "data",
      "name": "internal-python-packaging-sft",
      "version": "1.0.0",
      "licenses": [
        {"license": {"id": "LicenseRef-internal-use-only"}}
      ],
      "externalReferences": [
        {"type": "documentation", "url": "https://artifact-store.internal/datasets/internal-python-packaging-sft/1.0.0/DATASHEET.md"},
        {"type": "documentation", "url": "https://artifact-store.internal/datasets/internal-python-packaging-sft/1.0.0/DATA_STATEMENT.md"}
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "<sha256-of-dataset-tarball>"}
      ]
    }
  },

  "components": [
    {
      "bom-ref": "pkg:internal/source/pypa-docs@<commit-sha>",
      "type": "data",
      "name": "pypa-docs",
      "version": "<commit-sha>",
      "supplier": {"name": "Python Packaging Authority"},
      "licenses": [
        {"license": {"id": "MIT"}}
      ],
      "externalReferences": [
        {"type": "vcs", "url": "https://github.com/pypa/packaging.python.org"}
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "<sha256-of-pypa-snapshot>"}
      ]
    },
    {
      "bom-ref": "pkg:internal/source/stackoverflow-export@2026-01-01",
      "type": "data",
      "name": "stackoverflow-export",
      "version": "2026-01-01",
      "supplier": {"name": "Stack Exchange"},
      "licenses": [
        {"expression": "CC-BY-SA-4.0"}
      ],
      "externalReferences": [
        {"type": "website", "url": "https://data.stackexchange.com/"}
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "<sha256-of-so-export>"}
      ]
    },
    {
      "bom-ref": "pkg:internal/source/support-tickets@2024-01-01_2025-12-31",
      "type": "data",
      "name": "internal-support-tickets",
      "version": "2024-01-01_2025-12-31",
      "supplier": {"name": "Internal (Support Platform)"},
      "licenses": [
        {"license": {"id": "LicenseRef-internal-use-only"}}
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "<sha256-of-ticket-export>"}
      ]
    }
  ],

  "dependencies": [
    {
      "ref": "pkg:internal/dataset/internal-python-packaging-sft@1.0.0",
      "dependsOn": [
        "pkg:internal/source/pypa-docs@<commit-sha>",
        "pkg:internal/source/stackoverflow-export@2026-01-01",
        "pkg:internal/source/support-tickets@2024-01-01_2025-12-31"
      ]
    }
  ],

  "compositions": [
    {
      "aggregate": "complete",
      "assemblies": [
        "pkg:internal/dataset/internal-python-packaging-sft@1.0.0"
      ]
    }
  ]
}
```

Notes on the choices above:

- **`components[].type: "data"`** is used for each upstream data source so the
  SBOM carries the supply-chain graph. The ML-BOM profile also permits
  `type: "machine-learning-model"` for the model that was trained; that
  component is added by the model-card pipeline, not this one.
- **`licenses[].expression`** on Stack Overflow is `CC-BY-SA-4.0` — that is the
  per-source licence string; the dataset-level licence remains
  `LicenseRef-internal-use-only`. Both facts are correct and both belong in
  the BOM. This is the piece that most often trips up a first-pass BOM
  emitter.
- **`hashes`** on every component is non-negotiable; a BOM without content
  hashes is not useful downstream.
- **`dependencies`** turns the component list into a graph; without it the
  BOM is a flat list and the "what-fed-what" question cannot be answered.
- **`serialNumber`** is a stable URN per BOM — it must change when the
  dataset is re-authored. Reusing the same `serialNumber` with different
  content silently breaks downstream BOM diffing tools.

### 2.4 Co-location and emission order

```
artifact-store/datasets/internal-python-packaging-sft/1.0.0/
├── train/train.jsonl
├── dev/dev.jsonl
├── held/held.jsonl
├── attribution.jsonl
├── opt-out-log.jsonl
├── DATASHEET.md
├── DATA_STATEMENT.md
├── ERRATA.md
├── croissant.json      # from §2.2
└── bom.cdx.json        # from §2.3
```

Emission order: data files → hashes → `croissant.json` → `bom.cdx.json`. The
BOM references the Croissant file only via the dataset-level docs URL; it
does **not** embed Croissant JSON. The BOM-as-graph and the Croissant-as-load
contract are independent artifacts and should stay that way.

## 3. Validation steps

1. **Croissant schema validation.** Validate `croissant.json` against the
   current Croissant JSON Schema. Validation must pass; any warning about
   unknown properties indicates the manifest was authored against the wrong
   `conformsTo` version.
2. **Croissant loader round-trip.** Point the official Croissant loader at the
   manifest and read the first 10 records of `sft-records`. If the records
   don't materialise, the `source` / `extract` wiring is wrong.
3. **CycloneDX schema validation.** Validate `bom.cdx.json` against the
   CycloneDX JSON Schema for the declared `specVersion`.
4. **BOM linter.** Run a CycloneDX linter / CLI tool over the BOM; require
   zero errors.
5. **Cross-reference check.** Every `FileObject.sha256` in `croissant.json`
   has a matching `components[].hashes[].content` entry in the BOM's
   dataset-level `metadata.component` or in the dataset-level composition.
   A mismatch means one of the two files was re-emitted without the other.
6. **Licence cross-reference.** Every `spdx_license` value that appears in
   `attribution.jsonl` appears as an SPDX ID (or a `LicenseRef-*` string) in
   at least one BOM component's `licenses`. If an SPDX ID appears in the data
   that does not appear in the BOM, the BOM understates the supply chain.
7. **Dataset version matches datasheet.** `version` in `croissant.json` and
   the dataset `component.version` in the BOM both match the version pinned
   in `DATASHEET.md`.
8. **No PII in metadata.** `grep` the two metadata files for email addresses
   and names; neither should appear unless it is the public-role contact
   point from the datasheet.
9. **Reproducibility smoke test.** Re-run the emitter against the same
   frozen inputs; both files should be byte-identical (ordering of
   dependencies and components fixed). A non-deterministic BOM is a BOM that
   will mis-diff.
10. **Attribution file is present and non-empty** whenever any BOM component
    has an attribution-requiring licence (`CC-BY-*`, `Apache-2.0`, etc.).

## 4. Rubric or review checklist

Score each row 0–2. Passing solution ≥ 22/28 and zero critical misses.

| # | Dimension                                                                                           | Critical? |
|---|-----------------------------------------------------------------------------------------------------|-----------|
| 1 | `croissant.json` exists, has a pinned `conformsTo` version, and validates against the schema       | Yes       |
| 2 | Croissant `distribution` lists every file consumers need, including attribution and opt-out logs    | Yes       |
| 3 | Croissant `recordSet` wires fields to file columns via `source` / `extract`                          | Yes       |
| 4 | Dataset-level `license` is distinguished from per-row `spdx_license`                                 | Yes       |
| 5 | `bom.cdx.json` exists, pins `specVersion`, and validates against the CycloneDX schema                | Yes       |
| 6 | BOM `metadata.component` describes the dataset itself; sources are listed under `components`        | Yes       |
| 7 | BOM `dependencies` turns the flat component list into a graph                                        | Yes       |
| 8 | Every component has a `hashes[]` entry (content-addressable)                                        | Yes       |
| 9 | BOM `serialNumber` is unique per dataset version                                                     | Yes       |
| 10 | Croissant and BOM share the same dataset version and matching SHA-256 at the dataset level          | Yes       |
| 11 | Per-source SPDX identifiers in the BOM match the SPDX IDs in `attribution.jsonl`                    | Yes       |
| 12 | Emitter refuses to run when prerequisites (per-row licence, hashes, owner) are missing              | No        |
| 13 | Emitter is deterministic (re-running yields byte-identical files)                                    | No        |
| 14 | Open follow-up notes exist wherever spec version / schema URL has not been freshly verified         | No        |

## 5. Common mistakes

- **Collapsing dataset-level and row-level licensing.** The dataset-level
  `license` on the Croissant / BOM is the aggregate licence; the per-row
  `spdx_license` is what the Exercise 02 policy acted on. Both facts must
  survive into the metadata.
- **Omitting attribution and opt-out logs from `distribution`.** A consumer
  that pulls the dataset needs the attribution file; if it is not in
  `distribution`, it is not part of the dataset in practice.
- **No `conformsTo` / no `specVersion`.** The metadata becomes un-validatable
  the moment the spec moves.
- **BOM without `dependencies`.** The BOM is supposed to be a *graph*; a flat
  component list does not answer the "what fed what" question.
- **Re-using `serialNumber` across versions.** Downstream BOM diffing silently
  sees no change; governance reviewers lose the only automatic signal that
  the supply chain moved.
- **Embedding Croissant inside the BOM or vice versa.** They are independent
  artifacts; cross-link via URL, do not nest.
- **Hand-authoring without a validator in CI.** Unvalidated metadata rots
  within one or two dataset versions. The validators are the control.
- **Treating the metadata as optional.** If the metadata emitter fails, the
  dataset build fails. Otherwise the metadata becomes a lagging indicator.

## 6. References

- **MLCommons Croissant.** Primary stewardship under MLCommons; JSON-LD
  vocabulary layered on `schema.org/Dataset`. (Verify the canonical URL and
  stable version before pinning `conformsTo`.)
- **Croissant GitHub.** (Verify the current reference implementation /
  validator repository URL under `mlcommons/croissant` or equivalent.)
- **CycloneDX specification.** OWASP-stewarded SBOM specification; the
  ML-BOM profile sits inside the main spec. (Verify the current spec version
  and the stable JSON Schema URL before pinning.)
- **CycloneDX tooling.** OWASP CycloneDX CLI and language-specific SDKs.
  (Verify the current recommended CLI invocation to use in CI, e.g.
  `cyclonedx-cli validate`.)
- **SPDX Licence List.** <https://spdx.org/licenses/>
  (Canonical vocabulary for the SPDX identifiers that appear in both files.)
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (GOVERN 1.4 / MEASURE 2.11 — documented, machine-readable data lineage is
  one of the controls this exercise operationalises.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (ML02 Data Poisoning, ML05 Model Theft — a validated supply-chain BOM is
  prerequisite evidence for several of the ranked risks in the project.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Browse the matrix for current data-supply-chain technique IDs before
  citing in a BOM-derived threat model.)
- Local anchors: Exercise 02 (SPDX identifiers that must appear in the BOM),
  Exercise 03 (opt-out log that must appear in Croissant `distribution`),
  Exercise 04 (datasheet / data statement linked from both files), mod-109
  (lineage pipeline that supplies the content hashes).
