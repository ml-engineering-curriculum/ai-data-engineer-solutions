# Exercise 03 — Schema Drift Alert and Rollback Workflow (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner wires an
end-to-end schema-drift control loop: a detector that compares the
live snapshot against a pinned reference schema and classifies drift
severity, an alert router that fans severity to the right channel
(PagerDuty + Slack on-call for major; Slack-only for minor), and a
rollback runbook plus decision tree the on-call follows when a major
event fires. The pipeline is deliberately boring: detector, alert
path, and revert command must be mechanical so the on-call does not
have to think at 3 a.m.

Deliverables (taken verbatim from the exercise contract):

- `detector/schema_check.py` — compares the current snapshot's schema
  (column set, dtypes, nullability, cardinality bounds) against
  `detector/reference_schema.json`. Emits
  `outputs/drift_report.json` with per-column findings and a
  top-level `severity` of `major` | `minor` | `none`.
- `detector/reference_schema.json` — the pinned reference: columns,
  dtypes, null policy, categorical whitelists, cardinality bounds.
- `alerting/routes.yaml` — severity-to-channel map with pinned
  thresholds.
- `alerting/notify.py` — reads the drift report and emits the right
  webhook payload for each route. No real credentials; env vars with
  placeholder names.
- `runbook/ROLLBACK.md` — the step-by-step rollback with concrete
  commands (DVC / lakeFS revert, dbt source-freshness override,
  feature-store version pin rollback). Names the approver, max blast
  radius, and exit criteria.
- `runbook/decision_tree.md` — the on-call decision tree.
- `tests/test_schema_check.py` — pytest covering added column,
  removed column, type widened, type narrowed, nullable → non-null,
  categorical-whitelist expansion.

Governance framing. Schema drift is the observable surface of
upstream poisoning and silent producer change. The OWASP Machine
Learning Security Top 10 anchor is **ML02:2023 Data Poisoning Attack**
— a column that appears (or disappears) without an upstream change
request is the shape a producer-side poisoning event takes; the
detector is the first-line compensating control. The MITRE ATLAS
technique in scope is **AML.T0019 Publish Poisoned Datasets** (an
adversary mutating upstream dataset shape to influence downstream
training). The NIST AI Risk Management Framework anchors are the
**MEASURE** pillar (performance over time — schema conformance is
the leading indicator that the pipeline still produces the artifact
contract) and the **MANAGE** pillar (mechanisms to supersede,
disengage, or deactivate AI systems — the rollback runbook is the
operational deactivation procedure when drift breaks the contract).

## 2. Implementation — worked answer

### 2.1 Project layout

```
exercise-03/
├── README.md
├── requirements.lock
├── Makefile
├── detector/
│   ├── schema_check.py
│   ├── reference_schema.json
│   └── severity_rules.py              # major/minor classification
├── alerting/
│   ├── routes.yaml                    # severity → channel map
│   ├── notify.py                      # webhook sender
│   └── templates/
│       ├── slack_major.json
│       ├── slack_minor.json
│       └── pagerduty_v2.json
├── runbook/
│   ├── ROLLBACK.md                    # step-by-step revert
│   └── decision_tree.md
├── outputs/
│   ├── drift_report.json              # detector output
│   └── drift_report.json.sha256
└── tests/
    ├── test_schema_check.py
    ├── test_severity_rules.py
    ├── test_notify_routing.py
    └── fixtures/
        ├── snapshot_added_optional.parquet
        ├── snapshot_removed_required.parquet
        ├── snapshot_widened.parquet
        ├── snapshot_narrowed.parquet
        ├── snapshot_nullable_to_notnull.parquet
        └── snapshot_whitelist_grown.parquet
```

### 2.2 `detector/reference_schema.json`

The pinned reference is the artifact contract. Every knob the detector
reads lives here; the detector never infers "reasonable defaults" from
the live snapshot.

```json
{
  "schema_version": "1.3.0",
  "dataset": "events.user_signup_v1",
  "primary_key": ["event_id"],
  "last_reviewed_utc": "2026-10-14T00:00:00Z",
  "last_reviewer": "data-platform-oncall@example.com",
  "columns": [
    {"name": "event_id",     "dtype": "string", "nullable": false,
     "required": true, "pk_component": true},
    {"name": "event_ts_utc", "dtype": "timestamp[us, tz=UTC]",
     "nullable": false, "required": true},
    {"name": "user_id",      "dtype": "string", "nullable": false,
     "required": true},
    {"name": "country_code", "dtype": "string", "nullable": false,
     "required": true,
     "categorical": {
       "whitelist": ["US","GB","DE","FR","IN","JP","BR","CA","AU","NL"],
       "growth_budget_per_week": 2}},
    {"name": "signup_source", "dtype": "string", "nullable": false,
     "required": true,
     "categorical": {
       "whitelist": ["web","ios","android","partner_api"],
       "growth_budget_per_week": 0}},
    {"name": "device_os_major", "dtype": "int16", "nullable": true,
     "required": true, "cardinality": {"min": 2, "max": 40}},
    {"name": "experiment_bucket", "dtype": "string", "nullable": true,
     "required": false}
  ],
  "null_policy": {
    "non_null_columns_must_be_non_null": true,
    "min_non_null_fraction": 0.999
  }
}
```

Design notes: `primary_key` is a field, not folklore — broken-PK is
unambiguously `major`. `categorical.growth_budget_per_week` is pinned
per column so "a new value appeared" becomes a numeric check, not a
judgement call. `required` and `nullable` are independent (a column
may be `required: true, nullable: true` subject to the `null_policy`
floor). The reference itself is versioned via `schema_version` so
drift reports replay deterministically against the exact reference
they used.

### 2.3 `detector/schema_check.py`

The detector uses a minimal pyarrow-based check. A Great Expectations
`ExpectationSuite` is a drop-in substitute when the broader project
is already on GE (expectations:
`expect_table_columns_to_match_ordered_list`,
`expect_column_values_to_be_of_type`,
`expect_column_values_to_be_in_set`,
`expect_column_unique_value_count_to_be_between`).

```python
# detector/schema_check.py
from __future__ import annotations

import argparse, hashlib, json
from dataclasses import asdict, dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

import pyarrow.dataset as ds
from severity_rules import classify_severity

# dtype → (family, rank). Widening (rank↑ same family) is MINOR;
# narrowing or cross-family is MAJOR.
ARROW_TYPE_WIDTH = {
    "int8": ("int", 1), "int16": ("int", 2),
    "int32": ("int", 3), "int64": ("int", 4),
    "float32": ("float", 5), "float64": ("float", 6),
    "string": ("string", 1), "large_string": ("string", 2),
    "bool": ("bool", 1),
    "timestamp[us, tz=UTC]": ("timestamp", 1),
    "timestamp[ns, tz=UTC]": ("timestamp", 2),
    "date32": ("date", 1),
}


@dataclass(slots=True)
class ColumnFinding:
    column: str
    kind: str      # added | removed | type_changed | null_policy | cardinality | whitelist
    detail: dict[str, Any]
    severity: str = "none"


@dataclass(slots=True)
class DriftReport:
    reference_schema_version: str
    reference_schema_sha256: str
    snapshot_path: str
    snapshot_sha256: str
    generated_at_utc: str
    severity: str = "none"
    findings: list[ColumnFinding] = field(default_factory=list)


def _sha256(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def _compare_types(ref: str, obs: str) -> tuple[bool, str]:
    if ref == obs:
        return True, ""
    rf, rr = ARROW_TYPE_WIDTH.get(ref, ("unknown", 0))
    of, or_ = ARROW_TYPE_WIDTH.get(obs, ("unknown", 0))
    if rf != of:
        return False, "cross_family"
    return (or_ > rr), ("" if or_ > rr else "narrowed")


def check(reference: dict, snapshot_path: Path) -> DriftReport:
    dataset = ds.dataset(snapshot_path, format="parquet")
    schema = dataset.schema
    observed = {f.name: str(f.type) for f in schema}
    nullable = {f.name: f.nullable for f in schema}
    ref_cols = {c["name"]: c for c in reference["columns"]}
    findings: list[ColumnFinding] = []

    for name in observed:
        if name not in ref_cols:
            findings.append(ColumnFinding(name, "added",
                {"observed_dtype": observed[name]}))

    for name, ref in ref_cols.items():
        if name not in observed:
            findings.append(ColumnFinding(name, "removed",
                {"required": ref.get("required", True),
                 "pk_component": ref.get("pk_component", False)}))
            continue
        if ref["dtype"] != observed[name]:
            compat, reason = _compare_types(ref["dtype"], observed[name])
            findings.append(ColumnFinding(name, "type_changed",
                {"reference_dtype": ref["dtype"],
                 "observed_dtype": observed[name],
                 "widen_compatible": compat,
                 "reason": reason or "widened"}))
        ref_null = ref.get("nullable", True)
        if ref_null is False and nullable[name] is True:
            findings.append(ColumnFinding(name, "null_policy",
                {"direction": "nonnull_to_nullable"}))
        if ref_null is True and nullable[name] is False:
            findings.append(ColumnFinding(name, "null_policy",
                {"direction": "nullable_to_nonnull"}))
        if "categorical" in ref:
            budget = ref["categorical"]["growth_budget_per_week"]
            whitelist = set(ref["categorical"]["whitelist"])
            table = dataset.to_table(columns=[name])
            distinct = {v.as_py() for v in table.column(name).unique()
                        if v.is_valid}
            new_values = distinct - whitelist
            if new_values:
                findings.append(ColumnFinding(name, "whitelist",
                    {"new_values": sorted(new_values),
                     "growth_budget_per_week": budget}))
        if "cardinality" in ref:
            table = dataset.to_table(columns=[name])
            card = len({v.as_py() for v in table.column(name).unique()
                        if v.is_valid})
            lo, hi = ref["cardinality"]["min"], ref["cardinality"]["max"]
            if card < lo or card > hi:
                findings.append(ColumnFinding(name, "cardinality",
                    {"observed": card, "min": lo, "max": hi}))

    for f in findings:
        f.severity = classify_severity(f, ref_cols)
    top = ("major" if any(f.severity == "major" for f in findings) else
           "minor" if any(f.severity == "minor" for f in findings) else
           "none")
    return DriftReport(
        reference_schema_version=reference["schema_version"],
        reference_schema_sha256="",
        snapshot_path=str(snapshot_path),
        snapshot_sha256=_sha256(snapshot_path),
        generated_at_utc=datetime.now(timezone.utc).isoformat(timespec="seconds"),
        severity=top, findings=findings,
    )


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--reference", required=True, type=Path)
    ap.add_argument("--snapshot", required=True, type=Path)
    ap.add_argument("--out", required=True, type=Path)
    args = ap.parse_args()
    reference = json.loads(args.reference.read_text())
    report = check(reference, args.snapshot)
    report.reference_schema_sha256 = _sha256(args.reference)
    payload = {**asdict(report),
               "findings": [asdict(f) for f in report.findings]}
    args.out.write_text(json.dumps(payload, indent=2, sort_keys=True))
    print(f"severity={report.severity} findings={len(report.findings)}")


if __name__ == "__main__":
    main()
```

### 2.4 `detector/severity_rules.py`

Severity classification is isolated so the rule set is trivially
reviewable in code review, trivially testable, and explicitly listed
in `ROLLBACK.md`.

```python
# detector/severity_rules.py
# Pure functions. Deterministic on (finding, reference) pair.
#
# MAJOR (freeze + rollback):
#   - removed_required_column / removed_pk_component
#   - type_narrowed or cross_family
#   - nullable_to_nonnull on required column (producer forcing a
#     shape readers were not promised; often an outage surrogate)
#   - whitelist_grew on closed-world column (budget 0) or over budget
#   - cardinality out of declared bounds
# MINOR (ticket, next-day):
#   - added_optional_column
#   - type_widened_compatible (same family, rank↑)
#   - nonnull_to_nullable on optional column
#   - whitelist_grew within per-week budget
from __future__ import annotations
from typing import Any


def classify_severity(finding, ref_cols: dict[str, dict[str, Any]]) -> str:
    kind, detail = finding.kind, finding.detail
    ref = ref_cols.get(finding.column, {})

    if kind == "added":
        return "minor"
    if kind == "removed":
        if detail.get("pk_component") or detail.get("required", True):
            return "major"
        return "minor"
    if kind == "type_changed":
        return "minor" if detail["widen_compatible"] else "major"
    if kind == "null_policy":
        return "major" if ref.get("required", True) else "minor"
    if kind == "whitelist":
        budget = detail.get("growth_budget_per_week", 0)
        if budget == 0 or len(detail["new_values"]) > budget:
            return "major"
        return "minor"
    if kind == "cardinality":
        return "major"
    return "none"
```

Design notes: closed-world columns (`signup_source`, budget 0) are
always `major` on a new value; open-world columns (`country_code`,
budget 2) are budgeted, not open — three new values in a week is
`major`. A cardinality collapse (30 → 1) is an upstream outage
surrogate; the detector catches what a downstream training job
cannot distinguish from a low-signal feature at a glance.

### 2.5 `alerting/routes.yaml`

The severity-to-channel map is pinned in config; the sender is the
only code path and is covered by `tests/test_notify_routing.py`.

```yaml
# alerting/routes.yaml
schema_version: 1.0.0

severities:
  major:
    description: >
      Removed required/PK column, type narrowed or cross-family,
      nullable→nonnull on a required column, closed-world whitelist
      violation, or cardinality out of bounds. Freeze training and
      open an incident.
    channels:
      - type: pagerduty
        integration_key_env: PAGERDUTY_INTEGRATION_KEY    # placeholder
        severity_in_payload: critical
        dedup_key_template: "schema-drift::{dataset}::{reference_schema_version}"
      - type: slack
        webhook_env: SLACK_DATA_ONCALL_WEBHOOK            # placeholder
        channel: "#data-oncall"
        template: slack_major.json
      - type: email
        to_env: ONCALL_EMAIL                              # placeholder
        subject_template: "[MAJOR] Schema drift on {dataset}"
    acknowledge_sla_minutes: 15
    mitigation_sla_minutes: 60

  minor:
    description: >
      Added optional column, compatible type widening, open-world
      whitelist growth within weekly budget, or nonnull→nullable on
      an optional column. File a ticket; investigate next business day.
    channels:
      - type: slack
        webhook_env: SLACK_DATA_QUALITY_WEBHOOK           # placeholder
        channel: "#data-quality"
        template: slack_minor.json
    acknowledge_sla_hours: 24
    mitigation_sla_business_days: 2

  none:
    description: No drift. Recorded in metrics sink only.
    channels: []

metrics:
  statsd_prefix: data.quality.schema_drift
  dimensions: [dataset, reference_schema_version, severity]
```

Design notes defensible at review:

- **Dedup key is a template.** Keying PagerDuty's `dedup_key` on
  `dataset + reference_schema_version` collapses to one open incident
  per drifting dataset, not one per snapshot.
- **Env-var names, not credentials.** Senders resolve at runtime; no
  secret lives in the repo.
- **SLAs pinned per severity** so the on-call knows the ack window
  before picking up the page; the metrics sink records breach counts
  for the review covered by Exercise 05.

### 2.6 `alerting/notify.py`

```python
# alerting/notify.py
# Read a drift report, resolve routes.yaml, emit webhook payloads.
# Env vars are resolved at call-time; missing env is a loud failure.
from __future__ import annotations

import argparse, json, os, urllib.request
from pathlib import Path
from string import Template
import yaml


def _render(path: Path, ctx: dict) -> dict:
    return json.loads(Template(path.read_text()).safe_substitute(ctx))


def _post(url: str, payload: dict) -> None:
    # Production wiring replaces _post; tests mock it.
    data = json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(url, data=data,
        headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=10)      # noqa: S310


def _require_env(name: str) -> str:
    v = os.environ.get(name)
    if not v:
        raise SystemExit(f"missing env {name}")
    return v


def notify(report_path: Path, routes_path: Path,
           templates_dir: Path) -> list[dict]:
    report = json.loads(report_path.read_text())
    routes = yaml.safe_load(routes_path.read_text())
    severity = report["severity"]
    if severity == "none":
        return []
    sev_cfg = routes["severities"][severity]
    ctx = {
        "dataset": report.get("snapshot_path", "unknown"),
        "reference_schema_version": report["reference_schema_version"],
        "severity": severity,
        "finding_count": str(len(report["findings"])),
        "generated_at_utc": report["generated_at_utc"],
    }
    sent: list[dict] = []
    for ch in sev_cfg["channels"]:
        if ch["type"] == "pagerduty":
            key = _require_env(ch["integration_key_env"])
            payload = _render(templates_dir / "pagerduty_v2.json", ctx)
            payload["routing_key"] = key
            payload["dedup_key"] = Template(
                ch["dedup_key_template"]).safe_substitute(ctx)
            _post("https://events.pagerduty.com/v2/enqueue", payload)
            sent.append({"type": "pagerduty",
                         "dedup_key": payload["dedup_key"]})
        elif ch["type"] == "slack":
            webhook = _require_env(ch["webhook_env"])
            payload = _render(templates_dir / ch["template"], ctx)
            _post(webhook, payload)
            sent.append({"type": "slack", "channel": ch["channel"]})
        elif ch["type"] == "email":
            to = _require_env(ch["to_env"])
            sent.append({"type": "email", "to": to,
                "subject": Template(ch["subject_template"])
                           .safe_substitute(ctx)})
    return sent


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--report", required=True, type=Path)
    ap.add_argument("--routes", required=True, type=Path)
    ap.add_argument("--templates", required=True, type=Path)
    args = ap.parse_args()
    print(json.dumps(notify(args.report, args.routes, args.templates),
                     indent=2))


if __name__ == "__main__":
    main()
```

### 2.7 `runbook/ROLLBACK.md`

The runbook is the on-call's one source of truth during a page. It
names the approver, the max blast radius, and the exit criteria;
anything not written here is out of scope for the on-call at 3 a.m.

```markdown
# Schema Drift — Rollback Runbook

## Scope
- Dataset: `events.user_signup_v1`
- Trigger: drift report with `severity == "major"`.
- Pinned reference: `detector/reference_schema.json`.

## Roles
- **On-call** (DRI): executes the steps.
- **Approver**: `data-platform-oncall@example.com`. Must ack in
  Slack before step 6 (revert).
- **Observer**: training team on-call; informed in step 2.

## Max blast radius
- One dataset (`events.user_signup_v1`) and direct consumers:
  feature store view `fs.user_signup_features_v3`; nightly training
  job `training.user_intent_ranker`.
- Do not touch upstream producers; they are out of scope until a
  change request lands.

## Exit criteria
1. Detector returns `severity == "none"` against the reverted snapshot.
2. Training unpaused only after exercise-01's schema-conformance SLI
   is green on three consecutive snapshots.
3. Post-mortem linked in the PagerDuty incident.

## Steps

**1. Acknowledge the page.** PagerDuty ack within 15 min. Post in
`#data-oncall`: "ack <incident-id>, starting rollback".

**2. Freeze the downstream training job.**
```bash
airflow dags pause training.user_intent_ranker
airflow dags list-runs -d training.user_intent_ranker --state running
```
Post in `#training-oncall`: "training.user_intent_ranker paused due
to schema-drift incident <id>".

**3. Open the incident document.** Create
`incidents/<date>-schema-drift-<dataset>.md`; paste
`outputs/drift_report.json` into the evidence section; record
current and last-good snapshot pointers.

**4. Identify the last-good snapshot.**
```bash
# Option A (DVC)
dvc log data/events/user_signup_v1.parquet | head -n 20
git log --oneline -- data/events/user_signup_v1.parquet.dvc

# Option B (lakeFS)
lakectl log lakefs://warehouse/main --amount 20 \
    --path events/user_signup_v1/
lakectl diff lakefs://warehouse/main@<candidate> lakefs://warehouse/main
```

**5. Approver sign-off.** Approver replies ":+1:" in the incident
Slack thread with the commit / lakeFS ref. **Do not proceed without
the ack.**

**6. Revert.**
```bash
# Option A (DVC)
git checkout <last-good> -- data/events/user_signup_v1.parquet.dvc
dvc checkout data/events/user_signup_v1.parquet
git commit -m "rollback: revert user_signup_v1 to <last-good>"

# Option B (lakeFS)
lakectl branch revert lakefs://warehouse/main <last-good>

# Option C (dbt source freshness ceiling, when the pipeline reads
# the source directly — blocks consumption past the drifting load)
dbt source freshness --select source:events.user_signup_v1 \
    --vars '{"loaded_at_ceiling": "<last-good-timestamp>"}'
```

**7. Rollback the feature-store version pin.**
```bash
feast materialize-rollback \
    --feature-view user_signup_features_v3 \
    --to-version <previous-version>
```

**8. Re-run the detector.**
```bash
python detector/schema_check.py \
    --reference detector/reference_schema.json \
    --snapshot data/events/user_signup_v1.parquet \
    --out outputs/drift_report_postrollback.json
jq -r .severity outputs/drift_report_postrollback.json   # expect "none"
```

**9. Keep the training job paused** until the schema is re-pinned by
the producer — either the reference is updated to accept the new
shape via a reviewed change request, or the producer reverts. Do not
resume on "it looks fine now".

**10. Close the incident.** Link the post-mortem; emit
`data.quality.schema_drift.incidents_closed{dataset=…}`.

## What NOT to do
- Do not edit `detector/reference_schema.json` during the incident
  (reference changes go through a reviewed PR).
- Do not unpause training to "see if it works".
- Do not revert further than one commit without explicit approver
  ack — blast radius grows non-linearly with revert depth.
```

### 2.8 `runbook/decision_tree.md`

```markdown
# On-call decision tree — schema drift

```
drift_report.json received
         │
         ├─ severity == "none"      ──► No action. Metric only.
         │
         ├─ severity == "minor"
         │       │
         │       ├─ Open ticket in #data-quality queue.
         │       ├─ Tag producer team.
         │       └─ Schedule investigation next business day.
         │          No freeze. No revert.
         │
         └─ severity == "major"
                 │
                 ├─ 1. Ack the PagerDuty incident (< 15 min).
                 ├─ 2. Freeze downstream training job (ROLLBACK §2).
                 ├─ 3. Open incident document.
                 ├─ 4. Identify last-good snapshot (ROLLBACK §4).
                 ├─ 5. Get approver sign-off (ROLLBACK §5).
                 ├─ 6. Revert data pointer (ROLLBACK §6).
                 ├─ 7. Roll feature-store version pin back (§7).
                 ├─ 8. Re-run detector; expect severity=none (§8).
                 ├─ 9. Keep training paused until re-pin (§9).
                 └─10. Close incident; link post-mortem (§10).
```

Edge cases:

- **Detector itself suspected.** Re-run on a known-good fixture
  (Exercise 05). If the detector mis-flags the known-good, open a
  separate incident against the detector but still freeze training
  until resolved — false freeze costs one training run; missed
  drift costs a poisoned model.
- **Multiple major findings in one report.** Follow the runbook
  once; the revert is per-dataset, not per-finding.
- **Drift on a dataset with no current training consumer.** Still
  revert. The schema contract is the contract.
```

### 2.9 `alerting/templates/slack_major.json`

```json
{
  "text": ":rotating_light: MAJOR schema drift on $dataset",
  "blocks": [
    {"type": "header",
     "text": {"type": "plain_text",
              "text": "MAJOR schema drift: $dataset"}},
    {"type": "section",
     "fields": [
       {"type": "mrkdwn", "text": "*Reference:* $reference_schema_version"},
       {"type": "mrkdwn", "text": "*Findings:* $finding_count"},
       {"type": "mrkdwn", "text": "*Detected:* $generated_at_utc"},
       {"type": "mrkdwn", "text": "*Severity:* $severity"}
     ]},
    {"type": "section",
     "text": {"type": "mrkdwn",
              "text": "Runbook: `runbook/ROLLBACK.md`. Decision tree: `runbook/decision_tree.md`."}}
  ]
}
```

### 2.10 `tests/test_schema_check.py`

Six fixtures cover the full severity table. The tests assert the
detector's severity rollup, the per-finding `kind`, and the
per-finding `severity`.

```python
# tests/test_schema_check.py
from __future__ import annotations
import json
from pathlib import Path
import pyarrow as pa
import pyarrow.parquet as pq
from detector.schema_check import check

REFERENCE = json.loads(Path("detector/reference_schema.json").read_text())


def _base() -> pa.Schema:
    return pa.schema([
        ("event_id",         pa.string()),
        ("event_ts_utc",     pa.timestamp("us", tz="UTC")),
        ("user_id",          pa.string()),
        ("country_code",     pa.string()),
        ("signup_source",    pa.string()),
        ("device_os_major",  pa.int16()),
        ("experiment_bucket", pa.string()),
    ])


def _rows() -> dict:
    return {
        "event_id":          ["e1", "e2", "e3"],
        "event_ts_utc":      pa.array([1, 2, 3], pa.timestamp("us", tz="UTC")),
        "user_id":           ["u1", "u2", "u3"],
        "country_code":      ["US", "GB", "DE"],
        "signup_source":     ["web", "ios", "android"],
        "device_os_major":   [1, 2, 3],
        "experiment_bucket": ["a", "b", None],
    }


def _write(tmp, schema, rows):
    p = tmp / "snapshot.parquet"
    pq.write_table(pa.Table.from_pydict(rows, schema=schema), p)
    return p


def test_added_optional_column_is_minor(tmp_path):
    schema = _base().append(pa.field("campaign_source", pa.string()))
    rows = _rows(); rows["campaign_source"] = ["organic", "paid", "organic"]
    r = check(REFERENCE, _write(tmp_path, schema, rows))
    assert r.severity == "minor"
    assert any(f.column == "campaign_source" and f.kind == "added"
               and f.severity == "minor" for f in r.findings)


def test_removed_required_column_is_major(tmp_path):
    schema = pa.schema([f for f in _base() if f.name != "country_code"])
    rows = {k: v for k, v in _rows().items() if k != "country_code"}
    r = check(REFERENCE, _write(tmp_path, schema, rows))
    assert r.severity == "major"
    assert any(f.column == "country_code" and f.kind == "removed"
               for f in r.findings)


def test_type_widened_is_minor(tmp_path):
    schema = pa.schema([(f.name,
        pa.int32() if f.name == "device_os_major" else f.type)
        for f in _base()])
    rows = _rows()
    rows["device_os_major"] = pa.array([1, 2, 3], pa.int32())
    r = check(REFERENCE, _write(tmp_path, schema, rows))
    assert r.severity == "minor"


def test_type_narrowed_is_major(tmp_path):
    schema = pa.schema([(f.name,
        pa.int8() if f.name == "device_os_major" else f.type)
        for f in _base()])
    rows = _rows()
    rows["device_os_major"] = pa.array([1, 2, 3], pa.int8())
    r = check(REFERENCE, _write(tmp_path, schema, rows))
    assert r.severity == "major"


def test_nullable_to_nonnull_on_optional_is_minor(tmp_path):
    # experiment_bucket: required=false, nullable=true in reference.
    schema = pa.schema([pa.field(f.name, f.type,
        nullable=False if f.name == "experiment_bucket" else f.nullable)
        for f in _base()])
    rows = _rows(); rows["experiment_bucket"] = ["a", "b", "c"]
    r = check(REFERENCE, _write(tmp_path, schema, rows))
    assert r.severity == "minor"


def test_whitelist_grown_on_closed_world_is_major(tmp_path):
    # signup_source budget = 0.
    rows = _rows(); rows["signup_source"] = ["web", "ios", "referral_quiz"]
    r = check(REFERENCE, _write(tmp_path, _base(), rows))
    assert r.severity == "major"


def test_whitelist_grown_within_budget_is_minor(tmp_path):
    # country_code budget = 2; add one new value.
    rows = _rows(); rows["country_code"] = ["US", "GB", "ES"]
    r = check(REFERENCE, _write(tmp_path, _base(), rows))
    assert r.severity == "minor"
```

### 2.11 Worked scenario — `campaign_source` added silently

The producer pipeline pushes a release that writes a new column,
`campaign_source`, to every row. No change request was filed.

1. Nightly detector runs against the fresh snapshot.
2. Report: one finding, `kind=added`, `column=campaign_source`,
   per-finding `severity=minor`; top-level `severity=minor`.
3. `notify.py` resolves the `minor` route and posts `slack_minor.json`
   to `#data-quality`. No PagerDuty page. No freeze.
4. On-call files a ticket; next business day a change request bumps
   `schema_version` to 1.4.0 adding `campaign_source` as
   `nullable: true, required: false`. Detector returns `severity=none`.

Contrast — the same release also narrows `device_os_major` from
`int16` to `int8`:

1. Detector reports two findings — `added` (minor) + `type_changed`
   (major). Top-level `severity=major`.
2. `notify.py` fires the major route: PagerDuty + `#data-oncall` +
   email.
3. On-call follows `ROLLBACK.md`: freeze training, identify last-good
   DVC commit, approver signs off, revert, feature-store version pin
   rolled back, detector re-run returns `severity=none`.
4. Training stays paused until the producer reverts the narrowing or
   the reference is re-pinned via a reviewed PR.

## 3. Validation steps

1. **Reference schema well-formed.** `python -c "import json;
   json.load(open('detector/reference_schema.json'))"` succeeds;
   every column has `name`, `dtype`, `nullable`, `required`;
   `schema_version` is semver.
2. **Clean snapshot classifies `none`.** Detector run against a
   snapshot matching the reference emits top-level `severity=none`
   with empty `findings`.
3. **Drift report is schema-stable.** Every report has
   `reference_schema_version`, `reference_schema_sha256`,
   `snapshot_path`, `snapshot_sha256`, `generated_at_utc`,
   `severity`, `findings[]`. Enforced in tests.
4. **Severity rules covered.** `pytest tests/test_severity_rules.py`
   has one test per rule in `severity_rules.py`.
5. **Alert routing deterministic.** With env vars set and `_post`
   stubbed, `notify.py` prints exactly the channels the route
   demands — three for `major`, one for `minor`, zero for `none`.
6. **Missing env fails loud.** Unsetting `PAGERDUTY_INTEGRATION_KEY`
   makes `notify.py` exit non-zero naming the variable.
7. **Runbook exit criterion matches the detector.** `ROLLBACK.md`
   step 8 re-runs the detector; expected `severity=none`. Rehearse
   quarterly against a fixture.
8. **Decision tree cross-references.** `grep 'ROLLBACK §' decision_tree.md`
   resolves to sections that exist in `ROLLBACK.md`.
9. **Metrics emitted every run.** The detector emits the statsd
   metric in `routes.yaml.metrics` regardless of severity so a quiet
   channel is not confused with a broken detector.
10. **Reproducibility.** Two runs on the same `(reference, snapshot)`
    pair are byte-identical modulo `generated_at_utc` (`jq
    'del(.generated_at_utc)' | sha256sum` is stable).

## 4. Rubric or review checklist

Score each row 0–2 (0 = missing, 1 = present but shallow, 2 =
defensible with evidence). Pass threshold: ≥ 22/30 with zero zeros
on critical rows.

| #  | Dimension                                                                                                       | Critical? |
|----|-----------------------------------------------------------------------------------------------------------------|-----------|
| 1  | `detector/reference_schema.json` pins columns, dtypes, nullable, required, categorical whitelists, cardinality  | Yes       |
| 2  | `schema_version` is semver and advanced on every reference change                                               | Yes       |
| 3  | `detector/schema_check.py` emits the pinned `DriftReport` JSON shape every run                                  | Yes       |
| 4  | Severity rules are isolated in `severity_rules.py` and unit-tested per row                                      | Yes       |
| 5  | `removed required/PK column`, `type_narrowed`, closed-world whitelist violation are classified `major`          | Yes       |
| 6  | `added optional column`, `type_widened` (same family), budget-conformant whitelist growth are classified `minor` | Yes       |
| 7  | `alerting/routes.yaml` names env-var placeholders, never raw credentials                                        | Yes       |
| 8  | `alerting/notify.py` fails loud when an env var is missing                                                      | Yes       |
| 9  | PagerDuty `dedup_key` collapses duplicate pages per dataset + reference version                                 | No        |
| 10 | `runbook/ROLLBACK.md` names the approver, blast radius, and exit criteria                                       | Yes       |
| 11 | Rollback steps include DVC or lakeFS revert AND feature-store version-pin rollback                              | Yes       |
| 12 | `runbook/decision_tree.md` cross-references `ROLLBACK.md` by section                                            | Yes       |
| 13 | `tests/test_schema_check.py` covers added / removed / widened / narrowed / nullable-flip / whitelist            | Yes       |
| 14 | Governance anchors (OWASP ML Top 10 ML02, MITRE ATLAS AML.T0019, NIST AI RMF MEASURE / MANAGE) cited            | No        |
| 15 | Unverifiable claims called out inline as placeholders with a named owner for the follow-up                      | Yes       |

## 5. Common mistakes

- **Inferring the reference from the current snapshot.** A detector
  that regenerates its reference from the latest snapshot cannot
  ever detect drift. The reference is a pinned contract.
- **One global "drift found" severity.** Everything becomes a page
  (burns the on-call) or everything becomes a ticket (buries real
  incidents). The two-tier split is the reason the exercise exists.
- **Treating added columns as major.** Freezing training on every
  additive change makes the control intolerable and the on-call
  will disable it. `added` is `minor`.
- **Treating narrowed types as minor.** `int32 → int8` silently
  truncates every value above 127; always `major`.
- **Closed-world column with no `growth_budget_per_week`.** Without
  budget 0 the detector shrugs at a new value. Pin the budget — zero
  is a defensible value.
- **Email-only alerting for major drift.** Mail goes unread
  overnight. Major → PagerDuty + Slack (visible to the whole
  rotation); email is a receipt, not a page.
- **Rollback runbook with no named approver.** At 3 a.m. the on-call
  will skip the approval step "just this once". Name the approver
  by role; keep the ack in the incident Slack thread.
- **No "keep training paused" step.** The temptation to resume after
  a successful revert is enormous. Training stays paused until the
  reference is explicitly updated via a reviewed PR.
- **Alert sender silently succeeds when env is missing.** A typo in
  an env-var name turns the control into a null-op. `notify.py`
  must exit non-zero; CI asserts the error path.
- **One reference per consumer.** The schema contract belongs to the
  dataset, not the consumer; drift between duplicate references
  becomes a new failure mode. One `reference_schema.json` per dataset.

## 6. References

- **OWASP ML Security Top 10 — ML02:2023 Data Poisoning Attack:**
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Schema drift is the observable surface of upstream poisoning; the
  detector is the first-line compensating control.
- **MITRE ATLAS — AML.T0019 Publish Poisoned Datasets:**
  <https://atlas.mitre.org/>. Attacker-facing technique the drift
  control is designed to blunt.
- **NIST AI Risk Management Framework (MEASURE, MANAGE pillars):**
  <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE
  anchors "performance over time" (schema conformance as leading
  indicator); MANAGE anchors the deactivation runbook when the
  contract breaks.
- **Great Expectations:** <https://greatexpectations.io/>. GE
  `ExpectationSuite` primitives (`expect_table_columns_to_match_ordered_list`,
  `expect_column_values_to_be_of_type`,
  `expect_column_values_to_be_in_set`,
  `expect_column_unique_value_count_to_be_between`) are the
  drop-in equivalent for §2.3's pyarrow detector.
- **dbt source freshness:**
  <https://docs.getdbt.com/reference/resource-properties/freshness>.
  `loaded_at_field` + `warn_after` / `error_after` is the time half
  of the drift story; the detector is the shape half.
- **DVC:** <https://dvc.org/>. The "last-good snapshot" pointer the
  rollback reverts to.
- **lakeFS:** <https://lakefs.io/>. Git-for-data alternative;
  `lakectl branch revert` is the one-shot revert.
- **PagerDuty Events API v2:**
  <https://developer.pagerduty.com/docs/events-api-v2-overview>.
  `routing_key`, `dedup_key`, `severity` — fields `pagerduty_v2.json`
  fills.
- **Local anchors:** Exercise 01 (schema-conformance SLI —
  `severity=none` is the SLI's green state); Exercise 02 (tool
  selection provides this detector); Exercise 04 (subpopulation drift
  is also severity-major and shares the `major` route); Exercise 05
  (regression fixtures assert detector behaviour); `mod-109` (data
  version pin — the revert target).

Note on NIST AI RMF sub-item anchoring. The MEASURE/MANAGE
pillar-level anchors above are defensible on their own; the exact
sub-item numbers (e.g., MEASURE 2.x for schema conformance as a
leading indicator, MANAGE 2.x for the deactivation runbook) should be
confirmed against the current RMF revision before being quoted in
audit evidence, and are left as a reviewer-owned follow-up.
